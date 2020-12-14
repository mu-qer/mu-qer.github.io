---
layout: post
title: lifecycle源码分析
date: 2020-10-20 23:30:09
categories: Ceph
description: lifecycle
tags: Ceph
---


# lc 总流程
> lc的处理框架和gc的几乎一样, 总体流程如下：

lc handle 图


# lc 源码分析

## 总处理框架

```c++
int RGWLC::process()
{
  int max_secs = cct->_conf->rgw_lc_lock_max_time;

  //随机定义一个开始的地方
  const int start = ceph::util::generate_random_number(0, max_objs - 1);

  for (int i = 0; i < max_objs; i++) {
    int index = (i + start) % max_objs; //得到一个lc index
    int ret = process(index, max_secs);
    if (ret < 0)
      return ret;
  }
  return 0;
}
```

## 单个 lc.index 的处理

```c++
int RGWLC::process(int index, int max_lock_secs)
{
  //生成"lc_process锁"
  rados::cls::lock::Lock l(lc_index_lock_name); //lc_index_lock_name="lc_process
  do {
    utime_t now = ceph_clock_now();

    //string = bucket_name:bucket_id ,int = LC_BUCKET_STATUS
    pair<string, int > entry;

    ......

	//锁住该 obj_names[index] {"lc.index"}
    int ret = l.lock_exclusive(&store->lc_pool_ctx, obj_names[index]);
    if (ret == -EBUSY || ret == -EEXIST) { /* already locked by another lc processor */
      ldpp_dout(this, 0) << "RGWLC::process() failed to acquire lock on "
          << obj_names[index] << ", sleep 5, try again" << dendl;
      sleep(5);
      continue;
    }
    if (ret < 0)
      return 0;

	//在lc所在的pool中取 lc.index 对应的head信息, 因为header信息中含有当前lc.index应处理的bucket对应的marker
    cls_rgw_lc_obj_head head;
    ret = cls_rgw_lc_get_head(store->lc_pool_ctx, obj_names[index], head);
    if (ret < 0) {
      ldpp_dout(this, 0) << "RGWLC::process() failed to get obj head "
          << obj_names[index] << ", ret=" << ret << dendl;
      goto exit;
    }

    //如果当前header的start_date不是今天
    if(!if_already_run_today(head.start_date)) { 

	  //将head.start_date更新到今天的日期
      head.start_date = now;
	  //将当前header的marker清空
      head.marker.clear();

	  //由于当前header的start_date不是今天,即当前lc.index今天不用被执行, 那么将该lc中的item的状态全部更新为 uninitial
      ret = bucket_lc_prepare(index);
      if (ret < 0) {
      ldpp_dout(this, 0) << "RGWLC::process() failed to update lc object "
          << obj_names[index] << ", ret=" << ret << dendl;
      goto exit; //释放锁并退出
      }
    }

	//当前lc的start_date是今天,表示当前lc需要今天被处理
	//从当前lc的marker出取出一个entry来进行处理
    ret = cls_rgw_lc_get_next_entry(store->lc_pool_ctx, obj_names[index], head.marker, entry);
    if (ret < 0) {
      ldpp_dout(this, 0) << "RGWLC::process() failed to get obj entry "
          << obj_names[index] << dendl;
      goto exit;
    }

	//entry.first是bucket的sharding id
    if (entry.first.empty())
      goto exit;

	//将当前entry状态置为 processing
    entry.second = lc_processing;
    ret = cls_rgw_lc_set_entry(store->lc_pool_ctx, obj_names[index],  entry);
    if (ret < 0) {
      ldpp_dout(this, 0) << "RGWLC::process() failed to set obj entry " << obj_names[index]
          << " (" << entry.first << "," << entry.second << ")" << dendl;
      goto exit;
    }

	//更新header.marker,并写入到 lc_pool中的 lc.index 对象的header中
    head.marker = entry.first;
    ret = cls_rgw_lc_put_head(store->lc_pool_ctx, obj_names[index],  head);
    if (ret < 0) {
      ldpp_dout(this, 0) << "RGWLC::process() failed to put head " << obj_names[index] << dendl;
      goto exit;
    }
    l.unlock(&store->lc_pool_ctx, obj_names[index]);
    ret = bucket_lc_process(entry.first);
    bucket_lc_post(index, max_lock_secs, entry, ret);
  }while(1);

exit:
    l.unlock(&store->lc_pool_ctx, obj_names[index]);
    return 0;
}
```



### bucket_lc_prepare

```c++
int RGWLC::bucket_lc_prepare(int index)
{
  map<string, int > entries;

  string marker; //空

#define MAX_LC_LIST_ENTRIES 100
  do {
	//entries中每个对象<string, int>中string是bucket sharding id, 结构：bucket_name:bucket_id
    //从当前marker处取出100条lc item放到 entries中, 每个entry格式：pair<string(bucket sharding id), int(status: uninitial, complete,..)>
    int ret = cls_rgw_lc_list(store->lc_pool_ctx, obj_names[index], marker, MAX_LC_LIST_ENTRIES, entries);
    if (ret < 0)
      return ret;
    map<string, int>::iterator iter;
    for (iter = entries.begin(); iter != entries.end(); ++iter) {
      pair<string, int > entry(iter->first, lc_uninitial);
	  
	  //逐条设置为 lc_uninitial
      ret = cls_rgw_lc_set_entry(store->lc_pool_ctx, obj_names[index],  entry);
      if (ret < 0) {
        ldpp_dout(this, 0) << "RGWLC::bucket_lc_prepare() failed to set entry on "
            << obj_names[index] << dendl;
        return ret;
      }
    }

    if (!entries.empty()) { //如果 entries 非空
      marker = std::move(entries.rbegin()->first); //更新一下 marker
    }
  } while (!entries.empty());

  return 0;
}
```


### bucket内的核心处理

```c++
// 参数shard_id是bucket进行sharding时候的id
int RGWLC::bucket_lc_process(string& shard_id)
{
  RGWLifecycleConfiguration  config(cct);
  RGWBucketInfo bucket_info;
  map<string, bufferlist> bucket_attrs;
  string no_ns, list_versions;
  vector<rgw_bucket_dir_entry> objs;
  auto obj_ctx = store->svc.sysobj->init_obj_ctx();
  vector<std::string> result;
  boost::split(result, shard_id, boost::is_any_of(":"));
  string bucket_tenant = result[0];
  string bucket_name = result[1];
  string bucket_marker = result[2];

  ///获取bucket info, 以及bucket_attr; (bucket_info是引用， bucket_attrs是指针)
  int ret = store->get_bucket_info(obj_ctx, bucket_tenant, bucket_name, bucket_info, NULL, &bucket_attrs);
  if (ret < 0) {
    ldpp_dout(this, 0) << "LC:get_bucket_info for " << bucket_name << " failed" << dendl;
    return ret;
  }

  if (bucket_info.bucket.marker != bucket_marker) {
    ldpp_dout(this, 1) << "LC: deleting stale entry found for bucket=" << bucket_tenant
                       << ":" << bucket_name << " cur_marker=" << bucket_info.bucket.marker
                       << " orig_marker=" << bucket_marker << dendl;
    return -ENOENT;
  }

  RGWRados::Bucket target(store, bucket_info);

  //获取该bucket对应的名为lc的attr
  map<string, bufferlist>::iterator aiter = bucket_attrs.find(RGW_ATTR_LC);
  if (aiter == bucket_attrs.end())
    return 0;

  bufferlist::const_iterator iter{&aiter->second};
  try {
      config.decode(iter);
    } catch (const buffer::error& e) {
      ldpp_dout(this, 0) << __func__ <<  "() decode life cycle config failed" << dendl;
      return -1;
    }

  //获取配置文件中对应的待过滤的 prefix, 以及对应的 lc_opration
  multimap<string, lc_op>& prefix_map = config.get_prefix_map();

  ldpp_dout(this, 10) << __func__ <<  "() prefix_map size="
		      << prefix_map.size()
		      << dendl;

  rgw_obj_key pre_marker;
  rgw_obj_key next_marker;
  
  for(auto prefix_iter = prefix_map.begin(); prefix_iter != prefix_map.end(); ++prefix_iter) {
    auto& op = prefix_iter->second;
    if (!is_valid_op(op)) {
      continue;
    }
    ldpp_dout(this, 20) << __func__ << "(): prefix=" << prefix_iter->first << dendl;

	//更新marker
    if (prefix_iter != prefix_map.begin() && 
        (prefix_iter->first.compare(0, prev(prefix_iter)->first.length(), prev(prefix_iter)->first) == 0)) {
      next_marker = pre_marker;
    } else {
      pre_marker = next_marker;
    }

    LCObjsLister ol(store, bucket_info);
    ol.set_prefix(prefix_iter->first);

    ret = ol.init();

    if (ret < 0) {
      if (ret == (-ENOENT))
        return 0;
      ldpp_dout(this, 0) << "ERROR: store->list_objects():" <<dendl;
      return ret;
    }

    op_env oenv(op, store, this, bucket_info, ol);

    LCOpRule orule(oenv);
    //build lc handler对象
    orule.build();

    ceph::real_time mtime;
    rgw_bucket_dir_entry o;
    for (; ol.get_obj(&o); ol.next()) {
      ldpp_dout(this, 20) << __func__ << "(): key=" << o.key << dendl;

      //执行lc相关操作： expiration or transition
      int ret = orule.process(o); 
      if (ret < 0) {
        ldpp_dout(this, 20) << "ERROR: orule.process() returned ret="
			    << ret
			    << dendl;
      }

      if (going_down()) {
        return 0;
      }
    }
  }

  //针对分段上传的对象进行该操作
  ret = handle_multipart_expiration(&target, prefix_map);

  return ret;
}
```
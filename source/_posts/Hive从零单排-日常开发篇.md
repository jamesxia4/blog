---
title: Hive从零单排-日常开发篇[持续更新中]
date: 2016-03-01 11:19:10
tags: hive
---

Hive从零单排-日常开发篇<br/>
最后更新：2016-03-01<br/>

1. lateral view, explode与split的用于展开string中的list<br/>

		最近接了客服门户的需求，简而言之就是要统计每个来电原因在某个时段里的个数。
		在ods表里，主要是用一个biztype_id_list来记录，这个biztype_id_list记录了4层来电原因，以逗号分隔(比如: [1,2,3,4])。
		需求需要统计每一层的来电个数，因此我们需要用lateral view,explode和split来展开这个biztype_id_list

		具体写法如下:

		select  id as id
                ,biztype_id as biztype_id
                ,coalesce(concat(substr(a.gmt_create,0,4),substr(a.gmt_create,6,2),substr(a.gmt_create,9,2),substr(a.gmt_create,12,2))) as time
        from(
                select  *
                from    ods_dj_service_record
                where   pt = '${data_desc}'                          #把对pt的限制拉到前面优化性能
            )           a
            lateral view explode(
                        split(a.biztype_id_list,',')
            )           b
            as          biztype_id
        where   biztype_id <> ''                                 #对pt的限制放到上面，具体判断放下面
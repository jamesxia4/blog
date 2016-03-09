---
title: Hive从零单排-经验教训篇[持续更新中]
date: 2016-03-09 15:06:00
tags:
---

Hive从零单排-经验教训篇<br/>
最后更新：2016-03-09<br/>

1. 取数来源不准确与grouping sets共同导致的结果错误<br/>


**经验教训:<br/>
1.只要用了grouping sets一定要注意保证group by里面的维度不含null值<br/>
2.如果从子表取数，和父表join的时候一定要用left outer join,并且处理join为null的结果<br/>**

从上周三起就一直在对代金券券系统新老用户日报表进行修复，故障现象是 总订单数\>新用户订单数+老用户订单数。<br/>
新用户与老用户的判定依据为：在T-2交易日前是否完成过一个完整的代驾订单，取数的来源是订单结果表，只要有成单，我们认为就是老用户，剩下的就是新用户<br/>
新老用户日报表的取数来源是订单发单表，根据业务逻辑，乘客发单后经过司机抢单后的结果存入订单结果表。<br/>
因此，实际上的新用户订单数加上老用户订单数是会小于总订单数的，这是因为所有用户中除了用户画像中的老用户和新用户之外，还有一部分有过发单但是没有被抢单或者主动取消的用户的后续订单被计算逻辑遗漏了<br/>
解决方法：根据业务逻辑，把那些既不是1（老用户）又不是0（新用户）的用户(null)全部coalesce成0，结果应该就正确了<br/>
真的正确了么？永远不要以为你已经赢了斜眼笑<br/>
经过调整，测试结果似乎已经正常，将计算部分修改后加入线上代码，新用户订单数+老用户订单数反而更加小了！！！<br/>
这是为什么呢？？当时我百思不得其解，只能将计算部分与取数建临时表部分进行合并，尝试重现这个Bug。<br/>
最后发现，单独取数建临时表，订单总量正确，加入计算部分，结果偏小。 <br/>
突然想到前段时间师傅提醒我**凡是用了grouping sets的报表一定要检查grouping sets里面的维度是否是null** <br/>
这张报表的计算部分正巧用到了grouping sets，维度量有二: city\_id 和 order\_source <br/>
具体计算部分如下

		
		select '${data_desc}'
       	,nvl(k.city_id,'-1000000') as city_id
       	,nvl(k.source,'-1000000') as source
       	,grouping__id
       	,'新用户成单数'
       	,'new_suc_ord_cnt'
       	,count(case when k.user_type = '0' then k.ord_id else null end) as new_suc_ord_cnt
        group by  k.city_id,k.source
        grouping  sets((),(k.city_id),(k.source),(k.city_id,k.source))

所以出问题的地方可能就是city\_id与order\_source<br/>
那我们来看看取数建临时表的代码，看看到底可能错在哪里：<br/>

		select           a.ord_id as ord_id
                         ,d.source as source
                         ,c.city_id as city_id
                         ,coalesce(b.user_type,'0')  as user_type
                         ,b.pag_id         
                         ,e.ord_id as finish_ord_id   
                         ,e.voucher_amt          
        from
                         (
                         select         *
                         from           dwd_ord_result
                         where          end_charge_time is not null
                         and            pt = '2016-02-29' 
                         )              a
                         left outer join
                         (
                         select         *
                         from           dm_pag_portrait 
                         where          pt = '2016-02-28'
                         )              b
                         on             ( a.pag_id = b.pag_id )
                         left outer join 
                         (
                         select         *
                         from           dwd_pag_info
                         where          pt = '2016-02-29'  
                         )              c
                         on             ( b.pag_id = c.pag_id )
                         join
                         (
                         select         ord_id as ord_id
                                        ,(case when send_type = 2 then '司机报单'
                                            when send_type = 3 then '400叫单'
                                            when send_type = 3 and channel_id = 101 then '400叫单-安吉星'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 0 then 'WebApp默认'
                                            when send_type = 1 and channel_id = 3 and sub_channel is null then 'WebApp默认'
                                            when send_type = 1 and channel_id = 1 then '快的乘客发单'
                                            when send_type = 1 and channel_id = 2 then '滴滴乘客发单'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 1 then '微信公众号_钱包'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 2 then '微信扫码'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 3 then '微信扫码推广版'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 4 then 'Web通用版'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 5 then '滴滴专车司机端'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 6 then '支付宝钱包'
                                            when send_type = 1 and channel_id = 3 and sub_channel = 7 then '代驾官网'
                                        else '其他' end) as source
                         from           dwd_ord_info
                         where          pt = '2016-02-29'
                         and risk_type in (1,5)
                         )              d
                         on             ( a.ord_id = d.ord_id )
                         left outer join
                         (
                         select         *
                         from           dwd_tra_ord_pay_finish
                         where          pt = '2016-02-29'
                         )              e
                         on             ( d.ord_id = e.ord_id )

city\_id从dwd\_pag\_info（乘客信息总表）中取数，那么city\_id可能为null么？经过检查，的确存在city\_id为null的情况<br/>
再查grouping\_id, 确实在city\_id和order\_source均为null时中存在两项计算结果，grouping\__id分别为0和1,说明有一项结果中有**city\_id为null的数据干扰了grouping_sets**，这就是故障原因<br/>

对症下药<br/>
1.尝试在取数时过滤掉null的乘客id，无效果<br/>
2.最后只能coalesce city\_id，结果正确，说明的确是**city\_id有null值导致grouping sets计算结果错误**<br/>

最后贴改后代码
		
		from( 
            select           a.ord_id as ord_id
                             ,coalesce(d.source,'其他') as source
                             ,coalesce(c.city_id,'-10000') as city_id
                             ,coalesce(b.user_type,'0')  as user_type
                             ,b.pag_id         
                             ,e.ord_id as finish_ord_id   
                             ,e.voucher_amt          
            from
                             (
                             select         *
                             from           dwd_ord_result
                             where          end_charge_time is not null
                             and            pt = '2016-02-29' 
                             )              a
                             left outer join
                             (
                             select         *
                             from           dm_pag_portrait 
                             where          pt = '2016-02-28'
                             )              b
                             on             ( a.pag_id = b.pag_id )
                             left outer join 
                             (
                             select         *
                             from           dwd_pag_info
                             where          pt = '2016-02-29'  
                             and            city_id is not null
                             )              c
                             on             ( b.pag_id = c.pag_id )
                             join
                             (
                             select         ord_id as ord_id
                                            ,(case when send_type = 2 then '司机报单'
                                                when send_type = 3 then '400叫单'
                                                when send_type = 3 and channel_id = 101 then '400叫单-安吉星'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 0 then 'WebApp默认'
                                                when send_type = 1 and channel_id = 3 and sub_channel is null then 'WebApp默认'
                                                when send_type = 1 and channel_id = 1 then '快的乘客发单'
                                                when send_type = 1 and channel_id = 2 then '滴滴乘客发单'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 1 then '微信公众号_钱包'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 2 then '微信扫码'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 3 then '微信扫码推广版'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 4 then 'Web通用版'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 5 then '滴滴专车司机端'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 6 then '支付宝钱包'
                                                when send_type = 1 and channel_id = 3 and sub_channel = 7 then '代驾官网'
                                            else '其他' end) as source
                             from           dwd_ord_info
                             where          pt = '2016-02-29'
                             and risk_type in (1,5)
                             )              d
                             on             ( a.ord_id = d.ord_id )
                             left outer join
                             (
                             select         *
                             from           dwd_tra_ord_pay_finish
                             where          pt = '2016-02-29'
                             )              e
                             on             ( d.ord_id = e.ord_id )
            )   k
        select '${data_desc}'
       ,nvl(k.city_id,'-1000000') as city_id
       ,nvl(k.source,'-1000000') as source
       ,grouping__id
       ,'新用户成单数'
       ,'new_suc_ord_cnt'
       ,count(case when k.user_type = '0' then k.ord_id else null end) as new_suc_ord_cnt
        group by  k.city_id,k.source
        grouping  sets((),(k.city_id),(k.source),(k.city_id,k.source))



**经验教训:<br/>
1.只要用了grouping sets一定要注意保证group by里面的维度不含null值<br/>
2.如果从子表取数，和父表join的时候一定要用left outer join,并且处理join为null的结果<br/>**
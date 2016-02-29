---
title: Hive从零单排-故障修复篇[持续更新中]
date: 2016-02-25 11:19:10
tags:
---

Hive从零单排-故障修复篇<br/>
最后更新：2016-02-29<br/>

1. join的时候注意主表与副表key之间的对应关系，可能不是一一对应，比如一对多，这种情况可能造成结果集大小剧增<br/>
   例子:订单表中的用户id与每日登陆客户端表中的用户id 做join,登陆表中当日分区下同一个pag\_id会有多条记录，在取数时要用row_number（）来控制到底取哪条登陆记录

2. Container Killed on request, Err code 143
   爆内存<br/>
   解决方案1:拆分竖表代码<br/>
   解决方案2:改参数，待定<br/>
   
3. java.io.IOException: java.lang.reflect.InvocationTargetException
   在读取日志源时上游依赖还在写，导致冲突
   暂时没有解决方案
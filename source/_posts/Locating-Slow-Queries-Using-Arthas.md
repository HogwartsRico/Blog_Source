---
title: 使用Arthas定位慢查询
date: 2019-09-24 11:09:05
tags: Java
---



# 优化待评价的订单数量查询逻辑



最开始是xxxService.getOrderCount这个接口有慢日志日均80, 但是这个接口的调用量远超这个值, 判断是某个场景下触发了慢请求.

用Arthas监控了慢请求的入参, 发现了只有queryScene=WAIT_COMMNET(待评价)时，才会出现慢请求

![](1.jpg)  



查看了这个待评价场景的查询逻辑是有点问题的



![](2.jpg) 



优化点比较简单，主要还是发现问题和定位问题的过程。



# 接口慢查询定位分析

![](3.png)

可以看到该接口耗时2.2秒，太长了。 

该接口调用了4个方法，逐个对方法进行耗时统计

通过命令来对第一个方法进行统计

```shell
watch com.xxx.xxx.manager.xxx.impl.xxxManagerImpl  processAndGetTreamentScheme  '#cost>200' -x 5 
```

![](4.png)

耗时0.278ms，很明显不是这个方法导致的慢查询

通过这个方式一个个查看，最终发现了耗时长的方法，发现是一个全模糊查询

![](5.png)



```xml
  <select id="selectByLikeNames" resultMap="BaseResultMap"  >
    select
    <include refid="Base_Column_List" />
    from drug
    where name regexp #{name}
  </select>
```

这里name是多个，之前采用正则来实现,用程序拼装成字符串例如"张三|李四|王五",然后用mysql全模糊查。

这样不合理，较合理的方法是使用ElasticSearch或使用缓存来实现。但该项目没有这2个中间件，最后改成右模糊(会用到索引)用union来实现

```xml
<select id="selectByLikeNamesUnion" resultMap="BaseResultMap" >
    <foreach collection="names" item="name" separator="union" >
      SELECT
      <include refid="Base_Column_List" />
      FROM drug
      WHERE NAME LIKE concat(#{name},'%')  and org_code = #{orgCode}
    </foreach>
  </select>
```

经过优化后这个接口耗时在500ms左右。 



后续写一下Arthas的各个用法。 
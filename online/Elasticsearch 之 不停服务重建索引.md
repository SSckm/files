## 问题描述

当我们在使用Es做为我们应用的全文检索服务器的时候，文档结构避免不了会发生变化，比如增加字段，而且增加的字段需要进行分词时，再比如原来文档结构中的字段类型发生改变了，假设原来的字段类型是String, 而修改后的字段类型是Object，Es是不支持这种直接修改字段类型的。我们能做的可能只有进行重新索引，当然关系型数据库也不支持这种类型字段改变的情况。

Elasticsearch 的各种语言的API在此不做介绍了（包括rest api），很简单.

也许现在有人提问当进行重新的索引的时候程序难道不用改变么？

当然不用。相信用过Spring的同学都知道，spring配置中有一个叫“别名的东西“

比如：

```
<authentication-manager alias="authenticationManager">
<authentication-provider user-service-ref="sysUserDetailsService"/>
</authentication-manager>
```

幸运的是Es也有这个“别名“机制
索引当我们在第一次建立一个索引的时候最好还是创建一个别名。
避免以后改变的时候要不改变程序，要不删除原来的数据。很麻烦

好了，废话不多说了直接上代码：

##创建别名

```
CreateIndexResponse response = client.admin().indices().prepareCreate(indexName).setAliases(aliase).setSettings(builder).execute().actionGet();
```

其中setAliases就是设置创建的这个索引的别名，访问哪一个都一样，包括CRUD操作。

比如：

索引名称是：china_address1
索引别名是：address
访问那一个都是一样的。

##重新索引数据

```
SearchResponse scrollResp = client
        .prepareSearch(indexName)
        .setSearchType(SearchType.SCAN)
        .setScroll(new TimeValue(60000))
        .setSize(100)
        .execute()
        .actionGet();
    while (true) {
      SearchHit[] arrayScoll = scrollResp.getHits().getHits();
      //处理查询出来的数据放入到新的索引里面
      createIndexBuilk(arrayScoll, newIndexName, arrayMapping);
      scrollResp = client.prepareSearchScroll(scrollResp.getScrollId()).setScroll(new TimeValue(60000)).execute().actionGet();
      if (scrollResp.getHits().getHits().length == 0) {
        break;
      }
    }
```

然后再删除旧索引的别名address，再在新索引中加入别名address.这样应用就可以不用重新启动也不需要改就可以直接访问服务了。但是还有会有一个瞬间的增量问题。

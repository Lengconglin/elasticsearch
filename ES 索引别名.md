# ES 索引别名

### 使用场景：
当需要对旧索引的模板以及mapping、setting等配置做出改变，在不停服务的情况下将索引别名指向新索引，可以做到对用户完全无感知。  

---  

## 别名介绍：
别名aliases在ES中就像Linux中的软连接一样，可以将一个或者多个索引映射到该索引别名，提供了非常灵活的特性，使用它我们可以做到：
（1）在一个运行的ES集群中可以无缝的切换一个索引到另一个索引上  
（2）将多个索引进行分组，比如按天创建的索引，可以通过别名构造出最近一周或者最近一个月的索引  
（3）查询一个索引里面的部分数据构成一个类似数据库的视图

## 操作：
ES中操作索引别名的api有两个：  
	_alias 执行单个别名操作  
	_aliases 原子的执行多个别名操作  

--- 

## 实例：
假设有两个索引分别为forest_index_v1和forest_index_v2，想通过索引别名来实现无缝切换，设定对外的别名为：forest_index    

##### 1、首先在创建索引的时候就给index_v1添加别名  
PUT /forest_index_v1   //构建索引  
PUT /forest_index_v1/_alias/forest_index   //给索引添加别名

##### 2、创建完成后查看索引与别名之间的关系  
GET /*/_alias/forest_index         //查某个别名映射的所有index  
GET /forest_index_v1/_alias/*   //查询某个index拥有的别名

##### 3、构建新的index  
PUT /forest_index_v2   //构建索引  

##### 4、切换旧版本索引到新版本（原子操作）  

	POST /_aliases
	{
	    "actions": [
	        { "remove": { "index": "forest_index_v1", "alias": "forest_index" }},
	        { "add":    { "index": "forest_index_v2", "alias": "forest_index" }}
	    ]
	}  

以上操作必须是顺序执行，先解除旧版本的index别名，然后给版本index添加别名，这样索引就透明切换，用户全程无感知  

## Java api操作：
#### 1、添加别名  
```
client.admin().indices().prepareAliases().addAlias("forest_index_v1","forest_index");  
```

#### 2、移除别名  
```
client.admin().indices().prepareAliases().removeAlias("forest_index_v1","forest_index");  
```

#### 3、删除一个别名之后再添加一个  
```
client.admin().indices().prepareAliases().removeAlias("forest_index_v1","forest_index") .addAlias("forest_index_v2","forest_index").execute().actionGet();  
```

#### 4、在进行搜索，聚合等操作时使用别名  
```
 SearchRequestBuilder search=client.prepareSearch("forest_index");  
```

### 注意：使用别名之后不需要再填写type类型的值，如果填写了ES会抛异常，ES会认为这是一个新的index，所以在使用别名的时候只需要填写index即可，ES服务知道type
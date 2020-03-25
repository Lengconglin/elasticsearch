### 1、 ES 查询某字段为null或为空字符串  
	
	BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
	boolQueryBuilder.must(
	    QueryBuilders.boolQuery()
	        .should(QueryBuilders.termQuery("要查的字段名",""))
	        .should(QueryBuilders.boolQuery().mustNot(QueryBuilders.existsQuery("要查的字段名"))
	);
# ES 指标聚合  
对一个数据集合中的数据求sum、max、min、count、avg等指标的聚合，被称为指标聚合，Metric Aggregation。  
### 1、单值分析：只输出一个分析结果
max、min、sum、avg、count、cardinality（去重计数，类似distinct count）  

### 2、多值分析：输出多个分析结果  
stats、extend stats、percentile、percentile rank、top hits

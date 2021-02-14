这两天正好做个需求，需要用到聚合查询。前几篇文章只是简单的提到过，并没有真正的运用到实际产出中，本篇结合实际代码，专项学习ES的聚合查询。

## 1、业务背景

有一张地址索引表：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221145527507.png)
hisAddress与formatAddress是一对多的关系。
当一条地址进来查找hisAddress，然后对formatAddress做聚合，再根据count筛选聚合中的数据。
类似以下SQL：

```sql
select hisAddress,formatAddress,count(*) from addressIndex 
where hisAddress = "上海市静安区静安路100号静安寺" 
group by formatAddress having count(*) > 10
123
```

当然逻辑比这个稍稍复杂，需要使用嵌套聚合筛选数据。
流程如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181221152054429.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlbndlbjUxMw==,size_16,color_FFFFFF,t_70)

## 2、elasticsearch的query

```json
{
"size":0,
"query":{
	"bool":{
		"filter":{
			"term":{
				"hisAddress":"上海市上海市静安区静安路100号静安寺"
			}
		}
	}
},
"aggs":{
	"format_address":{		
			"terms":{
				"field":"formatAddress" 
			},
			
			"aggs":{
				"sign_org":{
					"terms":{
						"field":"signOrgCode"
					}
				},
				
				"having":{
					"bucket_selector":{
						"buckets_path":{
							"formatCount":"_count"		
						},
						"script":{						
		            		"inline": " formatCount>10"
						}							
					}
				},
			
				"stats_sign_bulk":{
					"stats_bucket":{
						"buckets_path":"sign_org > _count"
					}
				}
			}
	}
}
}
1234567891011121314151617181920212223242526272829303132333435363738394041424344
```

返回结果：

```json
{
    "took": 19,
    "timed_out": false,
    "_shards": {
        "total": 6,
        "successful": 6,
        "failed": 0
    },
    "hits": {
        "total": 67,
        "max_score": 0,
        "hits": []
    },
    "aggregations": {
        "format_address": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
                {
                    "key": "上海市上海市静安区静安寺",
                    "doc_count": 21,
                    "sign_org": {
                        "doc_count_error_upper_bound": 0,
                        "sum_other_doc_count": 0,
                        "buckets": [
                            {
                                "key": "022795",
                                "doc_count": 21
                            }
                        ]
                    },
                    "stats_sign_bulk": {
                        "count": 1,
                        "min": 21,
                        "max": 21,
                        "avg": 21,
                        "sum": 21
                    }
                },
                {
                    "key": "上海市上海市静安区静安路100号",
                    "doc_count": 21,
                    "sign_org": {
                        "doc_count_error_upper_bound": 0,
                        "sum_other_doc_count": 0,
                        "buckets": [
                            {
                                "key": "022795",
                                "doc_count": 21
                            }
                        ]
                    },
                    "stats_sign_bulk": {
                        "count": 1,
                        "min": 21,
                        "max": 21,
                        "avg": 21,
                        "sum": 21
                    }
                },
                {
                    "key": "上海市上海市静安区肯德基",
                    "doc_count": 16,
                    "sign_org": {
                        "doc_count_error_upper_bound": 0,
                        "sum_other_doc_count": 0,
                        "buckets": [
                            {
                                "key": "01111",
                                "doc_count": 16
                            }
                        ]
                    },
                    "stats_sign_bulk": {
                        "count": 1,
                        "min": 16,
                        "max": 16,
                        "avg": 16,
                        "sum": 16
                    }
                }
            ]
        }
    }
}
12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576777879808182838485
```

## 3、elasticsearchTemplate的实现

```java
String hisAddress = "上海市上海市静安区静安路100号静安寺";
List<HistoryIndexDocument> prepareList = new ArrayList<HistoryIndexDocument>();
Map<String,String> bucketMap = new HashMap<String, String>();
bucketMap.put("formatCount", "_count");

// 根据全量地址和寄派类型查询数据（此处使用filter过滤，它能缓存数据且不参与计算分值，比query速度快）
QueryBuilder queryBuilder = QueryBuilders
		.boolQuery()
		.filter(QueryBuilders.termQuery("hisAddress", entity.getHisAddress()))
		.filter(QueryBuilders.termQuery("rangeType", entity.getRangeType()));
	// 结构化地址聚合桶
TermsBuilder format_address_aggs = AggregationBuilders.terms("format_address_aggs").field("formatAddress");
	// 签收网点聚合桶
TermsBuilder sign_org_aggs = AggregationBuilders.terms("sign_org_aggs").field("signOrgCode");
// 管道聚合，类似having count(*) > 10
BucketSelectorBuilder bucketSelectorBuilder = PipelineAggregatorBuilders
 			.having("having")
 			.setBucketsPathsMap(bucketMap)
 			.script(new Script("formatCount>10"));

// 嵌套聚合，类似在group by formatAddress的基础上再group by signOrgCode
format_address_aggs.subAggregation(sign_org_aggs);
// 嵌套聚合，筛选数量大于10的结构化地址
format_address_aggs.subAggregation(bucketSelectorBuilder);
// 嵌套聚合，筛选数量大于10的签收网点
sign_org_aggs.subAggregation(bucketSelectorBuilder);

SearchQuery searchQuery = new NativeSearchQueryBuilder()
		.withIndices("my_index").withTypes("my_type")
		.withQuery(queryBuilder)
		.withPageable(new PageRequest(0, 1, null))
		.addAggregation(format_address_aggs)
		.build();

// 执行语句获取聚合结果
Aggregations aggregations = elasticsearchTemplate.query(searchQuery, new ResultsExtractor<Aggregations>() {

	@Override
	public Aggregations extract(SearchResponse response) {
		return response.getAggregations();
	}
});

// 获取聚合结果
StringTerms teamAgg = (StringTerms) aggregations.asMap().get("format_address_aggs");
List<Bucket> bucketList = teamAgg.getBuckets();
for(Bucket bucket:bucketList) {
	// 结构化地址
	String formatAddress = bucket.getKeyAsString();
	System.out.println(formatAddress);
	
	Aggregations signAggs = bucket.getAggregations();
	StringTerms signTerms = (StringTerms) signAggs.asMap().get("sign_org_aggs");
	List<Bucket> signBucketList = signTerms.getBuckets();
	// 签收网点只能一个
	if(signBucketList==null || signBucketList.size() >1) {
		continue;
	}
	
	Bucket signBucket = signBucketList.get(0);
	// 签收频次需要5次以上
	if(signBucket.getDocCount() >= 5) {
		
		// 满足条件的网点放入prepareList
		HistoryIndexDocument entity = new HistoryIndexDocument();
		entity.setFormatAddress(formatAddress);
		entity.setSignOrgCode(signBucket.getKeyAsString());
		prepareList.add(entity);
	}
}

System.out.println(FastJsonUtil.toJsonString(prepareList));
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172
```

## 4、更多java API聚合的用法

```java
//（1）统计某个字段的数量
  ValueCountBuilder vcb=  AggregationBuilders.count("count_uid").field("uid");
//（2）去重统计某个字段的数量（有少量误差）
 CardinalityBuilder cb= AggregationBuilders.cardinality("distinct_count_uid").field("uid");
//（3）聚合过滤
FilterAggregationBuilder fab= AggregationBuilders.filter("uid_filter").filter(QueryBuilders.queryStringQuery("uid:001"));
//（4）按某个字段分组
TermsBuilder tb=  AggregationBuilders.terms("group_name").field("name");
//（5）求和
SumBuilder  sumBuilder=	AggregationBuilders.sum("sum_price").field("price");
//（6）求平均
AvgBuilder ab= AggregationBuilders.avg("avg_price").field("price");
//（7）求最大值
MaxBuilder mb= AggregationBuilders.max("max_price").field("price"); 
//（8）求最小值
MinBuilder min=	AggregationBuilders.min("min_price").field("price");
//（9）按日期间隔分组
DateHistogramBuilder dhb= AggregationBuilders.dateHistogram("dh").field("date");
//（10）获取聚合里面的结果
TopHitsBuilder thb=  AggregationBuilders.topHits("top_result");
//（11）嵌套的聚合
NestedBuilder nb= AggregationBuilders.nested("negsted_path").path("quests");
//（12）反转嵌套
AggregationBuilders.reverseNested("res_negsted").path("kps ");

12345678910111213141516171819202122232425
```

了解更多详情，请参考官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search-aggregations.html
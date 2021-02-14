https://blog.csdn.net/alex_xfboy/article/details/86100037

## 1. 指标聚合

[Metrics Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics.html)，指标聚合。

它是对文档进行一些权值计算（比如求所有文档某个字段求最大、最小、和、平均值），输出结果往往是文档的权值，相当于为文档添加了一些统计信息。

它基于特定字段（field）或脚本值（generated using scripts），计算聚合中文档的数值权值。数值权值聚合（注意分类只针对数值权值聚合，非数值的无此分类）输出单个权值的，也叫 single-value numeric metrics，其它生成多个权值（比如：stats）的被叫做 multi-value numeric metrics。

### 1.1 max min sum avg

[Max Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-max-aggregation.html)，求最大值。基于文档的某个值（可以是特定的数值型字段，也可以通过脚本计算而来），计算该值在聚合文档中的均值。

[Min Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-min-aggregation.html)，求最小值。同上

[Sum Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-sum-aggregation.html)，求和。同上

[Avg Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-avg-aggregation.html)，求平均数。同上

> POST /sales/_search?size=0
> {
>   "aggs" : {
>     "max_price" : { "max" : { "field" : "price" } }
>   }
> }
> {//返回
>  "took": 2080,
>  "timed_out": false,
>  "_shards": {
>   "total": 5,
>   "successful": 5,
>   "skipped": 0,
>   "failed": 0
>  },
>  "hits": {
>   "total": 1000,
>   "max_score": 0,
>   "hits": []
>  },
>  "aggregations": {
>     "max_price": {
>       "value": 200.0
>     }
>   }
> }
> POST /sales/_search?size=0
> {
>   "aggs" : {
>     "min_price" : { "min" : { "field" : "price" } }
>   }
> }
> POST /sales/_search?size=0
> {
>   "query" : {
>     "constant_score" : {
>       "filter" : {
>         "match" : { "type" : "hat" }
>       }
>     }
>   },
>   "aggs" : {
>     "hat_prices" : { "sum" : { "field" : "price" } }
>   }
> }
> POST /exams/_search?size=0
> {
>   "aggs" : {
>     "avg_grade" : { "avg" : { "field" : "grade" } }
>   }
> }

### 1.2 值统计

[Value Count Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-valuecount-aggregation.html)，值计数聚合。计算聚合文档中某个值（可以是特定的数值型字段，也可以通过脚本计算而来）的个数。该聚合一般域其它 single-value **聚合联合使用**，比如在计算一个字段的平均值的时候，可能还会关注这个平均值是由多少个值计算而来。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_count": {
>    "value_count": {
>     "field": "age"
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_count": {
>    "value": 1000
>   }
>  }
> }

### 1.3 distinct 聚合

[Cardinality Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-cardinality-aggregation.html)，基数聚合。它属于multi-value，基于文档的某个值（可以是特定的字段，也可以通过脚本计算而来），计算文档非重复的个数（去重计数），相当于sql中的distinct。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_count": {
>    "cardinality": {
>     "field": "age"
>    }
>   },
>   "state_count": {
>    "cardinality": {
>     "field": "state.keyword"
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "state_count": {
>    "value": 51
>   },
>   "age_count": {
>    "value": 21
>   }
>  }
> }

### 1.4 统计聚合

[Stats Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-stats-aggregation.html)，统计聚合。它属于multi-value，基于文档的某个值（可以是特定的数值型字段，也可以通过脚本计算而来），计算出一些统计信息（min、max、sum、count、avg5个值）。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_stats": {
>    "stats": {
>     "field": "age"
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_stats": {
>    "count": 1000,
>    "min": 20,
>    "max": 40,
>    "avg": 30.171,
>    "sum": 30171
>   }
>  }
> }

### 1.5 拓展的统计聚合

[Extended Stats Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-extendedstats-aggregation.html)，扩展统计聚合。它属于multi-value，比stats多4个统计结果： 平方和、方差、标准差、平均值加/减两个标准差的区间

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_stats": {
>    "extended_stats": {
>     "field": "age"
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_stats": {
>    "count": 1000,
>    "min": 20,
>    "max": 40,
>    "avg": 30.171,
>    "sum": 30171,
>    "sum_of_squares": 946393,
>    "variance": 36.10375899999996,
>    "std_deviation": 6.008640362012022,
>    "std_deviation_bounds": {
>     "upper": 42.18828072402404,
>     "lower": 18.153719275975956
>    }
>   }
>  }
> }

### 1.6 百分比统计

[Percentiles Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-percentile-aggregation.html)，百分比聚合。它属于multi-value，对指定字段（脚本）的值按从小到大累计每个值对应的文档数的占比（占所有命中文档数的百分比），返回指定占比比例对应的值。默认返回[ 1, 5, 25, 50, 75, 95, 99 ]分位上的值。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_percents": {
>    "percentiles": {
>     "field": "age"
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_percents": {
>    "values": {
>     "1.0": 20,
>     "5.0": 21,
>     "25.0": 25,
>     "50.0": 31,  //占比为50%的文档的age值 <= 31，或反过来：age<=31的文档数占总命中文档数的50%
>     "75.0": 35.00000000000001,
>     "95.0": 39,
>     "99.0": 40
>    }
>   }
>  }
> }
>
> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_percents": {
>    "percentiles": {
>     "field": "age",
>     "percents" : [95, 99, 99.9]    //指定分位值
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_percents": {
>    "values": {
>     "95.0": 39,
>     "99.0": 40,
>     "99.9": 40
>    }
>   }
>  }
> }

### 1.7 百分比排名聚合

[Percentile Ranks Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-percentile-rank-aggregation.html)，统计年龄小于25和年龄小于30的文档的占比，这里需求可以使用。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "gge_perc_rank": {
>    "percentile_ranks": {
>     "field": "age",
>     "values": [
>      25,
>      30
>     ]
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "gge_perc_rank": {
>    "values": {  //年龄小于25的文档占比为26.1%，年龄小于30的文档占比为49.2%
>     "25.0": 26.1, 
>     "30.0": 49.2
>    }
>   }
>  }
> }

### 1.8 Top Hits

[Top Hits Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-top-hits-aggregation.html)，最高匹配权值聚合。获取到每组前n条数据，相当于sql 中Top（group by 后取出前n条）。它跟踪聚合中相关性最高的文档，该聚合一般用做 sub-aggregation，以此来聚合每个桶中的最高匹配的文档，较为常用的统计。

> GET index/type/_search?search_type=count
> {
>  "query": {
>   "match_all": {}
>  },
>  "aggs": {
>   "all_interests": {
>    "terms": {
>     "field": "zxw_id",
>     "size": 100
>    },
>    "aggs": {
>     "top_tag_hits": {
>      "top_hits": {
>       "size": 1  //返回的最大文档个数（default 3）
>      }
>     }
>    }
>   }
>  }
> }

### 1.9 Geo Bounds Aggregation

[Geo Bounds Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-geobounds-aggregation.html)，地理边界聚合。基于文档的某个字段（geo-point类型字段），计算出该字段所有地理坐标点的边界（左上角/右下角坐标点）。

> GET index/type/_search?search_type=count
> {
>  "query": {
>   "match_all": {}
>  },
>  "aggs": {
>   "viewport": {
>    "geo_bounds": {
>     "field": "location",
>     "wrap_longitude": true //是否允许地理边界与国际日界线存在重叠
>    }
>   }
>  }
> }
> {//返回，
>   ...
>   "aggregations": {
>     "viewport": {
>       "bounds": {
>         "top_left": { //这个矩形区域左上角坐标
>           "lat": 80.45,
>           "lon": -160.22
>         },
>         "bottom_right": {//这个矩形区域右下角坐标
>           "lat": 40.65,
>           "lon": 42.57
>         }
>       }
>     }
>   }
> }

### 1.10 Geo Centroid Aggregation

 [Geo Centroid Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-metrics-geocentroid-aggregation.html)，地理重心聚合。基于文档的某个字段（geo-point类型字段），计算所有坐标的加权重心。

> GET index/type/_search?search_type=count
> {
>   "query" : {
>     "match" : { "crime" : "burglary" }
>   },
>   "aggs" : {
>     "centroid" : {
>       "geo_centroid" : {
>         "field" : "location" 
>       }
>     }
>   }
> }
> {//输出
>   ...
>   "aggregations": {
>     "centroid": {
>       "location": {    //重心经纬度
>         "lat": 80.45,
>         "lon": -160.22
>       }
>     }
>   }
> }

## 2. 桶聚合

[Bucket Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket.html)，桶聚合。

它执行的是对文档分组的操作（与sql中的group by类似），把满足相关特性的文档分到一个桶里，即桶分，输出结果往往是一个个包含多个文档的桶（一个桶就是一个group）。

它有一个关键字（field、script），以及一些桶分（分组）的判断条件。执行聚合操作时候，文档会判断每一个分组条件，如果满足某个，该文档就会被分为该组（fall in）。

它不进行权值的计算，他们对文档根据聚合请求中提供的判断条件（比如：{"from":0,  "to":100}）来进行分组（桶分）。桶聚合还会额外返回每一个桶内文档的个数。

它可以包含子聚合——sub-aggregations（权值聚合不能包含子聚合，可以作为子聚合），子聚合操作将会应用到由父聚合产生的每一个桶上。

它根据聚合条件，可以只定义输出一个桶；也可以输出多个（multi-bucket）；还可以在根据聚合条件动态确定桶个数（比如：terms aggregation）。

### 2.1 Terms Aggregation

[Terms Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-terms-aggregation.html)，词聚合。基于某个field，该 field 内的每一个【唯一词元】为一个桶，并计算每个桶内文档个数。默认返回顺序是按照文档个数多少排序。它属于multi-bucket。当不返回所有 buckets 的情况（它size控制），文档个数可能不准确。

> POST /bank/_search?size=0
> {
>   "aggs" : {
>     "age_terms" : {
>       "terms" : { 
>        "field" : "age",
>        "size" : 10,               //size用来定义需要返回多个 buckets（防止太多），默认会全部返回。
>        "order" : { "_count" : "asc" }, //根据文档计数排序，根据分组值排序（{ "_key" : "asc" }）
>        "min_doc_count": 10,      //只返回文档个数不小于该值的 buckets
>        "include" : ".*sport.*",      //包含过滤
>        "exclude" : "water_.*",     //排除过滤
>        "missing": "N/A" 
>       }
>     }
>   }
> }
>
> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_terms": {
>    "terms": {
>     "field": "age",
>     "size": 5,
>     "shard_size": 20, //指定每个分片返回多少个分组，默认值（索引只有一个分片：= size，多分片：= size * 1.5 + 10）
>     "show_term_doc_count_error": true    //每个分组上显示偏差值
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_terms": {
>    "doc_count_error_upper_bound": 0, //文档计数的最大偏差值
>    "sum_other_doc_count": 463,      //未返回的其他项的文档数
>    "buckets": [        //默认情况下返回按文档计数从高到低的前10个分组
>     {
>      "key": 31,       //年龄为31的文档有61个
>      "doc_count": 61
>     },
>     {
>      "key": 39,      //年龄为39的文档有60个
>      "doc_count": 60
>     },
>     {
>      "key": 34,
>      "doc_count": 49
>     }
>    ]
>   }
>  }
> }

### 2.2 filter Aggregation

[Filter Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-filter-aggregation.html)，过滤聚合。基于一个条件，来对当前的文档进行过滤的聚合。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_terms": {
>    "filter": {"match":{"gender":"F"}},
>    "aggs": {
>     "avg_age": {
>      "avg": {
>       "field": "age"
>      }
>     }
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_terms": {
>    "doc_count": 493,
>    "avg_age": {
>     "value": 30.3184584178499
>    }
>   }
>  }
> }

### 2.3 Filters Aggregation

[Filters Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-filters-aggregation.html)，多过滤聚合。基于多个过滤条件，来对当前文档进行【过滤】的聚合，每个过滤都包含所有满足它的文档（多个bucket中可能重复），先过滤再聚合。它属于multi-bucket。

> GET logs/_search
> {
>  "size": 0,
>  "aggs": {
>   "messages": {
>    "filters": { // 配置过滤条件，支持 HASH 或 数组格式
>     "filters": {
>      "errors": {
>       "match": {
>        "body": "error"
>       }
>      },
>      "warnings": {
>       "match": {
>        "body": "warning"
>       }
>      }
>     }
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "messages": {
>    "buckets": {
>     "errors": {
>      "doc_count": 1
>     },
>     "warnings": {
>      "doc_count": 2
>     }
>    }
>   }
>  }
> }

### 2.4 范围聚合

[Range Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-range-aggregation.html)，范围分组聚合。基于某个值（可以是 field 或 script），以【字段范围】来桶分聚合。范围聚合包括 from 值，不包括 to 值（区间前闭后开）。它属于multi-bucket。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "age_range": {
>    "range": {
>     "field": "age",
>     "ranges": [ //配置区间
>      {
>       "to": 25
>      },
>      {
>       "from": 25,
>       "to": 35
>      },
>      {
>       "from": 35
>      }
>     ]
>    },
>    "aggs": {
>     "bmax": {
>      "max": {
>       "field": "balance"
>      }
>     }
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "age_range": {
>    "buckets": [
>     {
>      "key": "*-25.0",
>      "to": 25,
>      "doc_count": 225,
>      "bmax": {
>       "value": 49587
>      }
>     },
>     {
>      "key": "25.0-35.0",
>      "from": 25,
>      "to": 35,
>      "doc_count": 485,
>      "bmax": {
>       "value": 49795
>      }
>     },
>     {
>      "key": "35.0-*",
>      "from": 35,
>      "doc_count": 290,
>      "bmax": {
>       "value": 49989
>      }
>     }
>    ]
>   }
>  }
> }

### 2.5 时间范围聚合

[Date Range Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-daterange-aggregation.html)，日期范围聚合。基于日期类型的值，以【日期范围】来桶分聚合。期范围可以用各种 [Date Math](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/common-options.html#date-math) 表达式。同样的，包括 from 的值，不包括 to 的值。它属于multi-bucket。

> POST /bank/_search?size=0
> {
>  "aggs": {
>   "range": {
>    "date_range": {
>     "field": "date",
>     "format": "MM-yyy",
>     "ranges": [  //包含2个桶
>      {
>       "to": "now-10M/M"
>      },
>      {
>       "from": "now-10M/M"
>      }
>     ]
>    }
>   }
>  }
> }
> {//返回
>   ...
>  "aggregations": {
>   "range": {
>    "buckets": [
>     {
>      "key": "*-2017-08-01T00:00:00.000Z",
>      "to": 1501545600000,
>      "to_as_string": "2017-08-01T00:00:00.000Z",
>      "doc_count": 0
>     },
>     {
>      "key": "2017-08-01T00:00:00.000Z-*",
>      "from": 1501545600000,
>      "from_as_string": "2017-08-01T00:00:00.000Z",
>      "doc_count": 0
>     }
>    ]
>   }
>  }
> }

### 2.6 时间柱状聚合

你要先了解下[Histogram Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-histogram-aggregation.html)，直方图聚合。基于文档中的某个【数值类型】字段，通过计算来动态的分桶。它属于multi-bucket。

> {
>   "aggs" : {
>     "prices" : {
>       "histogram" : {
>         "field" : "price",   //字段，必须为数值类型
>         "interval" : 50,    //分桶间距
>         "min_doc_count" : 1,  //最少文档数桶过滤，只有不少于这么多文档的桶才会返回
>         "extended_bounds" : { //范围扩展
>           "min" : 0,
>           "max" : 500
>         },
>         "order" : { "_count" : "desc" },//对桶排序，如果 histogram 聚合有一个权值聚合类型的"直接"子聚合，那么排序可以使用子聚合中的结果
>         "keyed":true, //hash结构返回，默认以数组形式返回每一个桶
>         "missing":0 //配置缺省默认值
>       }
>     }
>   }

[Date Histogram Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-datehistogram-aggregation.html)，日期直方图聚。基于日期类型，以【日期间隔】来桶分聚合。可用的时间间隔类型为：year、quarter、month、week、day、hour、minute、second，其中，除了year、quarter 和 month，其余可用小数形式。

> POST /bank/_search?size=0
> {
>   "aggs" : {
>     "articles_over_time" : {
>       "date_histogram" : {
>         "field" : "date",
>         "interval" : "month",
>         "format" : "yyyy-MM-dd",  //定义日期的格式
>         "time_zone": "+08:00"    //定义时区，用作时间值的调整
>       }
>     }
>   }
> }

### 2.7 Missing Aggregation

[Missing Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-missing-aggregation.html)，缺失值的桶聚合

> POST /bank/_search?size=0
> {
>   "aggs" : {
>     "account_without_a_age" : {
>       "missing" : { "field" : "age" }
>     }
>   }
> }

### 2.8 IP范围聚合

[IP Range Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-iprange-aggregation.html)，基于一个 IPv4 字段，对文档进行【IPv4范围】的桶分聚合。和 Range Aggregation 类似，只是应用字段必须是 IPv4 数据类型。它属于multi-bucket。

> GET /ip_addresses/_search
> {
>   "size": 10,
>   "aggs" : {
>     "ip_ranges" : {
>       "ip_range" : {
>         "field" : "ip",
>         "ranges" : [
>           { "to" : "10.0.0.5" },
>           { "from" : "10.0.0.5" }
>         ]
>       }
>     }
>   }
> }
> {//返回
>   "aggregations": {
>     "ip_ranges": {
>       "buckets" : [
>         {
>           "to": "10.0.0.5",
>           "doc_count": 10
>         },
>         {
>           "from": "10.0.0.5",
>           "doc_count": 260
>         }
>       ]
>     }
>   }
> }

### 2.9 Nested Aggregation

[Nested Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-bucket-nested-aggregation.html)，嵌套类型聚合。基于嵌套（nested）数据类型，把该【嵌套类型的信息】聚合到单个桶里，然后就可以对嵌套类型做进一步的聚合操作。

## 3. 矩阵聚合

[Matrix](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-matrix.html)，矩阵聚合。此功能是实验性的，在将来的版本中可能会完全更改或删除。

它对**多个字段进行操作**并根据从请求的文档字段中提取的值生成矩阵结果的聚合系列。与度量聚合和桶聚合不同，此聚合系列尚不支持脚本编写。

## 4. 管道聚合

[Pipeline](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline.html)，管道聚合。它对其它聚合操作的输出（桶或者桶的某些权值）及其关联指标进行聚合，而不是文档，是一种后期对每个分桶的一些计算操作。管道聚合的作用是为输出增加一些有用信息。

管道聚合不能包含子聚合，但是某些类型的管道聚合可以链式使用（比如计算导数的导数）。

管道聚合大致分为两类：

- parent，它输入是其【父聚合】的输出，并对其进行进一步处理。一般不生成新的桶，而是对父聚合桶信息的增强。
- sibling，它输入是其【兄弟聚合】的输出。并能在同级上计算新的聚合。

管道聚合通过 buckets_path 参数指定他们要进行聚合计算的权值对象，**bucket_path语法**：

> 聚合分隔符 = ">"，                           指定父子聚合关系，如："my_bucket>my_stats.avg"
> 权值分隔符= "."，                            指定聚合的特定权值
> 聚合名称  = <name of the aggregation> ，       直接指定聚合的名称
> 权值      = <name of the metric> ，            直接指定权值
> 完整路径  = agg_name[> agg_name]*[. metrics] ， 综合利用上面的方式指定完整路径
> 特殊值    = "_count"，                       输入的文档个数

**特殊情况**

- 要进行 pipeline aggregation 聚合的对象名称或权值名称包含小数点，"buckets_path": "my_percentile[99.9]"
- 处理对象中包含空桶（无文档的桶分），参数 gap_policy，可选值有 skip、insert_zeros

### 4.1 Derivative Aggregation：parent

[Derivative Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-derivative-aggregation.html)，求导聚合。基于父聚合（只能是histogram或date_histogram类型）的某个权值，对权值求导。用于求导的权值必须是数值类型。封闭直方图（histogram）聚合的 min_doc_count 必须是 0。

> {
>   "aggs" : {
>     "sales_per_month" : {
>       "date_histogram" : {
>         "field" : "date",
>         "interval" : "month"
>       },
>       "aggs": {
>         "sales": {
>           "sum": {
>             "field": "price"
>           }
>         },
>         "sales_deriv": {    //对每个月销售总和 sales 求导
>           "derivative": {
>             "buckets_path": "sales"  //用于计算均值的权值路径，同级，直接用metric值
>           }
>         }
>       }
>     }
>   }
> }

### 4.2 Moving Average Aggregation：parent

[Moving Average Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-movavg-aggregation.html)，窗口平均值聚合。基于已经排序过的数据，计算出处在当前出口中数据的平均值。比如窗口大小为 5 ，对数据 1—10 的部分窗口平均值如下：

- (1 + 2 + 3 + 4 + 5) / 5 = 3
- (2 + 3 + 4 + 5 + 6) / 5 = 4
- (3 + 4 + 5 + 6 + 7) / 5 = 5
- etc

> POST /_search
> {
>   "size": 0,
>   "aggs": {
>     "my_date_histo":{
>       "date_histogram":{
>         "field":"date",
>         "interval":"1M"
>       },
>       "aggs":{
>         "the_sum":{
>           "sum":{ "field": "price" }
>         },
>         "the_movavg":{
>           "moving_avg":{
>             "buckets_path": "the_sum",//用于计算均值的权值路径
>             "window" : 30,       //窗口大小
>             "model" : "simple"     //移动模型
>           }
>         }
>       }
>     }
>   }
> }

![img](https://img-blog.csdnimg.cn/20190109164344345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2l0c29mdGNoZW5mZWk=,size_16,color_FFFFFF,t_70)

### 4.3 Bucket Script Aggregation：parent

[Bucket Script Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-bucket-script-aggregation.html)，桶脚本聚合。基于父聚合的【一个或多个权值】，对这些权值通过脚本进行运算。用于计算的父聚合必须是多桶聚合。用于计算的权值必须是数值类型。执行脚本必须要返回数值型结果。

### 4.4 Bucket Selector Aggregation：parent

[Bucket Selector Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-bucket-selector-aggregation.html)，桶选择器聚合。基于父聚合的【一个或多个权值】，通过脚本对权值进行计算，并决定父聚合的哪些桶需要保留，其余的将被丢弃。用于计算的父聚合必须是多桶聚合。用于计算的权值必须是数值类型。运算的脚本必须是返回 boolean 类型，如果脚本是脚本表达式形式给出，那么允许返回数值类型。

### 4.5 Serial Differencing Aggregation：parent

[Serial Differencing Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-serialdiff-aggregation.html)，串行差分聚合。基于父聚合（只能是histogram或date_histogram类型）的某个权值，对权值值进行差分运算，（取时间间隔，后一刻的值减去前一刻的值：f(X) = f(Xt) – f(Xt-n)）。用于计算的父聚合必须是多桶聚合。

### 4.6 Avg Max Min Sum：sibliing

[Avg Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-avg-bucket-aggregation.html)，桶均值聚合。基于兄弟聚合的某个权值，求所有桶的权值均值。使用规则：用于计算的兄弟聚合必须是多桶聚合。用于计算的权值必须是数值类型。

[Max Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-max-bucket-aggregation.html)，桶最大值聚合。基于兄弟聚合的某个权值，输出权值最大的那一个桶。使用规则同上

[Min Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-min-bucket-aggregation.html)，桶最小值聚合。基于兄弟聚合的某个权值，输出权值最小的一个桶。使用规则同上

[Sum Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-sum-bucket-aggregation.html)，桶求和聚合。基于兄弟聚合的权值，对所有桶的权值求和。使用规则同上

> POST /_search
> {
>  "size": 0,
>  "aggs": {
>   "sales_per_month": {
>    "date_histogram": {
>     "field": "date",
>     "interval": "month"
>    },
>    "aggs": {
>     "sales": {
>      "sum": {
>       "field": "price"
>      }
>     }
>    }
>   },
>   "avg_monthly_sales": {
>    "avg_bucket": {
>     "buckets_path": "sales_per_month>sales" 
>    }
>   }
>  }
> }

### 4.7 Stats Bucket Aggregation：sibliing

[Stats Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-stats-bucket-aggregation.html)，桶统计信息聚合。基于兄弟聚合的某个权值，对【桶的信息】进行一些统计学运算（总计多少个桶、所有桶中该权值的最大值、最小等）。用于计算的权值必须是数值类型。用于计算的兄弟聚合必须是多桶聚合类型。

> {
>   "aggs" : {
>     "sales_per_month" : {
>       "date_histogram" : {
>         "field" : "date",
>         "interval" : "month"
>       },
>       "aggs": {
>         "sales": {
>           "sum": {
>             "field": "price"
>           }
>         }
>       }
>     },
>     "stats_monthly_sales": { //对父聚合的每个桶（每月销售总和）的一些基本信息进行聚合
>       "stats_bucket": {
>         "buckets_paths": "sales_per_month>sales" 
>       }
>     }
>   }
> }
> {//输出结果
>   "aggregations": {
>    "sales_per_month": {
>      "buckets": [
>       {
>         "key_as_string": "2015/01/01 00:00:00",
>         "key": 1420070400000,
>         "doc_count": 3,
>         "sales": {
>          "value": 550
>         }
>       },
>       {
>         "key_as_string": "2015/02/01 00:00:00",
>         "key": 1422748800000,
>         "doc_count": 2,
>         "sales": {
>          "value": 60
>         }
>       },
>       {
>         "key_as_string": "2015/03/01 00:00:00",
>         "key": 1425168000000,
>         "doc_count": 2,
>         "sales": {
>          "value": 375
>         }
>       }
>      ]
>    },
>    "stats_monthly_sales": {     //注意，统计的是桶的信息
>      "count": 3,
>      "min": 60,
>      "max": 550,
>      "avg": 328.333333333,
>      "sum": 985
>    }
>   }
> }

### 4.8 Extended Stats Bucket Aggregation：sibliing

[Extended Stats Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-extended-stats-bucket-aggregation.html)，扩展桶统计聚合。基于兄弟聚合的某个权值，对【桶信息】进行一系列统计学计算（比普通的统计聚合多了一些统计值）。用于计算的权值必须是数值类型。用于计算的兄弟聚合必须是多桶聚合类型。

### 4.9 Percentiles Bucket Aggregation：sibliing

[Percentiles Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/search-aggregations-pipeline-percentiles-bucket-aggregation.html)，桶百分比聚合。基于兄弟聚合的某个权值，计算权值的百分比。用于计算的权值必须是数值类型。用于计算的兄弟聚合必须是多桶聚合类型。对百分比的计算是精确的（不像Percentiles Metric聚合是近似值），所以可能会消耗大量内存。

**总结**，ES默认给 *大多数* 字段启用 doc values，所以在一些搜索场景大大的节省了内存使用量，但是需要注意的是只有不分词的 string 类型的字段才能使用这种特性。使聚合运行在 `not_analyzed` 字符串而不是 `analyzed` 字符串，这样可以有效的利用 doc values 。![img](https://blog.csdn.net/alex_xfboy/article/details/85332335)
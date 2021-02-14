## Table API 和 Flink SQL 是什么

1. Flink 对批处理和流处理，提供了统一的上层 API
2. Table API 是一套内嵌在 Java 和 Scala 语言中的查询API，它允许以非常直观的方式组合来自一些关系运算符的查询
3. Flink 的 SQL 支持基于实现了 SQL 标准的 Apache Calcite

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531165328668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 基本程序结构

Table API 和 SQL 的程序结构，与流式处理的程序结构十分类似

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531170305218.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## POM

```java
<dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-planner_2.11</artifactId>
            <version>1.10.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-planner-blink_2.11</artifactId>
            <version>1.10.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-csv</artifactId>
            <version>1.10.0</version>
        </dependency>
123456789101112131415
```

## 简单实例

```java
import com.atguigu.bean.SensorReading
// 隐式转换
import org.apache.flink.streaming.api.scala._
import org.apache.flink.table.api.Table
// 隐式转换
import org.apache.flink.table.api.scala._

object Example {
  def main(args: Array[String]): Unit = {

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    val inputDStream: DataStream[String] = env.readTextFile("D:\\MyWork\\WorkSpaceIDEA\\flink-tutorial\\src\\main\\resources\\SensorReading.txt")

    val dataDstream: DataStream[SensorReading] = inputDStream.map(
      data => {
        val dataArray: Array[String] = data.split(",")
        SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble)
      })

    // 1. 基于env创建表环境
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)

    // 2. 基于tableEnv 将流转换成表
    val dataTable: Table = tableEnv.fromDataStream(dataDstream)

    // 3. 只输出id为sensor_1的id和温度值
    // 3.1 调用table api，做转换操作
    val resultTable: Table = dataTable
      .select("id, temperature")
      .filter("id == 'sensor_1'")

    // 3.2 直接调用SQL - 写sql实现转换
    tableEnv.registerTable("dataTable", dataTable) // 注册表
    val resultSqlTable: Table = tableEnv.sqlQuery(
      """
        |select
        |   id, temperature
        |from
        |   dataTable
        |where
        |   id = 'sensor_1'
        |""".stripMargin
    )

    // 4. 将表转换成流操作
    resultTable.toAppendStream[ (String, Double) ].print("table")
    resultSqlTable.toAppendStream[ (String, Double) ].print( " sql " )

    env.execute("table test job")

  }
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531170329692.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 新老版本Flink批流处理对比

TableEnvironment 是 flink 中集成 Table API 和 SQL 的核心概念，所有对表的操作都基于 TableEnvironment

1. 注册catalog
2. 在内部 catalog 中注册表
3. 执行 SQL 查询
4. 注册用户自定义函数
5. 将 DataStream 或 DataSet 转换为表
6. 保存对 ExecutionEnvironment 或 StreamExecutionEnvironment 的引用

```java
import com.atguigu.bean.SensorReading
import org.apache.flink.api.scala.ExecutionEnvironment
import org.apache.flink.streaming.api.scala._
import org.apache.flink.table.api.{EnvironmentSettings, TableEnvironment}
import org.apache.flink.table.api.scala._

object OldAndNewInStreamComparisonBatchTest {
  def main(args: Array[String]): Unit = {

    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 创建表执行环境
    // val TableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)


    // --------------    Old -> 老版本批流处理得方式    ---------------


    // 老版本planner的流式查询
    val oldStreamSettings: EnvironmentSettings = EnvironmentSettings
      .newInstance() // 创建实例  -> return new Builder()
      .useOldPlanner() // 用老版本得Planner
      .inStreamingMode() //流处理模式
      .build()
    // 创建老版本planner的表执行环境
    val oldStreamTableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env, oldStreamSettings)

    // 老版本得批处理查询
    val oldBatchEnv: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
    val oldBatchTableEnv: BatchTableEnvironment = BatchTableEnvironment.create(oldBatchEnv)



    // -------------------------------    new -> 新版本批流处理得方式    -------------------------------



    // 新版本planner的流式查询
    val newStreamSettings: EnvironmentSettings = EnvironmentSettings
      .newInstance()
      .useBlinkPlanner() // 用新版本得Blink得Planner(添加pom依赖)
      .inStreamingMode() // 流处理
      .build()

    val newStreamTableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env, newStreamSettings)


    // 新版本得批处理查询
    val newBatchSettings: EnvironmentSettings = EnvironmentSettings
      .newInstance()
      .useBlinkPlanner()
      .inBatchMode() // 批处理
      .build()

    val newBatchTableEnv: TableEnvironment = TableEnvironment.create( newBatchSettings )



    // 数据读入并转换
    val inputDStream: DataStream[String] = env.readTextFile("D:\\MyWork\\WorkSpaceIDEA\\flink-tutorial\\src\\main\\resources\\SensorReading.txt")
    val dataDStream: DataStream[SensorReading] = inputDStream.map(
      data => {
        val dataArray: Array[String] = data.split(",")
        SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble)
      }
    )

    
    env.execute(" table test jobs " )
  }
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531173555800.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 运行报错

**Static methods in interface require -target:jvm-1.8**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531180928538.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 表（Table）

1. TableEnvironment 可以注册目录 Catalog，并可以基于 Catalog 注册表
2. 表（Table）是由一个“标识符”（identifier）来指定的，由3部分组成：Catalog名、数据库（database）名和对象名
3. 表可以是常规的，也可以是虚拟的（视图，View）
4. 常规表（Table）一般可以用来描述外部数据，比如文件、数据库表或消息队列的数据，也可以直接从 DataStream转换而来
5. 视图（View）可以从现有的表中创建，通常是 table API 或者 SQL 查询的一个结果集

## 连接外部系统

官网：https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/connect.html

## 连接文件数据

连接外部系统在Catalog中注册表，直接调用tableEnv.connect()就可以，里面参数要传入一个**ConnectorDescriptor**，也就是connector描述器。对于文件系统的connector而言，flink内部已经提供了，就叫做**FileSystem()**。

定义表得数据来源，和外部建立连接
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531174840319.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

这是旧版本的csv格式描述器。由于它是非标的，跟外部系统对接并不通用，所以将被弃用，以后会被一个符合RFC-4180标准的新format描述器取代。新的描述器就叫`Csv()`，但flink没有直接提供，需要引入依赖`flink-csv`

定义从外部文件读取数据之后的格式化方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531175121915.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

定义表结构

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531175445177.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

```java
import org.apache.flink.streaming.api.scala._
import org.apache.flink.table.api.{DataTypes, Table}
import org.apache.flink.table.api.scala._
import org.apache.flink.table.descriptors.{FileSystem, OldCsv, Schema}

object CreateTableTest {
  def main(args: Array[String]): Unit = {

    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 创建表环境
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)

    // --------------------- 读取文件数据 ---------------------------
    val filePath = "D:\\MyWork\\WorkSpaceIDEA\\flink-tutorial\\src\\main\\resources\\SensorReading.txt"

    tableEnv.connect( new FileSystem().path(filePath) ) // 定义表的数据来源，和外部系统建立连接
      .withFormat( new OldCsv() ) // 定义从外部文件读取数据之后的格式化方法
      .withSchema( new Schema() // 定义表结构
          .field("id", DataTypes.STRING())
          .field("timestamp", DataTypes.BIGINT())
          .field("temperature", DataTypes.DOUBLE())
      )
      .createTemporaryTable( "inputTable" ) // 在表环境中注册一张表(创建)


    // 测试输出
    val sensorTable: Table = tableEnv.from( "inputTable" )
    val resultTable: Table = sensorTable
        .select('id, 'temperature) // 查询id和temperature字段
        .filter('id === "sensor_1") // 输出sensor_1得数据
    resultTable.toAppendStream[ (String, Double) ].print( "FileSystem" )


    env.execute(" table connect fileSystem test job")
  }
}
1234567891011121314151617181920212223242526272829303132333435363738
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531181040229.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 连接Kafka 消费Kafka得数据

kafka的连接器flink-kafka-connector中，1.10版本的已经提供了Table API的支持。我们可以在 connect方法中直接传入一个叫做Kafka的类，这就是kafka连接器的描述器ConnectorDescriptor。

```java
import org.apache.flink.streaming.api.scala._
import org.apache.flink.table.api.{DataTypes, Table}
import org.apache.flink.table.api.scala._
import org.apache.flink.table.descriptors.{Csv, FileSystem, Kafka, OldCsv, Schema}

object CreateTableFromKafkaTest {
  def main(args: Array[String]): Unit = {

    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 创建表环境
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)

    // --------------------- 消费Kafka数据 ---------------------------

    tableEnv.connect( new Kafka()
        .version( "0.11" ) // 版本
        .topic( "sensor" ) // 主题
        .property("zookeeper.connect", "hadoop102:2181")
        .property("bootstrap.servers", "hadoop102:9092")
    )
      .withFormat( new Csv() ) // 新版本得Csv
      .withSchema( new Schema()
        .field("id", DataTypes.STRING())
        .field("timestamp", DataTypes.BIGINT())
        .field("temperature", DataTypes.DOUBLE())
      )


    // 测试输出
    val sensorTable: Table = tableEnv.from( "inputTable" )
    val resultTable: Table = sensorTable
      .select('id, 'temperature) // 查询id和temperature字段
      .filter('id === "sensor_1") // 输出sensor_1得数据
    resultTable.toAppendStream[ (String, Double) ].print( "FileSystem" )


    env.execute(" table connect fileSystem test job")
  }
}
1234567891011121314151617181920212223242526272829303132333435363738394041
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531181820729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 表的查询

利用外部系统的连接器connector，我们可以读写数据，并在环境的Catalog中注册表。接下来就可以对表做查询转换了。
Flink给我们提供了两种查询方式：**Table API和 SQL。**

## 表的查询 – Table API

这里Table API里指定的字段，前面加了一个单引号’，这是Table API中定义的Expression类型的写法，可以很方便地表示一个表中的字段。
字段可以直接全部用双引号引起来，也可以用半边单引号+字段名的方式。以后的代码中，一般都用后一种形式。

```java
import org.apache.flink.streaming.api.scala._
import org.apache.flink.table.api.{DataTypes, Table}
import org.apache.flink.table.api.scala._
import org.apache.flink.table.descriptors.{FileSystem, OldCsv, Schema}

object QueryTableAPITest {
  def main(args: Array[String]): Unit = {

    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 创建表环境
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)

    // --------------------- 读取文件数据 ---------------------------
    val filePath = "D:\\MyWork\\WorkSpaceIDEA\\flink-tutorial\\src\\main\\resources\\SensorReading.txt"

    tableEnv.connect( new FileSystem().path(filePath) ) // 定义表的数据来源，和外部系统建立连接
      .withFormat( new OldCsv() ) // 定义从外部文件读取数据之后的格式化方法
      .withSchema( new Schema() // 定义表结构
        .field("id", DataTypes.STRING())
        .field("timestamp", DataTypes.BIGINT())
        .field("temperature", DataTypes.DOUBLE())
      )
      .createTemporaryTable( "inputTable" ) // 在表环境中注册一张表(创建)


    // ----------------------- 表得查询 ---------------------

    val sensorTable: Table = tableEnv.from( "inputTable" )

    // 1. Table API的调用

    // 简单查询
    val resultTable: Table = sensorTable
      .select('id, 'temperature) // 查询id和temperature字段
      .filter('id === "sensor_1") // 输出sensor_1得数据

    // 聚合查询
    val aggResultTable: Table = sensorTable
        .groupBy('id)
        .select('id, 'id.count as 'count)


    // 测试输出
    resultTable.toAppendStream[ (String, Double) ].print( "easy" )
    aggResultTable.toAppendStream[ (String, Int) ].print( "agg" )

    env.execute(" tableAPI query test job")
  }
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531183148262.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

这里在测试得时候发现会报错一个并不是简单得插入追加操作，这里指得是，在读取数据得时候，如果做聚合操作，相当于把前面得数据做了更改或者覆盖操作，Flink不允许聚合查询出现这种。如果是简单查询，来一条数据根据条件就会输出一条，但是聚合查询会如首先出现sensor_1得数据，当第二条seneor_1得数据来时做聚合操作，将要更改原有得数据，不然如果后面得数据在累加，前面得数据还存在，聚合操作得意义不大
这里得更改意见是 修改成`toRetractStream()`

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020053118374221.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

## 表的查询 – SQL

1. Flink 的 SQL 集成，基于实现 了SQL 标准的 Apache Calcite
2. 在 Flink 中，用常规字符串来定义 SQL 查询语句
3. SQL 查询的结果，也是一个新的 Table

```java
import org.apache.flink.streaming.api.scala._
import org.apache.flink.table.api.scala._
import org.apache.flink.table.api.{DataTypes, Table}
import org.apache.flink.table.descriptors.{Csv, FileSystem, OldCsv, Schema}

object QueryTableSQLTest {
  def main(args: Array[String]): Unit = {

    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 创建表环境
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)

    // --------------------- 读取文件数据 ---------------------------
    val filePath = "D:\\MyWork\\WorkSpaceIDEA\\flink-tutorial\\src\\main\\resources\\SensorReading.txt"

    tableEnv.connect( new FileSystem().path(filePath) ) // 定义表的数据来源，和外部系统建立连接
      .withFormat( new Csv() ) // 定义从外部文件读取数据之后的格式化方法
      .withSchema( new Schema() // 定义表结构
        .field("id", DataTypes.STRING())
        .field("timestamp", DataTypes.BIGINT())
        .field("temperature", DataTypes.DOUBLE())
      )
      .createTemporaryTable( "inputTable" ) // 在表环境中注册一张表(创建)


    // ----------------------- 表得查询 ---------------------

    val sensorTable: Table = tableEnv.from( "inputTable" )

    tableEnv.registerTable("sensorTable", sensorTable) // 注册表
    val resultSqlTable: Table = tableEnv.sqlQuery(
      """
        |select
        |   id, temperature
        |from
        |   sensorTable
        |where
        |   id = 'sensor_1'
        |""".stripMargin
    )

    val aggResultSqlTable: Table = tableEnv
      .sqlQuery("select id, count(id) as cnt from sensorTable group by id")


    // 测试输出
    resultSqlTable.toAppendStream[ (String, Double) ].print( "easy " )
    aggResultSqlTable.toRetractStream[ (String, Long) ].print( "agg" )

    env.execute(" tableSQL query test job")
  }
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200531185419880.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)
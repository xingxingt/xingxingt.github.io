---
layout:     post
title:      Spark&Hbase操作
subtitle:   Spark&Hbase的基本操作操作
date:       2018-09-26
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Hbase
---

>
>参考:http://wuchong.me/blog/2015/04/06/spark-on-hbase-new-api/
> 

### HBase 新版 API 进行 CRUD 基本操作
#### 配置环境
    <properties>
        <scala-version>2.11.8</scala-version>
        <spark-version>2.1.0</spark-version>
        <hadoop-version>2.6.0</hadoop-version>
        <hbase-version>1.4.7</hbase-version>
    </properties>
    
    <dependency>
        <groupId>org.apache.hbase</groupId>
        <artifactId>hbase-client</artifactId>
        <version>${hbase-version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.hbase</groupId>
        <artifactId>hbase-server</artifactId>
        <version>${hbase-version}</version>
    </dependency>
    <dependency>
        <groupId>org.scala-lang</groupId>
        <artifactId>scala-library</artifactId>
        <version>${scala-version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>${spark-version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_2.11</artifactId>
        <version>${spark-version}</version>
    </dependency>
    
    
#### Hbase基本操作
新版 API 中加入了 Connection，HAdmin成了Admin，HTable成了Table，而Admin和Table只能通过Connection获得.
Connection的创建是个重量级的操作，由于Connection是线程安全的，所以推荐使用单例，
其工厂方法需要一个HBaseConfiguration.
    
    val conf = HBaseConfiguration.create()
    conf.set("hbase.zookeeper.property.clientPort", "2181")
    conf.set("hbase.zookeeper.quorum", "master")
    //Connection 的创建是个重量级的工作，线程安全，是操作hbase的入口
    val conn = ConnectionFactory.createConnection(conf)

使用Admin创建和删除表
    
    val userTable = TableName.valueOf("user")
    //创建 user 表
    val tableDescr = new HTableDescriptor(userTable)
    tableDescr.addFamily(new HColumnDescriptor("basic".getBytes))
    println("Creating table `user`. ")
    if (admin.tableExists(userTable)) {
      admin.disableTable(userTable)
      admin.deleteTable(userTable)
    }
    admin.createTable(tableDescr)
    println("Done!")
      

插入、查询、扫描、删除操作
HBase 上的操作都需要先创建一个操作对象Put,Get,Delete等，然后调用Table上的相对应的方法

    try{
    //获取 user 表
    val table = conn.getTable(userTable)
      try{
        //准备插入一条 key 为 id001 的数据
        val p = new Put("id001".getBytes)
        //为put操作指定 column 和 value （以前的 put.add 方法被弃用了）
        p.addColumn("basic".getBytes,"name".getBytes, "wuchong".getBytes)
        //提交
       table.put(p)
       //查询某条数据
       val g = new Get("id001".getBytes)
       val result = table.get(g)
       val value = Bytes.toString(result.getValue("basic".getBytes,"name".getBytes))
       println("GET id001 :"+value)
       //扫描数据
       val s = new Scan()
       s.addColumn("basic".getBytes,"name".getBytes)
       val scanner = table.getScanner(s)
       try{
          for(r <- scanner){
            println("Found row: "+r)
            println("Found value: "+Bytes.toString(
              r.getValue("basic".getBytes,"name".getBytes)))
          }
        }finally {
         //确保scanner关闭
         scanner.close()
       }
       //删除某条数据,操作方式与 Put 类似
       val d = new Delete("id001".getBytes)
       d.addColumn("basic".getBytes,"name".getBytes)
       table.delete(d)
     }finally {
       if(table != null) table.close()
     }
    }finally {
      conn.close()
    }
     
### Spark 操作 HBase
#### 写入HBase
首先要向 HBase 写入数据，我们需要用到PairRDDFunctions.saveAsHadoopDataset。
因为 HBase 不是一个文件系统，所以saveAsHadoopFile方法没用。这个方法需要一个 
JobConf 作为参数，类似于一个配置项，主要需要指定输出的格式和输出的表名。

##### Step 1：我们需要先创建一个 JobConf.

    //定义 HBase 的配置
    val conf = HBaseConfiguration.create()
    conf.set("hbase.zookeeper.property.clientPort", "2181")
    conf.set("hbase.zookeeper.quorum", "master")
    //指定输出格式和输出表名
    val jobConf = new JobConf(conf,this.getClass)
    jobConf.setOutputFormat(classOf[TableOutputFormat])
    jobConf.set(TableOutputFormat.OUTPUT_TABLE,"user")

##### Step 2： RDD 到表模式的映射
在 HBase 中的表 schema 一般是这样的：

    row     cf:col_1    cf:col_2

而在Spark中，我们操作的是RDD元组，比如(1,"lilei",14), (2,"hanmei",18)。我们需要将
RDD[(uid:Int, name:String, age:Int)] 转换成 RDD[(ImmutableBytesWritable, Put)]。
所以，我们定义一个 convert 函数做这个转换工作

    def convert(triple: (Int, String, Int)) = {
      val p = new Put(Bytes.toBytes(triple._1))
      p.addColumn(Bytes.toBytes("basic"),Bytes.toBytes("name"),Bytes.toBytes(triple._2))
      p.addColumn(Bytes.toBytes("basic"),Bytes.toBytes("age"),Bytes.toBytes(triple._3))
      (new ImmutableBytesWritable, p)
    }

##### Step 3： 读取RDD并转换

    //read RDD data from somewhere and convert
    val rawData = List((1,"lilei",14), (2,"hanmei",18), (3,"someone",38))
    val localData = sc.parallelize(rawData).map(convert)
    
##### Step 4： 使用saveAsHadoopDataset方法写入HBase

    localData.saveAsHadoopDataset(jobConf)

#### 读取 HBase

Spark读取HBase，我们主要使用SparkContext 提供的newAPIHadoopRDDAPI/saveAsHadoopDataset将表的内容以 RDDs 的形式加载到 Spark 中。

    val conf = HBaseConfiguration.create()
    conf.set("hbase.zookeeper.property.clientPort", "2181")
    conf.set("hbase.zookeeper.quorum", "master")
    //设置查询的表名
    conf.set(TableInputFormat.INPUT_TABLE, "user")
    val usersRDD = sc.newAPIHadoopRDD(conf, classOf[TableInputFormat],
      classOf[org.apache.hadoop.hbase.io.ImmutableBytesWritable],
      classOf[org.apache.hadoop.hbase.client.Result])
    val count = usersRDD.count()
    println("Users RDD Count:" + count)
    usersRDD.cache()
    //遍历输出
    usersRDD.foreach{ case (_,result) =>
      val key = Bytes.toInt(result.getRow)
      val name = Bytes.toString(result.getValue("basic".getBytes,"name".getBytes))
      val age = Bytes.toInt(result.getValue("basic".getBytes,"age".getBytes))
      println("Row key:"+key+" Name:"+name+" Age:"+age)
    }



#####  [个人详细案例请点击这里](https://github.com/xingxingt/centrecode/tree/master/src/main/scala/cn/spark/hbase)



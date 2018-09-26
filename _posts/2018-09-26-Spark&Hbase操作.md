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
##### 新版 API 中加入了 Connection，HAdmin成了Admin，HTable成了Table，而Admin和Table只能通过Connection获得.Connection的创建是个重量级的操作，由于Connection是线程安全的，所以推荐使用单例，其工厂方法需要一个HBaseConfiguration.
    
    val conf = HBaseConfiguration.create()
    conf.set("hbase.zookeeper.property.clientPort", "2181")
    conf.set("hbase.zookeeper.quorum", "master")
    //Connection 的创建是个重量级的工作，线程安全，是操作hbase的入口
    val conn = ConnectionFactory.createConnection(conf)

##### 使用Admin创建和删除表
    
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
      

### 插入、查询、扫描、删除操作
#### HBase 上的操作都需要先创建一个操作对象Put,Get,Delete等，然后调用Table上的相对应的方法

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
      
    

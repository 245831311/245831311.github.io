---
title: Spark Sql基础知识点
date: 2021-08-27 11:16:39
categories: 
- 大数据
tags:
- 大数据
---

# Spark Sql基础知识点

## Spark Sql 总体图

![spark-sql](/images/bdata/spark-sql.png)

## Spark Sql 功能图

## 基础知识点

DataSet概念：
DataSet代表被分配的数据集合

DataFrame概念：
DataFrame是dataSet组织的列名集合。可看作是关系数据库的二维数据表，但相比于此，更具有丰富优化。它可从多个数据源构建，如hive tables、已有RDD、文件等


## 思考

为什么要使用Spark Sql？Hive Sql不是能够足够支持？

因为Hive中底层执行sql是需要转化成mapreduce程序来执行，众所周知，mapreduce的程序计算模型效率相对比较慢，而且程序的复杂性会相对比较高。因此，Spark Sql能有效解决这个问题，因为是RDD执行效率会更快。

## demo例子

1.读取json文件demo
```Java
SparkConf conf = new SparkConf().setAppName("default").setMaster("local");//.setJars(jars);
        conf.set("spark.executor.memory", "512m");
        conf.set("spark.executor.cores", "1");
        sc = new JavaSparkContext(conf);
        
        //创建spark session
        SparkSession spark = SparkSession
        		  .builder()
        		  .appName("Java Spark SQL basic example")
        		  .config("spark.some.config.option", "some-value")
        		  .getOrCreate();
		
		Dataset<Row> df = spark.read().json("./src/main/java/org/jay/spark/areaInfo.json");

		Dataset<Row> df2 = df.select("address");
		
		df2.show();
		
		
		//输出schema
		df.printSchema();

		//Register the DataFrame as a SQL temporary view
		df.createOrReplaceTempView("people");
		Dataset<Row> namesDF = spark.sql("SELECT address.city FROM people");		
		//数据输出
		namesDF.show();
```

2.创建schema的例子
```Java

SparkConf conf = new SparkConf().setAppName("default").setMaster("local");//.setJars(jars);
        conf.set("spark.executor.memory", "512m");
        conf.set("spark.executor.cores", "1");
        sc = new JavaSparkContext(conf);
        
        //创建spark session
        SparkSession spark = SparkSession
        		  .builder()
        		  .appName("Java Spark SQL basic example")
        		  .config("spark.some.config.option", "some-value")
        		  .getOrCreate();
		
        
        JavaRDD<HostBean> rdd = spark.read().text("./src/main/java/org/jay/spark/itcast.log").javaRDD().map(new Function<Row, HostBean>() {

			@Override
			public HostBean call(Row v1) throws Exception {
				String line = v1.toString();
				String[] strs = line.split("\t");
				HostBean bean = new HostBean();
				bean.setHost(strs[1]);
				bean.setTime(strs[0]);
				return bean;
			}
		});

        //创建schema
        List<StructField> fields = new ArrayList<>();
        for (String fieldName : "time host".split(" ")) {
          StructField field = DataTypes.createStructField(fieldName, DataTypes.StringType, true);
          fields.add(field);
        }
        StructType schema = DataTypes.createStructType(fields);
        //将rdd转为row
        JavaRDD<Row> rowRDD = rdd.map((Function<HostBean, Row>) record -> {
        	  return RowFactory.create(record.getTime(), record.getHost());
        	});
        //创建df
        Dataset<Row> df = spark.createDataFrame(rowRDD,schema);
        
        //Dataset<Row> df = spark.createDataFrame(rdd, HostBean.class);
        
        df.createOrReplaceTempView("test");
        
        Dataset<Row> host = spark.sql("SELECT name FROM test");

```


# 参考网站

1.[Spark Sql](http://spark.apache.org/sql/)

2.[Spark Sql Guide](http://spark.apache.org/docs/latest/sql-programming-guide.html)


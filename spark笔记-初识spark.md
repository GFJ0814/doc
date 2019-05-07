spark笔记-初识spark （参考书籍：深入理解spark核心思想与源码分析）

1.spark是什么，都有什么，怎么来的、有什么特点。

​	spark：快速而通用的集群计算平台，核心是一个由很多计算任务组成的，运行在多个工作机器或者是一个计算集群上的应用进行调度、分发、监控的计算引擎。

​	快速：可以基于内存进行计算，扩展了Mapreduce  快速可以体现在：支持交互式查询和流式计算

​	通用：自身拥有支持丰富语言的API和类库 Python、java、Scala、SQL，与其他大数据工具可以很好配合，比如可以运行在hadoop集群上。



大一统的软件栈：

​	包含了：sparkCore（Rdd的定义与操作、任务调度、内存管理、错误恢复、与存储系统交互）、SparkSQL SparkStreaming、机器学习的Mlib、图运算的Graphx、支持集群运算的集群管理器CM

​        组件之间密切联系可相互调用。设计好处：1、底层优化，高层应用受益  2、spark一套系统替代了原来的多套系统部署维护。3、支持构建无缝整合不同处理模型的应用，比如可以对流处理数据进行机器学习算法，支持SQL查询分析。



spark简史

​	1、诞生于2009 加州伯克利分校RAD实验室、因为觉得MapReduce的在迭代运算和交互运算上的效率低下所以研发的，一开始的项目是作为机器学习领域研究项目，用spark监控并预测旧金湾区的交通拥堵情况。后来外部机构也开始用了。

​	2、2009年，spark研究论文发表、同年spark项目诞生

​	3、2010年3月开源 

​	4、2013年6月交给Apache基金会 现在已经是顶级项目





​	

spark笔记-下载、安装spark

环境准备：

​	（1）jdk 因为spark用的是Scala语言，需要依赖JVM

​	（2）如果需要用到pyspark，要装Python解释器

​	（3）登录官网下载就行，解压缩可以了，没啥好说的。安装spark没必要安装hadoop，但是安装了hadoop的情况下，要下载对应版本的spark。



打开spark-shell：用来做即时数据分析   cd bin  ./spark-shell即可

交互式shell提供了两个版本：Python的pyspark 和Scala版本的spark-shell



spark核心概念简介：

​	RDD：对用于spark集群上的数据集就是RDD（resilient distributed dataset 弹性分布式数据集） 是spark对分布式数据和运算的基本抽象

​	驱动程序（Driver）：用于发起集群上的并行操作。包含了程序执行的main函数，定义RDD，以及如操作RDD。shell交互中的驱动程序就是shell本身。



​	sparkContext：和集群的一个连接，Driver通过sparkContext来访问spark，shell启动时会自己创建一个sparkContext对象。



​	executor：执行节点：由驱动程序统一管理，各个节点并行执行运算任务。



单机部署独立的spark应用（java版本的Wordcount）：

pom文件：

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.study</groupId>
    <artifactId>TestSpark</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>2.3.2</version>
            <scope>provided</scope>
        </dependency>

    </dependencies>


</project>
```

核心代码：

```java
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Iterator;


public class WordCount {
    public static void main(String[] args) {
        //创建一个sparkConf对象
        SparkConf conf = new SparkConf().setMaster("local").setAppName("My App");
        //创建sparkContext对象
        JavaSparkContext sc = new JavaSparkContext(conf);
        String inputFile ="/root/testSpark.txt";
        JavaRDD<String> input = sc.textFile(inputFile);

        System.out.println(input.first());
        JavaRDD<String> words = input.flatMap(new FlatMapFunction<String, String>() {
            public Iterator<String> call(String s) throws Exception {
                return Arrays.asList(s.split(" ")).iterator();
            }
        });
        JavaPairRDD<String,Integer> counts = words.mapToPair(new PairFunction<String, String, Integer>() {
            public Tuple2<String, Integer> call(String s) {
                return new Tuple2(s,1);
            }
        });


        JavaPairRDD<String,Integer> result = counts.reduceByKey(new Function2<Integer, Integer, Integer>() {
            public Integer call(Integer integer, Integer integer2) throws Exception {
                return integer+integer2;
            }
        });

        result.saveAsTextFile("/root/testWordCount");
    }
}
```

部署应用，对/root目录下的testSpark.text文件进行Wordcount：

先看下testspark.text文件的内容：

```
[root@izbp137hlazpqpuwrp37qsz ~]# cat testSpark.txt 
hello spark
hello bigData
hello world
```

（1）在我自己的阿里云上新建一个文件夹project，打包该测试应用mvn package,将生成的jar包上传到project下，如图：

```
[root@izbp137hlazpqpuwrp37qsz ~]# cd project/
[root@izbp137hlazpqpuwrp37qsz project]# ll
total 8
-rw-r--r-- 1 root root 4476 Dec 21 16:05 TestSpark-1.0-SNAPSHOT.jar
[root@izbp137hlazpqpuwrp37qsz project]# pwd
/root/project
```

（2）切换到spark命令目录，执行该程序：
```
[root@izbp137hlazpqpuwrp37qsz project]# cd /root/spark-2.0.2-bin-hadoop2.6/bin/
[root@izbp137hlazpqpuwrp37qsz bin]# ./spark-submit --class WordCount  /root/project/TestSpark-1.0-SNAPSHOT.jar 
```

计算结果展示,在testWord：
```
[root@izbp137hlazpqpuwrp37qsz ~]# cd testWordCount/
[root@izbp137hlazpqpuwrp37qsz testWordCount]# ll
total 4
-rw-r--r-- 1 root root 42 Dec 28 17:23 part-00000
-rw-r--r-- 1 root root  0 Dec 28 17:23 _SUCCESS
[root@izbp137hlazpqpuwrp37qsz testWordCount]# cat part-00000 
(spark,1)
(hello,3)
(bigData,1)
(world,1)
[root@izbp137hlazpqpuwrp37qsz testWordCount]# 
```



spark笔记-RDD

1.什么是RDD 怎么解释弹性，怎么创建，RDD怎么操作，惰性计算是什么，持久化方式，转化、行动是什么，设计背景





这么创建的：






---
layout: post
title: '3min阅读: Spark在AWS "centralized VPC Endpoint"环境下无法正常消费kinesis stream 数据 '
date: 2024-11-09
author: Calvin Wang
# cover: '/assets/img/202209/saml.drawio.png'
tags: spark streaming kinesis
---

>  发现一年多没更新。 写这篇两个目的： 1. 记录一下一个比较有意思的错误  2. 借着这个错误一块读读代码分析下问题

## 遇到了什么问题？
在使用Spark Streaming(Glue) 消费Kinesis stream(下称kinesis)数据时遇到了错误。
错误信息主要是：

```text
query terminated with error. 
Error while List shared. 
balabal... 
connect timeout
```

## 定位错误：
第一眼看上去这错误不是很简单。 从错误信息`connect time` 和 `Error List shared`来看要么是网络错误， 要么是权限错误（虽然错误信息和标准的权限错误信息不一样）。
这个就很好验证， 在同样的环境直接写一段消费kinesis数据的代码(不依赖Spark Streaming)，问题就能定位了。
* 结果这段代码跑的很顺畅！

现在问题就变得有意思了！这段Spark Streaming 代码十分简单， 几乎就是官网的demo，只是按照文档修改了几个必要的参数。并且类似的代码在其它AWS账户内运行的很好。那可能的错误就是我们修改的几个参数了。
* 多轮测试下来， 问题出现在`endpoint`配置。 当本账户创建了kinesis 的private endpoint（下称VPCE）并启用DNS解析， 那一切正常。 当仅创建VPCE 或者使用通过Transition Gateway连接账户下的VPCE(centralized vpce)并且没有DNS解析时会出现错误。
> 标准带地区的Endpoint name 类似： kinesis.cn-north-1.amazonaws.com.cn
> VPCE的私有域名类似： vpce-xxxxxxxx.kinesis.cn-north-1.vpce.amazonaws.com.cn

就很巧我们的账户情况就是Centralized VPCE的模式。 就像下图， 但是我们的环境没有Route53(也是为啥要在标题里把Centralized VPCE 打上引号)。
![](/assets/img/2024/centralizing-interface-vpc-endpoints.png)

* 定位到问题以后就很好修复了。 但是呢咱们也想看看这个错误是怎么产生的。

## 分析错误产生原因：
首先呢，查看Spark-Kinesis 这个包是如何维护VPCE的。
如下先打开 Kinesis Receiver 文件[link](https://github.com/apache/spark/blob/v3.4.1/connector/kinesis-asl/src/main/scala/org/apache/spark/streaming/kinesis/KinesisReceiver.scala)可以看到Spark这边只是把配置信息封装到 kinesisClientLibConfiguration 对象中，然后传递给Worker去消费数据了(Worker 来自com.amazonaws.services.kinesis.clientlibrary.lib.worker) 
* 那这个错误大概率发生在AWS的代码那里了。

```java
val kinesisClientLibConfiguration = {
      val baseClientLibConfiguration = new KinesisClientLibConfiguration(
        checkpointAppName,
        streamName,
        kinesisProvider,
        dynamoDBCreds.map(_.provider).getOrElse(kinesisProvider),
        cloudWatchCreds.map(_.provider).getOrElse(kinesisProvider),
        workerId)
        .withKinesisEndpoint(endpointUrl)
        .withTaskBackoffTimeMillis(500)
        .withRegionName(regionName)
        .withMetricsLevel(metricsLevel)
        .withMetricsEnabledDimensions(metricsEnabledDimensions.asJava)
/* 折叠N行 */
worker = new Worker(recordProcessorFactory, kinesisClientLibConfiguration)
```

如下，咱们打开Worker的代码, 这里包含大量的重载initial函数， 顺着读， 最后会发现这么一段代码, 显示使用了VPCE会用VPCE的参数覆盖Region配置。[link](https://github.com/eroad/amazon-kinesis-client/blob/23cdfca358058d94106f4cfaff432e996d14cd1a/src/main/java/com/amazonaws/services/kinesis/clientlibrary/lib/worker/Worker.java#L245)

* 看注释错误应该就是在这里`setEndpoint`发生的了。他的本意应该是像支持kinesis跨账户消费， 但是没考虑到我们现在这种情况。

```java
  // If a kinesis endpoint was explicitly specified, use it to set the region of kinesis.
        if (config.getKinesisEndpoint() != null) {
            kinesisClient.setEndpoint(config.getKinesisEndpoint());
            if (config.getRegionName() != null) {
                LOG.warn("Received configuration for both region name as " + config.getRegionName()
                        + ", and Amazon Kinesis endpoint as " + config.getKinesisEndpoint()
                        + ". Amazon Kinesis endpoint will overwrite region name.");
                LOG.debug("The region of Amazon Kinesis client has been overwritten to " + config.getKinesisEndpoint());
            } else  {
                LOG.debug("The region of Amazon Kinesis client has been set to " + config.getKinesisEndpoint());
            }
        }
```

最后我没找到kinesisClient.setEndpoint这段代码， 只能通过API文档反推。[link](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/AmazonWebServiceClient.html#setEndpoint-java.lang.String-)

* 如下图， 它有两个重载setEndpoint函数。 第一个只接受Endpoint URL。 第二个接受Endpoint URL， Service， Region。 推测是第一个重载函数从Endpoint URL里解析了需要的参数然后调用了第二个重载的setEndpoint函数。

![](/assets/img/2024/kinesisclient-setendpoint.png)


> 最后： KPI 达成！

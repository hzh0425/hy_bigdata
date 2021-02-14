# 1.什么是主副分片

1：当新建一个索引库时，可以预先设置其会被分为N个分片（主分片），同时可以为每个主分片产生N个备份分片（副分片）。

2：N个主分片随机分布在集群的多个节点中；N个副分片也是随机的分布在集群的多个节点中，但是副分片和其主分片不会在一个节点上。

# 2.PacificA 算法

![image-20210204202815401](https://gitee.com/zisuu/picture/raw/master/img/20210204202815.png)

![image-20210204203019313](https://gitee.com/zisuu/picture/raw/master/img/20210204203019.png)

## 1.数据副本策略

![image-20210204203301039](https://gitee.com/zisuu/picture/raw/master/img/20210204203301.png)

## 2.配置管理

![image-20210204203345367](https://gitee.com/zisuu/picture/raw/master/img/20210204203345.png)

## 3.错误监测

![image-20210204203410297](https://gitee.com/zisuu/picture/raw/master/img/20210204203410.png)

# 3.ES的数据副本模型

![image-20210204204132481](https://gitee.com/zisuu/picture/raw/master/img/20210204204132.png)

## 3.1基本写入模型

![image-20210204204211802](https://gitee.com/zisuu/picture/raw/master/img/20210204204211.png)

## 3.2基本写入模型

![image-20210204204254505](https://gitee.com/zisuu/picture/raw/master/img/20210204204254.png)

# 4.Allocation IDS

![image-20210204204445000](https://gitee.com/zisuu/picture/raw/master/img/20210204204445.png)

## 4.1安全分配主分片

![image-20210204204625245](https://gitee.com/zisuu/picture/raw/master/img/20210204204625.png)

## 4.2 一个列子

![image-20210204204712736](https://gitee.com/zisuu/picture/raw/master/img/20210204204712.png)

![image-20210204204721079](https://gitee.com/zisuu/picture/raw/master/img/20210204204721.png)



![image-20210204204737198](https://gitee.com/zisuu/picture/raw/master/img/20210204204737.png)

![image-20210204204746077](https://gitee.com/zisuu/picture/raw/master/img/20210204204746.png)

![image-20210204204758981](https://gitee.com/zisuu/picture/raw/master/img/20210204204759.png)

![image-20210204204808720](https://gitee.com/zisuu/picture/raw/master/img/20210204204808.png)

![image-20210204204822064](https://gitee.com/zisuu/picture/raw/master/img/20210204204822.png)

![image-20210204204828286](https://gitee.com/zisuu/picture/raw/master/img/20210204204828.png)

![image-20210204204841278](https://gitee.com/zisuu/picture/raw/master/img/20210204204841.png)

![image-20210204204849857](https://gitee.com/zisuu/picture/raw/master/img/20210204204849.png)



![image-20210204204904667](https://gitee.com/zisuu/picture/raw/master/img/20210204204904.png)

![image-20210204204934591](https://gitee.com/zisuu/picture/raw/master/img/20210204204934.png)

# 5.Sequence IDS

![image-20210204205055853](https://gitee.com/zisuu/picture/raw/master/img/20210204205055.png)



## 5.1 Primary Terms和Sequence Numbers

![image-20210204205133048](https://gitee.com/zisuu/picture/raw/master/img/20210204205133.png)

![image-20210204205150143](https://gitee.com/zisuu/picture/raw/master/img/20210204205150.png)

![image-20210204205202537](https://gitee.com/zisuu/picture/raw/master/img/20210204205202.png)

## 5.2本地及全局检查点

![image-20210204205230647](https://gitee.com/zisuu/picture/raw/master/img/20210204205230.png)

![image-20210204205312226](https://gitee.com/zisuu/picture/raw/master/img/20210204205312.png)

![image-20210204205326565](https://gitee.com/zisuu/picture/raw/master/img/20210204205326.png)

![image-20210204205345349](https://gitee.com/zisuu/picture/raw/master/img/20210204205345.png)

## 5.3用于快速恢复

![image-20210204205449318](https://gitee.com/zisuu/picture/raw/master/img/20210204205449.png)



# 6._version

![image-20210204205557693](https://gitee.com/zisuu/picture/raw/master/img/20210204205557.png)

![image-20210204205604912](https://gitee.com/zisuu/picture/raw/master/img/20210204205605.png)


































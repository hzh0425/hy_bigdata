## 1.启动流程做了什么

![image-20210201200448295](https://gitee.com/zisuu/picture/raw/master/img/20210201200448.png)

## 2.启动流程分析

### 1.启动脚本

当我们通过启动脚本bin/elasticsearch 启动ES 时，脚本通过exec 加载Java 程序。代
码如下：

![image-20210201200709343](https://gitee.com/zisuu/picture/raw/master/img/20210201200709.png)

### 2.解析命令行参数和配置文件

目前支持的命令行参数有下面几种，默认启动时都不使用，如下表所示。

![image-20210201200902663](https://gitee.com/zisuu/picture/raw/master/img/20210201200902.png)



### 3.加载安全配置

![image-20210201201007184](https://gitee.com/zisuu/picture/raw/master/img/20210201201007.png)

### 4.检查内部环境

![image-20210201201045544](https://gitee.com/zisuu/picture/raw/master/img/20210201201045.png)



### 5.检查外部环境

![image-20210201201150543](https://gitee.com/zisuu/picture/raw/master/img/20210201201153.png)

![image-20210201201233192](https://gitee.com/zisuu/picture/raw/master/img/20210201201233.png)



![image-20210201201404599](https://gitee.com/zisuu/picture/raw/master/img/20210201201404.png)



![image-20210201201643421](https://gitee.com/zisuu/picture/raw/master/img/20210201201643.png)

![image-20210201201757567](https://gitee.com/zisuu/picture/raw/master/img/20210201201757.png)

![image-20210201201826080](https://gitee.com/zisuu/picture/raw/master/img/20210201201826.png)

### 6.启动内部模块

![image-20210201201933977](https://gitee.com/zisuu/picture/raw/master/img/20210201201934.png)

### 7.keepAlive线程

![image-20210201202017796](https://gitee.com/zisuu/picture/raw/master/img/20210201202017.png)

## 3.节点关闭流程

![image-20210201202243427](https://gitee.com/zisuu/picture/raw/master/img/20210201202243.png)

## 4.关闭流程分析

![image-20210201202458272](https://gitee.com/zisuu/picture/raw/master/img/20210201202458.png)



![image-20210201202513229](https://gitee.com/zisuu/picture/raw/master/img/20210201202513.png)



![image-20210201202522946](https://gitee.com/zisuu/picture/raw/master/img/20210201202523.png)

![image-20210201202602378](https://gitee.com/zisuu/picture/raw/master/img/20210201202602.png)




















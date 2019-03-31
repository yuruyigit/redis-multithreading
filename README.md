
# redis3.0实现多线程
implement multithreading in redis3.0

在 [@huangz1990](https://github.com/huangz1990)的Redis3.0中文注释版的基础上修改 https://github.com/huangz1990/redis-3.0-annotated

也可以在我的博客看更详细的实践笔记 [多线程redis的设计思路，与实践笔记](http://onceme.me/post/redis-multithreading-implement/)
## 新增的特性

* 参考了阿里云Redis多线程实现方式
* 新增多个reactor线程(即IO线程)
* 新增一个worker线程  
    考虑线程安全问题，后面可以引入之前在gedis项目的做法，尽量不引入锁的情况下实现线程安全
* 使用无锁队列在在reactor线程与worker线程之间传递数据    
* 已经测试过客户端命令，主从全量同步，部分同步，以及master节点主动发出的命令传播
* 进过压测大部分命令的性能是原版redis的两倍

## 几种多线程模型的对比
* 首先是阿里云Redis采用的简化方案，增加多个reactor线程(IO线程)和一个worker线程  
  
* 第二个方案是多个reactor线程,多个worker线程  

* 第三个方案就是多线程不区分IO线程和工作线程，从IO到命令执行都在同一个线程  (开了实验性的分支，只支持对GET命令的压测)
  
* 第四个方案是综合了第一第三个方案，多个reactor线程，一个worker线程，不过只有写入操作会分配个worker线程，读取命令由reactor线程直接执行
  
* 在开工实现第一个方案的时候还意外发现了唯品会实现的多线程版redis：vire


> 对各种线程模型的压测对比(后面的两种方案只是实验性质的拉了分支，并没有处理线程安全问题，所以只能对get命令压测)

机器环境

|cpu     |  内存   | 操作系统 |
| ------ | ------ | ------|
| Intel(R) Xeon(R) CPU E5-1650 0 @ 3.20GHz (6核12线程) | DDR3 32G|Ubuntu 16.04.4 LTS|

压测结果

|        | 原版redis3.0   | 多reactor单worker |多reactor（io线程直接执行命令）|多reactor多worker(*)|
| ------ | ------        | ------             | ------ | ------ |
| GET    | 110533/107540 | 304697/208500 |   427069.456|  350202.9|
| SET    | 104119.06   | 217941.196 | | | |
| HSET    | 101584.73   | 181488.2 | | | |
| HGET    | 98058.45   | 180018 | | | |
| ZADD    | 87382.03   | 132996.41 | | | |

```
//这里直接使用了vire的多线程压测客户端
//e.g get命令的压测  -T 表示压测程序开启的线程数
./tests/vire-benchmark -p 6381 -c 600 -T 12 -t get -n 1000000
```
GET命令的压测会有两个值，斜杆"/"前是直接对空数据库的GET请求，另一个是有数据情况下的GET请求  
&emsp;&emsp;从压测对比上看，目前实现的多reactor单worker线程模型(虽然只在空数据集的情况下进行压测)，在大部分命令中性能大概是原版redis是两倍，不过还是直接多个工作线程的方案三性能最好   


## TODO

* 内存满的时候将淘汰的key落地
* 压测，找出瓶颈，向C10M方向优化
* 加上磁盘搜索功能，让redis不再局限与一个内存数据库(待定)
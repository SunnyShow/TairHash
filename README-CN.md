# TairHash: field带有过期时间和版本的hash数据结构
## 简介  [英文说明](README.md)
     TairHash是基于redis module开发的一种hash数据结构，和redis原生的hash数据结构相比，TairHash不但和原生hash一样具有丰富的数据接口和高性能，还可以为field设置过期时间和版本，这极大的提高了hash数据结构的灵活性，在很多场景下可以大大的简化业务开发。TairHash提供active expire机制，即使field在过期后没有被访问也可以被主动删除，释放内存。


### 主要的特性如下：

- field支持单独设置expire和version
- 针对field支持高效的active expire和passivity expire
- 语法和原生hash数据类型类似
- 支持redis的swapdb、rename、move、copy等语义

### 高效过期实现原理：
![avatar](imgs/tairhash_index.png)

- 使用两级排序索引，第一级对tairhash主key进行排序，第二级针对每个tairhash内部的field进行排序
- 第一级排序使用第二级排序里最小的ttl进行排序，因此主key是按照ttl全局有序的
- 内置定时器会周期扫描第一级索引，找出一部分已经过期的key，然后分别再检查这些key的二级索引，进行field的淘汰，也就是active expire
- 每一次对tairhash的写操作，也会先检查第一级索引，并最多过期三个field，这些field不一定属于当前正在操作的key，因此理论上写的越快淘汰速度也就越快
- 每一次读写field，也会触发对这个field自身的过期淘汰操作
- 排序中所有的key和field都是指针引用，无内存拷贝，无内存膨胀问题


<br/>

同时，我们还开源了一个增强型的string结构，它可以给value设置版本号并支持memcached语义，具体可参见[这里](https://github.com/alibaba/TairString)。

## 快速开始

```go
127.0.0.1:6379> EXHSET k f v ex 10
(integer) 1
127.0.0.1:6379> EXHGET k f
"v"
127.0.0.1:6379> EXISTS k
(integer) 1
127.0.0.1:6379> debug sleep 10
OK
(10.00s)
127.0.0.1:6379> EXISTS k
(integer) 0
127.0.0.1:6379> EXHGET k f
(nil)
127.0.0.1:6379> EXHSET k f v px 10000
(integer) 1
127.0.0.1:6379> EXHGET k f
"v"
127.0.0.1:6379> EXISTS k
(integer) 1
127.0.0.1:6379> debug sleep 10
OK
(10.00s)
127.0.0.1:6379> EXISTS k
(integer) 0
127.0.0.1:6379> EXHGET k f
(nil)
127.0.0.1:6379> EXHSET k f v  VER 1
(integer) 1
127.0.0.1:6379> EXHSET k f v  VER 1
(integer) 0
127.0.0.1:6379> EXHSET k f v  VER 1
(error) ERR update version is stale
127.0.0.1:6379> EXHSET k f v  ABS 1
(integer) 0
127.0.0.1:6379> EXHSET k f v  ABS 2
(integer) 0
127.0.0.1:6379> EXHVER k f
(integer) 2
```  
## 编译及使用

```
mkdir build  
cd build  
cmake ../ && make -j
```
编译成功后会在lib目录下产生tairhash_module.so库文件
## 测试方法

1. 修改`tests`目录下tairhash.tcl文件中的路径为`set testmodule [file your_path/tairhash_module.so]`
2. 将`tests`目录下tairhash.tcl文件路径加入到redis的test_helper.tcl的all_tests中
3. 在redis根目录下运行./runtest --single tairhash

## 客户端
### Java : https://github.com/aliyun/alibabacloud-tairjedis-sdk
### 其他语言：可以参考java的sendcommand自己封装

## API
[参考这里](CMDDOC-CN.md)

<br/>

## 适用redis版本  
由于TairHash的active expire实现依赖`unlink2`回调函数(见这个[PR](https://github.com/redis/redis/pull/8999)))同步释放索引结构，因此需要确保你的Redis中REDISMODULE_TYPE_METHOD_VERSION不低于4。
如果想要在低版本的redis(如redis-5.x)中使用TairhHash，则可以将CMakeLists.txt中的`add_definitions(-DENABLE_ACTIVE)`注释掉并重新编译，这样TairHash将禁止active expire功能，只支持passive expire，如果你会主动读取自己写入的field，那么passive expire也可以满足你的需求。
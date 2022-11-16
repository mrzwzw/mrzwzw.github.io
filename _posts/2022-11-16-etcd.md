---
layout: article
titile: "etcd介绍及其用go语言使用etcd"
---



@[TOC](etcd介绍及其用go语言使用etcd)

# 一、Etcd简介
etcd 是一个分布式键值对存储系统，内部采用 raft 协议作为一致性算法，用于可靠、快速地保存关键数据，并提供访问。通过分布式锁、leader选举和写屏障(write barriers)，来实现可靠的分布式协作。etcd集群是为高可用、持久化数据存储和检索而准备。

etcd 以一致和容错的方式存储元数据。分布式系统使用 etcd 作为一致性键值存储系统，用于配置管理、服务发现和协调分布式工作。使用 etcd 的通用分布式模式包括领导选举、分布式锁和监控机器活动。

目前，etcd 会监听两个端口，默认是 2379 端口和 2380 端口。其中，2380 端口用于集群内部通信，主要涉及集群间数据同步、心跳、选举等。2379 端口用于与客户端通信，比如接收客户端发起的读/写数据请求。


# 二、etcd概念词汇
| 词条      | 词条含义                                                     |
| --------- | ------------------------------------------------------------ |
| Raft      | etcd所采用的保证分布式系统强一致性的算法。                   |
| Node      | 一个Raft状态机实例。                                         |
| Member    | 一个etcd实例。它管理着一个Node，并且可以为客户端请求提供服务。 |
| Cluster   | 由多个Member构成、可以协同工作的etcd集群。                   |
| Peer      | 对同一个etcd集群中另外一个Member的称呼。                     |
| Client    | 向etcd集群发送HTTP请求的客户端。                             |
| WAL       | 预写式日志，etcd用于持久化存储的日志格式。                   |
| snapshot  | etcd防止WAL文件过多而设置的快照，存储etcd数据状态。          |
| Leader    | Raft算法中，通过竞选而产生的、处理所有数据提交的节点。       |
| Follower  | 竞选失败的节点作为Raft中的从属节点，为算法提供强一致性保证。 |
| Candidate | 当Follower超过一定时间接收不到Leader的心跳时转变为Candidate开始竞选。 |
| Term      | 某个节点成为Leader到下一次竞选时间，称为一个Term。           |
| Index     | 数据项编,aft中通过Term和Index来定位数据。                    |

# 三、工作原理
![在这里插入图片描述](https://img-blog.csdnimg.cn/576df002b5d041ebb5f85fd3daa712e5.png)
1. wal：进行log日志记录，记录之后会进行广播给集群中其他节点，然后返回给leader告诉leader是否同意修改，超过一半的同意修改就会把数据进行提交，把他刷到磁盘中去。主要是通过wal来实现集群中数据的强一致性。
2. snapshot：集群中加入一个节点我们需要一个快照数据，snapshot就是一个快照数据，然后发送给follower，为了保证follower和leader保持一致。
3. boltdb：相当于mysql中的存储引擎，etcd的事务是基于boltdb的事务实现的，etcd中每一个key 要创建一个索引，每一个索引要创建一个b+树，其原因是etcd要存储每一个key历史版本信息。
# 四、基本操作
下面的操作都是v3版本的操作，目前下载最新版本的都默认是v3版本，如果是v2版本可以根据第一个操作手动更改为v3版本

 1. 修改ETCDCTL的api版本：
```powershell
 export ETCDCTL_API=3
```

2. 查看etcdctl内部的命令：
```powershell
etcdctl --help
```

3. 添加/修改键值对
```powershell
etcdctl put key value
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/449e0c2a748e4b55833a45dedcc69809.png)
显示`ok`说明键值对添加成功

4. 查看键值对
```powershell
etcdctl get key
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/b081a94c166c489c827e15e65e5d6eda.png)

5. 修改键值对
```powershell
etcdctl put key value
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/ed1bdb55797d40aab5a7d60277e1befc.png)
修改成功，get操作也查看到value值改变了。

6. 删除键值对
```powershell
etcdctl del /key/test
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/cd630a4ed9b542a785d2f70593aaa958.png)
数字1是说明有一项被修改

7. 检测键值对
```powershell
etcdctl watch key
```

8. 查看版本
```powershell
etcdctl version
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/a934a8dfdfd744dbb7adaf323df29407.png)



# 五、使用go语言操作etcd

## 安装

```
go get go.etcd.io/etcd/clientv3
```

## put和get操作
```go
package main

import (
	"context"
	"fmt"
	"time"

	"go.etcd.io/etcd/clientv3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		// handle error!
		fmt.Printf("connect to etcd failed, err:%v\n", err)
		return
	}
    fmt.Println("connect to etcd success")
	defer cli.Close()
	// put
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	_, err = cli.Put(ctx, "q1mi", "dsb")
	cancel()
	if err != nil {
		fmt.Printf("put to etcd failed, err:%v\n", err)
		return
	}
	// get
	ctx, cancel = context.WithTimeout(context.Background(), time.Second)
	resp, err := cli.Get(ctx, "q1mi")
	cancel()
	if err != nil {
		fmt.Printf("get from etcd failed, err:%v\n", err)
		return
	}
	for _, ev := range resp.Kvs {
		fmt.Printf("%s:%s\n", ev.Key, ev.Value)
	}
}
```
默认会在2379端口监听客户端通信，在2380端口监听节点通信。所以上面的网络号为127.0.0.1:2379。
`注意`：put和get操作需要使用cancel，如果请求超过一秒钟(自己定义)。但是watch操作不需要。
## watch操作
```go
package main

import (
	"context"
	"fmt"
	"time"
	"go.etcd.io/etcd/clientv3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		fmt.Printf("connect to etcd failed, err:%v\n", err)
		return
	}
	fmt.Println("connect to etcd success")
	defer cli.Close()
	watch_ch := cli.Watch(context.Background(), "q1mi") 
	for wresp := range watch_ch {
		for _, ev := range wresp.Events {
			fmt.Printf("Type: %s Key:%s Value:%s\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
		}
	}
}
```
`注意`：其中watch_ch是一个channel，监测这个通道，要是这个通道有值返回，拿到wresp，从这个wresp.Events拿到这个事件是什么类型的。如果这个chan没有值的话他就阻塞着。
`思路`：程序一启动etcd里面的key是一个状态，以后要是通过etcd更改、创建、删除这些操作，这个时候etcd就会通知我们，可以通过它来实现程序的热加载，程序一启动的时候先从etcd里面把我程序的配置文件加载出来。再创建一个watch，用一个单独的gorouting一直watch tecd中的key，当更改配置文件，我们就可以通过这个gorouting就会收到通知。
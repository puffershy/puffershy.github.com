# Fabric 概念整理 #
## Fabric架构 ##
1. 在Fabric的区块链网络中，有四类节点：MSP，Ordering Node，Endorsing Peer，Commtting Peer。
- MSP
MSP(Membership Service Provider), 这类节点主管区块链网络中其他的节点的授权，准入，踢除。通过给不同节点颁发证书的方式，授予不同类型的节点相应的权限。

- Ordering Node
中文可以称作排序节点。通常在一个网络中至少有一个或多个排序节点，这类节点负责 按照指定的算法，将交易进行排序，并返回给Committing Peer。其并不关心具体的交易细节。

- Endorsing Peer
这类节点的主要负责接收交易请求，验证这笔交易之后，并做一些预处理之后，并将签名后的数据传回给客户端。

- Committing Peer
这类节点做是区块链网络中的全节点，它们需要记录完整的区块信息，并且验证每笔交易的正确性，是最终将交易打包进区块链的节点。

> 推荐阅读
> https://www.jianshu.com/p/eb33c7288ce7?from=timeline

Fabric交易的整个生命周期，分成7个阶段
![](https://upload-images.jianshu.io/upload_images/3959874-02f11ae0159f264d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1005/format/webp)

- 第一阶段
在交易的第一阶段，客户端应用发起智能合约A的一个交易请求给背书节点E0（）。智能合约A配置的背书策略要求只需要E0,E1和E2签名。其他节点不包含在策略的要求中，因此可以不签名。注意图中的红色和蓝色小矩形块分别代表不同的子链或者叫通道，这个通道是1.0架构引入的新概念，用来实现各条业务链的数据隔离，保护交易的隐私性，避免像比特币那样的交易对所有人都公开。
![](https://upload-images.jianshu.io/upload_images/3959874-b769e744c2c92712.png?imageMogr2/auto-orient/strip|imageView2/2/w/742/format/webp)
如图所示，这里说明一下，图中的C代表共识网络（Consensus Network）,黄色的圆角矩形表示在peer上部署的智能合约。如E0，E1，E2上部署了A，B两个智能合约，E3上部署了A，D两个智能合约，而E4，E5上部署了Y，Z两个智能合约。
备注：智能合约有些地方称为：“链码”；背书节点认为是peer。


- 第二阶段
背书节点E0使用 MSP（MSP=Member Service Provider）验证签名，判断是否客户端应用被正确授权可以执行发起交易请求。背书节点获取交易请求中的参数（链代码）作为输入参数，然后E0会与交易对应ChainCode所属的docker实例通信，并为其提供模拟执行的State Database的读写集，也就是说ChainCode会执行完智能合约中的业务逻辑，但是并不会在stub.PutState的时候写数据库，ChainCode所属的docker实例执行完ChainCode后产生交易执行结果，然后将执行结构返回给E0。这个执行结果包括下列数据：响应值，读集合和写集合。不过，这个时候，并不会更新账本。这些值的集合，连同背书节点的签名和一个YES/NO 的陈述一起放到 proposal response 中返回给客户端应用（图中的Client App）。


- 第三阶段
客户端应用验证背书节点签名，然后继续发送背书请求给E1和E2，过程跟与E0的交互时一样。
![](https://upload-images.jianshu.io/upload_images/3959874-c3aa3fc031d1bd89.png?imageMogr2/auto-orient/strip|imageView2/2/w/733/format/webp)

- 第四阶段
背书节点E1和E2发送背书处理结果给客户端应用。客户端应用收集完所有背书节点的签名后，检查是否指定的背书策略已经满足。根据 Fabric 的架构设计，即使应用选择不检查交易的背书反馈，或者继续发送一个没有经过背书处理的交易，在commit交易的验证阶段，这个背书策略仍然会被peer 强制执行。
![](https://upload-images.jianshu.io/upload_images/3959874-44f31f8105bd9c68.png?imageMogr2/auto-orient/strip|imageView2/2/w/697/format/webp)

- 第五阶段
客户端应用将交易和响应信息封装到一个事务消息（transaction message）中，然后广播到共识网络（这里的共识网络又称为排序服务Ordering Service，后续统一称为共识网络）。交易中包含读写集，背书节点签名和通道 ID。
共识网络节点不会关注交易细节和交易消息的具体内容，只是简单地从网络中接收来自所有通道的交易，然后按通道按时间顺序排序，处理的结果是一个Batch的交易，也就是一个区块，这个区块的产生有两种情况，一种情况是区块中的交易很多，区块的大小达到了配置文件中配置的大小，而另一种情况是区块中的交易很少，没有达到配置的大小，那么共识网络节点就会等，等到大小足够大或者超时时间。这些设置是在configtx.yaml中配置的。

开发人员可以自定义出块的时间和每个区块内交易的数量。详细见《Fabric配置文件详解》

- 第六阶段
共识服务节点将打包的区块广播道同一个通道的所有peer，通过Fabric提供的deliver RPC服务，共识服务节点和peer节点之间的通信细节，我们在稍后详细介绍。必须说明的是，E4和E5不在同样的通道上（或者说不属于同样的子账本），因此E4,E5不会收到任何更新消息。另外，共识网络节点只是涉及到排序交易和打包区块，不会执行智能合约。
![](https://upload-images.jianshu.io/upload_images/3959874-4cc16d54d48243b0.png?imageMogr2/auto-orient/strip|imageView2/2/w/687/format/webp)

- 第七阶段
Peers收到共识网络发来的区块后，会先进行以下校验：

n 再次验证区块中的交易以确保背书策略满足。

n 检查区块的数据是否正确。

n 对每个交易进行验证，确保自从读集合数据在交易执行生成后，读集合变量对应的账本的状态没有变化，也就是验证交易中的读写数据集是否与State Database的数据版本一致。

验证通过后，区块中的交易打上合法和非法交易的标签，然后添加区块到通道对应的链上，同时把所有验证通过的交易的读写集中的写的部分写入状态数据库State Database。对于每个合法交易，写集合被提交到当前的状态数据库。同时，一个区块事件产生并发出，通知客户应用，交易已经不可更改的添加到了链上，也是告诉应用客户端，交易是合法还是非法。

另外对于区块链，本身是文件系统，不是数据库，所有也会有把区块中的数据在LevelDB中建立索引。
![](https://upload-images.jianshu.io/upload_images/3959874-78d02d0714683065.png?imageMogr2/auto-orient/strip|imageView2/2/w/678/format/webp)




2. 每个组织都拥有自己的一套MSP(Membership Service Provider，联盟链成员的证书管理).
![](http://www.taohui.pub/wp-content/uploads/2018/05/MSP%E4%B8%8E%E7%BB%84%E7%BB%87%E9%97%B4%E5%85%B3%E7%B3%BB.png)
ORG可以管理自己的MSP

3. 链码升级的时候会调用init初始化方法
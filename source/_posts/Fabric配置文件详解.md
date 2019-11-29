# Fabric配置文件详解 #

## 目录 ##
配置文件|描述
---|---
configtx.yaml|配置交易相关参数，包含出块时间、一个区块最大交易数
crypto-config.yaml|证书相关配置
docker-compose-couch.yaml|couchDB配置

## configtx.yaml ##
自定义出块时间和每个区块内交易的数量。
主要配置如下：
```
    # Batch Timeout: The amount of time to wait before creating a batch
    BatchTimeout: 2s


    # Max Message Count: The maximum number of messages to permit in a batch
    MaxMessageCount: 10
```

这里主要的配置项是BatchTimeout和MaxMessageCount。

BatchTimeout是配置多久产生一个区块，默认是2秒，通常在项目实践中，我们发现交易量并不大，如果配置的时间过小就会产生很多空的区块，配置时间太长，则发现等待产生区块的时间太长。具体时间由交易频率和业务量决定。我们实际项目中，通常配置在30秒。

MaxMessageCount是配置在一个区块中允许的交易数的最大值。默认值是10。交易数设置过小，导致区块过多，增加orderer的负担，因为要orderer要不断的打包区块，然后deliver给通道内的所有peer,这样还容易增加网络负载，引起网络拥堵。我们实际项目中通常配置500，不过具体还应该看业务情况，因为如果每个交易包含的数据的size如果太大，那么500个交易可能导致一个区块太大，因此需要根据实际业务需求权衡。

读者可能有一个疑问，这里有2个参数可以配置区块的出块策略，那么究竟那个因素优先发生作用呢？实际上根据Fabric设计的出块策略，BatchTimeout和MaxMessageCount的任何一个参数条件满足，都会触发产生新的区块。举个例子，假设我们配置了出块时间BatchTimeout为30秒，块内交易最大数量MaxMessageCount为500。第一种情况，当出块时间为20秒(时间上还没达到出块要求)，但是交易数已经累积到500个了，这个时候也会触发新的区块产生。第二种情况，交易数才1个，但是出块时间已经30秒了，这个时间也会触发新的区块产生，尽管这个新的区块里只有一个交易。

Fabric的这种出块策略设计相比还是比较灵活的，可配置的。相比而言，在比特币中，大家都知道出块机制是固定的，就是每隔10分钟（600秒）产生一个区块，就一个陌生，不可更改。而以太坊类似，也是基于事件的出块策略，只是时间更短，每15秒产生一个区块。因此，Fabric的出块策略在设计上还是比较进步的。

# Fabric 私有数据 #

Hyperledger Fabric private data是1.2版本引入的新特性，fabric private data是利用旁支数据库（SideDB）来保存若干个通道成员之间的私有数据，从而在通道之上又提供了一层更灵活的数据保护机制。

## 1.私密数据的定义 ##

如果某个渠道上的一组组织需要将数据与该渠道上的其他组织保密，他们可以选择创建一个仅包含需要访问数据的组织的新渠道。但是，在每种情况下创建单独的通道会产生额外的管理开销（维护链代码版本，策略，MSP等）。

超级账本 Fabric 引入了 sideDB 机制，通过 Hash 处理和私有数据结构，在通道内部实现了更细粒度的隐私保护。Hash 后的交易内容仍然会发送到排序节点，并提交到公共的数据库和账本结构中。而授权组的 Peer 节点则在本地维护私有的状态数据库和区块链结构，保存交易的明文内容。这就保证了通道内其他节点无法看到授权组的交易内容。

私密数据分为两部分：

一个是真正的key，value，它被存在 peer的私密数据库（private state）中。
另一部分为公共数据，它是真实的私密数据key，value 哈希后的值 hash(key),hash(value)，它被存在普通的peer数据库中（state），orderer端可以拿到该值。没有被分配私密数据权限的peer，也仅仅可以存储hash后的key和value。

![](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/PrivateDataConcept-2.png)

## 2.私有数据在fabric中的交易流程 ##
SDK将交易发送给背书节点，该背书节点需要通过policy(此处的policy为instantiation设定的)验证。
背书节点模拟执行交易，并将真实的私密数据 key和value存储于瞬时数据库(private transient DB)中。基于collection policy，验证通过的peer节点，通过gossip同步真实的私密数据，并将其存储于瞬时数据库中。背书节点将模拟执行后的结果返回给SDK，返回的数据中仅有公共数据(hash过的私密数据)。
SDK将peer返回的结果打包后发送给orderer，和普通的区块一样，orderer切块后，将其分发给peer，此时所有的peer都拿到了公共数据，所有的peer都可以去验证私有数据，没有通过collection policy的peer，也可以验证，而且还不拥有真实的私密数据，保证了数据的私密性。
在commit之前，peer首先去判断自己是否通过collection policy的检查，若通过，将去检查自己的瞬时数据库中是否有真实的私密数据，如果没有，将尝试从别的peer处拉取数据。
取到私密数据后，首先去和公共数据的hash去做比对，若一致，此时进行commit操作，将私密数据的HASH写入到公共数据库中。提交该交易和这个区块到账本中，成功提交后，私密数据将会从瞬时数据库拷贝到私密数据库中，并从瞬时数据库中删除。此时整个交易流程结束，数据成功写入到账本内。

## 3.私有数据使用案例 ##

- 构建集合定义
将通道上的数据私有化的第一步是构建一个集合定义，用于定义对私有数据的访问。最新可以从github上下载“[fabric-sdk-java](https://github.com/hyperledger/fabric-sdk-java "fabric-sdk-java")”，参考PrivateDataIT.yaml
配置解释：

配置项|描述
---|---
name|集合名称
blockToLive|此值表示数据应以块为单位存储在私有数据库中的时间。数据将在专用数据库上为指定数量的块生效，之后将被清除，从而使这些数据从网络中过时。要无限期地保留私有数据，即永远不要清除私有数据，请将blockToLive属性设置为0。
maximumPeerCount|此集合可以传播到的最大对等体数
requiredPeerCount|传播私有数据所需的对等数量，作为认可链代码的条件
SignaturePolicyEnvelope.identities|定义允许持久保存集合数据的组织peer节点
SignaturePolicyEnvelope.policy|指向identities定义的节点，即私有数据存放节点

样例：
```
#
# Copyright IBM Corp. All Rights Reserved.
#
# SPDX-License-Identifier: Apache-2.0
#
---

  - StaticCollectionConfig: # protobuf oneof identifies type. At this time the only option.
       name: "collectionMarbles" # how to reference this collection in chaincode api.
       blockToLive: 10 # number blocks that have the data kept in the database after which older database data is pruned.
       maximumPeerCount: 0 # the maximum number of peers this collection can be disseminated to.
       requiredPeerCount: 0 # the minimum number of peers this collection must be disseminated to.
       SignaturePolicyEnvelope: # protobuf oneOf identifies type. At this time the only option.
         identities:
             - user1: {"role": {"name": "member", "mspId": "Org1MSP"}}  #name can be: member, admin, client, or peer
             - user2: {"role": {"name": "member", "mspId": "Org2MSP"}}
         policy:
             1-of: # must be signed by at least one of these (OR) can be 2-of for both (AND).
               - signed-by: "user1" #reference to user1 in identities section must sign
               - signed-by: "user2" #reference to user2 in identities section must sign
  - StaticCollectionConfig:
       name: "collectionMarblePrivateDetails"
       blockToLive: 10
       maximumPeerCount: 0
       requiredPeerCount: 0
       SignaturePolicyEnvelope:
         identities:
             - user1: {"role": {"name": "member", "mspId": "Org1MSP"}}
             - user2: {"role": {"name": "member", "mspId": "Org2MSP"}}
         policy:
             1-of:
               - signed-by: "user1"

```
样例表示
1. 集合“collectionMarbles”保存组织“Org1MSP”“Org2MSP”的所有peer。
2. 集合“collectionMarblePrivateDetails”保存组织“Org1MSP”的所有peer。

![](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/SideDB-org2.png)
![](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/SideDB-org2.png)


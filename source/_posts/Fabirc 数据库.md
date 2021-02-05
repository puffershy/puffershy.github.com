# Fabric #
## 扩展阅读 ##
标题|备注|网址
---|---|---
Hyperledger Fabric启用CouchDB为状态数据库|包含couch配置，以及链码编写|https://blog.51cto.com/clovemfong/2154997?source=dra
Fabric学习笔记(三) - Fabric v1.0.5 使用CouchDB|提及持久化|https://segmentfault.com/a/1190000012889682
Fabric 1.0 alpha快速部署和CouchDB使用|包含fabric部署，故障排除，以及CouchDB,|https://zhuanlan.zhihu.com/p/25849348
Hyperledger Fabric &CouchDB 查询|mongoDB查询|https://blog.csdn.net/cyberspecter/article/details/85014568
Hyperledger Fabric常用命令|常用命令|https://blog.csdn.net/yanhuibin315/article/details/81560141
使用CouchDB作为状态数据库|使用chaincode的CouchDB查询|https://www.cnblogs.com/aberic/p/8384999.html
Hyperledger中文文档|中文使用手册|https://hyperledgercn.github.io/hyperledgerDocs/
fabric 官方文档|英文使用手册|https://hyperledger-fabric.readthedocs.io/en/latest/build_network.html
10分钟弄懂当前各主流区块链架构|-|https://blog.csdn.net/weixin_42758350/article/details/81153647
Fabric 架构和概念|-|https://www.jianshu.com/p/722736b52c34
IMB fabric 社区|IBM官网社区,包含视频|https://www.ibm.com/developerworks/community/wikis/home?lang=en#!/wiki/W30b0c771924e_49d2_b3b7_88a2a2bc2e43/page/%E8%AF%A6%E8%A7%A3Hyperledger%20Fabric%20v1.4%20LTS
HyperLedger Fabric社区|官方社区|https://www.hyperledger.org/community
HyperLedger Fabric 资料网址大全|知乎非常全的网址导航|https://zhuanlan.zhihu.com/p/26333761
hyperledger github|github|https://github.com/hyperledger
fabric-ca-server|ca mysql|https://stackoverflow.com/questions/57402608/fabric-ca-server-connect-to-azure-mysql-this-authentication-plugin-is-not-suppo
在HyperLedger Fabric中启用CouchDB作为State Database|counchdb连接查询|https://www.cnblogs.com/studyzy/p/7101136.html
fatric 1.0学习记录|作者学习fatric1.0的过程|https://www.cnblogs.com/aberic/category/1079974.html
Fabric Java SDK最新教程|精选教程|https://my.oschina.net/u/2472105/blog/3042504
链码API文档|ChaincodeStub|http://cw.hubwiz.com/card/c/fabric-chaincode-node/1/2/29/

fabric 官网地图

描述|网址
---|---
在线聊天|https://chat.hyperledger.org/channel/general
wiki文档|https://wiki.hyperledger.org/


## 搭建Fatric ##

yum install python-pip


1.用uname -r命令检查内核版本
2.安装docker。
>sudo wget -qO- https://get.docker.com
>install docker-ce -y

3.安装docker-compose
>sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
>sudo chmod +x /usr/local/bin/docker-compose

pip install docker-compose


# 3.2 保存并生效
source /etc/profile

## LevelDB ##

https://www.jianshu.com/p/dd4b4582f5ad
https://blog.csdn.net/itcastcpp/article/details/80384373

## CouchDB ##

peer0.org1.example.com: 
  container_name: peer0.org1.example.com 
  environment: 
    - CORE_LEDGER_STATE_STATEDATABASE=CouchDB 
    - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=192.168.12.231:5984 
    - CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME=admin 
    - CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD=password 
  extends: 
    file:  base/docker-compose-base.yaml 
    service: peer0.org1.example.com


这里的192.168.12.231:5984是我映射CouchDB后的Linux的IP地址和IP。然后是设置用户名和密码。把4个Peer的配置都改好后，保存，我们试着启用Fabric：
./network_setup.sh up
等Fabric启动完成并运行了ChainCode测试后，我们刷新http://192.168.12.231:5984/_utils
，可以看到以Channel名字创建的Database，另外还有几个是系统数据库。

### 连接查询 ###
>  curl http://IP地址:7984/mychannel_puffer/_all_docs
>  {"total_rows":2,"offset":0,"rows":[
{"id":"a","key":"a","value":{"rev":"1-de8d5fec120cc887a129b25896e27fe8"}},
{"id":"b","key":"b","value":{"rev":"1-903662ec9ffc2687a9a07bd0b2c37d99"}}
]}



>   curl http://IP地址:7984/mychannel_puffer/a
>   {"_id":"a","_rev":"1-de8d5fec120cc887a129b25896e27fe8","~version":"\u0000CgMBBQA=","_attachments":{"valueBytes":{"content_type":"application/octet-stream","revpos":1,"digest":"md5-XjotKUY1VBQxAUMQDoSRIA==","length":4,"stub":true}}}



3. 启动网络
docker-compose -f docker-compose-cli.yaml up -d

docker-compose -f docker-compose-cli.yaml down
# couchdb 启动命令
docker-compose -f docker-compose-cli.yaml -f docker-compose-couch.yaml up


# fabric1.4.0——用fabric-samples工程体验fabric部署安装的过程 #


1. 下载fabric-samples
> git clone  -b release-1.4 https://github.com/hyperledger/fabric-samples.git

git clone -b branchA http://admin@192.168.1.101:7070/r/virtualbox_all_versions.git

2. 进入basic-network目录，利用docker-compose启动容器
>[root@master opt]# cd fabric-samples/basic-network/
>[root@master basic-network]# docker-compose -f docker-compose.yml up -d

3. 切换环境到管理员用户的MSP，进入peer节点容器peer0.org1.example.com
>[root@master basic-network]# docker exec -it -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.example.com/msp" peer0.org1.example.com bash

4. 创建通道
>`peer channel create -o orderer.example.com:7050 -c mychannel -f /etc/hyperledger/configtx/channel.tx`
>root@8bce6cf7abcd:/opt/gopath/src/github.com/hyperledger/fabric# peer channel create -o orderer.example.com:7050 -c mychannel -f /etc/hyperledger/configtx/channel.tx
>2019-11-19 03:42:23.463 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-11-19 03:42:23.496 UTC [cli.common] readBlock -> INFO 002 Received block: 0

5. 加入通道
>`peer channel join -b mychannel.block`
>root@69d1ad88e343:/opt/gopath/src/github.com/hyperledger/fabric# peer channel join -b mychannel.block
2019-11-19 03:47:08.236 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized

6. 退出peer节点容器peer0.org1.example.com
> exit

6. 进入cli容器安装链码和实例化
>`docker exec -it cli /bin/bash`
>[root@master basic-network]# docker exec -it cli /bin/bash

7. 给peer节点peer0.org1.example.com安装链码
>`peer chaincode install -n mycc -v v0 -p github.com/chaincode_example02/go`
>root@27955f99dcc5:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode install -n mycc -v v0 -p github.com/chaincode_example02/go
2019-11-20 02:58:57.445 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2019-11-20 02:58:57.445 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2019-11-20 02:58:57.731 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK"  

8. 实例化链
>`peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc -v v0 -c '{"Args":["init","a","100","b","200"]}'`

9.  链码调用和查询
链码实例化之后就可以查询初始值了，同样是在cli容器中进行
>`peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'`
>root@27955f99dcc5:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
100
调用链码，从“a”转移10到“b”
>`peer chaincode invoke -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'`
>root@27955f99dcc5:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode invoke -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'
2019-11-20 03:19:02.438 UTC [chaincodeCmd] InitCmdFactory -> INFO 001 Retrieved channel (mychannel) orderer endpoint: orderer.example.com:7050
2019-11-20 03:19:02.450 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 002 Chaincode invoke successful. result: status:200 
root@27955f99dcc5:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
90
root@27955f99dcc5:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode query -C mychannel -n mycc -c '{"Args":["query","b"]}'
210


## 证书路径 ##
>user1-key = 
cd /home/yytmp/mj/hyperledger-fabric/fabric-samples/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore
user1-cert.pem =
cd /home/yytmp/mj/hyperledger-fabric/fabric-samples/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts
user2-key =
cd /home/yytmp/mj/hyperledger-fabric/fabric-samples/fabric-samples/first-network/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/keystore
user2-cert.pem =
cd /home/yytmp/mj/hyperledger-fabric/fabric-samples/fabric-samples/first-network/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/signcerts



E:\workspace\workspace-jaws\jaws-blockchain\jaws-blockchain-server\target\classes\cert\user1-key
E:\workspace\workspace-jaws\jaws-blockchain\jaws-blockchain-server\target\classes\cert\user1-cert.pem



# 链码 peer 命令 #

参数说明： https://blog.csdn.net/qq_37133717/article/details/80932884

INSERT
peer chaincode invoke -C mychannel -n borrower_info_cc2 -c '{"Args":["{\"invokeType\":\"INSERT\",\"key\":\"zsf03\",\"value\":\"{\\\"companyAddress\\\":\\\"1shenzhen nanshan\\\",\\\"educationExperience\\\":\\\"UNDERGRADUATE_COURSE\\\",\\\"employeeType\\\":\\\"SALARIED_PERSON1\\\",\\\"familyAddress\\\":\\\"1shenzhen nanshan xunmei\\\",\\\"gender\\\":\\\"FMALE\\\",\\\"headShip\\\":\\\"yaungong1\\\",\\\"maritalStatus\\\":\\\"MARRIED1\\\",\\\"registeredCapital\\\":\\\"200001\\\",\\\"userName\\\":\\\"yY1\\\",\\\"workCity\\\":\\\"shenzhen1\\\",\\\"workTime\\\":\\\"3years12\\\"}\"}"]}'
RICH_QUERY
peer chaincode invoke -C mychannel -n borrower_info_cc2 -c '{"Args":["{\"invokeType\":\"RICH_QUERY\",\"key\":\"{\\\"selector\\\":{\\\"companyAddress\\\":{\\\"$regex\\\":\\\"shenzhen*\\\"}}}\"}"]}'
HISTORY
peer chaincode invoke -C mychannel -n borrower_info_cc2 -c '{"Args":["{\"invokeType\":\"HISTORY\",\"key\":\"zsf03\"}"]}' 
UPDATE
peer chaincode invoke -C mychannel -n borrower_info_cc2 -c '{"Args":["{\"invokeType\":\"UPDATE\",\"key\":\"zsf03\",\"value\":\"{\\\"companyAddress\\\":\\\"123shenzhen nanshan\\\",\\\"educationExperience\\\":\\\"UNDERGRADUATE_COURSE\\\",\\\"employeeType\\\":\\\"SALARIED_PERSON1\\\",\\\"familyAddress\\\":\\\"1shenzhen nanshan xunmei\\\",\\\"gender\\\":\\\"FMALE\\\",\\\"headShip\\\":\\\"yaungong1\\\",\\\"maritalStatus\\\":\\\"MARRIED1\\\",\\\"registeredCapital\\\":\\\"200001\\\",\\\"userName\\\":\\\"yY1\\\",\\\"workCity\\\":\\\"shenzhen1\\\",\\\"workTime\\\":\\\"3years1\\\"}\"}"]}'
QUERY
peer chaincode query -C mychannel -n borrower_info_cc2 -c '{"Args":["{\"invokeType\":\"DEL\",\"key\":\"zsf03\"}"]}'
DEL
peer chaincode invoke -C mychannel -n borrower_info_cc2 -c '{"Args":["{\"invokeType\":\"DEL\",\"key\":\"zsf03\"}"]}'

# 链码 #
 https://www.cnblogs.com/studyzy/p/7360733.html
链码开发编译测试|https://www.cnblogs.com/informatics/p/8051981.html

# TLS #
https://developer.ibm.com/tutorials/hyperledger-fabric-java-sdk-for-tls-enabled-fabric-network/


# 背书策略 #
https://www.jianshu.com/p/33a23471bdcc
https://blog.csdn.net/yeasy/article/details/88536882

默认情况下，会以当前 MSP 的管理员身份作为默认的策略，即只有当前 MSP 管理员可以进行链码实例化操作。这可以避免链码被通道中其他组织成员私自在其它通道内进行实例化。

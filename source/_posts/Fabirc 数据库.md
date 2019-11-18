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
>  curl http://172.30.0.129:7984/mychannel_liukecc/_all_docs
>  {"total_rows":2,"offset":0,"rows":[
{"id":"a","key":"a","value":{"rev":"1-de8d5fec120cc887a129b25896e27fe8"}},
{"id":"b","key":"b","value":{"rev":"1-903662ec9ffc2687a9a07bd0b2c37d99"}}
]}



>   curl http://172.30.0.129:7984/mychannel_liukecc/a
>   {"_id":"a","_rev":"1-de8d5fec120cc887a129b25896e27fe8","~version":"\u0000CgMBBQA=","_attachments":{"valueBytes":{"content_type":"application/octet-stream","revpos":1,"digest":"md5-XjotKUY1VBQxAUMQDoSRIA==","length":4,"stub":true}}}



3. 启动网络
docker-compose -f docker-compose-cli.yaml up -d

docker-compose -f docker-compose-cli.yaml down
# couchdb 启动命令
docker-compose -f docker-compose-cli.yaml -f docker-compose-couch.yaml up

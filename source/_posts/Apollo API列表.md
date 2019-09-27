
# Apollo API列表 #

http://ip:port/openapi/v1/

序号|接口名称|接口路径
--- |--- |---
1|获取集群下所有命名空间|envs/DEVELOP/apps/yyfax-zuul/clusters/default/namespaces
2|获取指定的命名空间|envs/DEVELOP/apps/yyfax-zuul/clusters/default/namespaces/application
3|创建命名空间|apps/yyfax-zuul/appnamespaces
4|锁定命名空间|envs/DEVELOP/apps/yyfax-zuul/clusters/default/namespaces/application/lock
5|创建key|envs/%s/apps/%s/clusters/%s/namespaces/%s/items
6|更新key|envs/%s/apps/%s/clusters/%s/namespaces/%s/items/%s
7|创建或者更新key|envs/%s/apps/%s/clusters/%s/namespaces/%s/items/%s?createIfNotExists=true
8|删除key|envs/%s/apps/%s/clusters/%s/namespaces/%s/items/%s?operator=%s
9|发布命名空间|envs/%s/apps/%s/clusters/%s/namespaces/%s/releases
10|获取最近生效的版本|envs/%s/apps/%s/clusters/%s/namespaces/%s/releases/latest



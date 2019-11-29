# Fabric CounchDB查询使用说明 #

推荐阅读：

描述|地址
---|---
CounchDB|http://docs.couchdb.org/en/stable/api/database/find.html?highlight=find
couchdb 丰富查询 selector 语法|https://blog.csdn.net/weixin_34037173/article/details/91809461
CouchDb SDK使用|https://www.cnblogs.com/studyzy/p/7360733.html

## CouchDB管理平台 ##
地址：http://172.30.0.129:5984/_utils
账号/密码:开发环境关闭登录校验

## URL查询 ##

1.查询数据库全部信息
URL格式：http://[IP地址]:[端口]/[数据库]/_all_docs

>`curl http://172.30.0.129:5984/mychannel_borrower_info_cc12/_all_docs`
>{"total_rows":2,"offset":0,"rows":[
{"id":"123456","key":"123456","value":{"rev":"1-692c9e9874947ce7071c613921d196ec"}},
{"id":"49eeea953b02e4cdeef3da9c61004889","key":"49eeea953b02e4cdeef3da9c61004889","value":{"rev":"1-692c9e9874947ce7071c613921d196ec"}}
]}
>![](https://i.imgur.com/ZJWQzRk.png)

2.指定key查询

URL格式：http://[IP地址]:[端口]/[数据库]/[key]
>`curl http://172.30.0.129:5984/mychannel_borrower_info_cc12/123456`
>{
    "_id": "123456",
    "_rev": "1-692c9e9874947ce7071c613921d196ec",
    "companyAddress": "shenzhen nanshan",
    "educationExperience": "UNDERGRADUATE_COURSE",
    "employeeType": "SALARIED_PERSON",
    "familyAddress": "shenzhen nanshan xunmei",
    "gender": "MALE",
    "headShip": "yaungong",
    "maritalStatus": "MARRIED",
    "registeredCapital": "20000",
    "userName": "yY",
    "workCity": "shenzhen",
    "workTime": "3years",
    "~version": "\u0000CgMBBgA="
}
![](https://i.imgur.com/mP5LV9c.jpg)

3.修改值

>`curl -X PUT http://172.30.0.129:5984/mychannel_borrower_info_cc12/123456 -d '{"_id":"123456","_rev":"1-692c9e9874947ce7071c613921d196ec","companyAddress":"shenzhen nanshan","educationExperience":"UNDERGRADUATE_COURSE","employeeType":"SALARIED_PERSON","familyAddress":"shenzhen nanshan xunmei","gender":"MALE","headShip":"yaungong","maritalStatus":"MARRIED","registeredCapital":"20000","userName":"布衣","workCity":"shenzhen","workTime":"3years","~version":"\u0000CgMBBgA="}'`
>{"ok":true,"id":"123456","rev":"2-8fd34cfea31a06bc61767153c080b597"}
>
>使用命令查询链码：
>`peer chaincode query -C mychannel -n borrower_info_cc12 -c '{"Args":["invoke","{\"invokeType\":\"QUERY\",\"key\":\"123456\"}"]}'`
>
>root@e615240f0340:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode query -C mychannel -n borrower_info_cc12 -c '{"Args":["invoke","{\"invokeType\":\"QUERY\",\"key\":\"123456\"}"]}'
{"companyAddress":"shenzhen nanshan","educationExperience":"UNDERGRADUATE_COURSE","employeeType":"SALARIED_PERSON","familyAddress":"shenzhen nanshan xunmei","gender":"MALE","headShip":"yaungong","maritalStatus":"MARRIED","registeredCapital":"20000","userName":"布衣","workCity":"shenzhen","workTime":"3years"}


**注意**：通过直接修改CounchDB数据库的更改都是有效的，Fatric并不知道我们修改了CounchDB的内容。所以，如果存在多个peer，通过URL直接修改counchDB的值，其他几个peer不会同步，<span style="color:red">后期跟踪。</span>

## JSON查询 ##

### json字段说明 ###
字段名|类型|是否必填|作用
---|---|---
selector|json| 必填|描述用于选择文档的条件的JSON对象。
limit |number| 可选|返回的最大结果数。 默认值为25。
skip |number | 可选| 跳过第一个“ n”个结果，其中“ n”是指定的值
sort |json| 可选| JSON数组遵循排序语法
fields |array| 可选|JSON数组，指定应返回每个对象的哪些字段。 如果省略，则返回整个对象。 有关过滤字段的部分中提供了更多信息
use_index |string/array| 可选|指示查询使用特定索引。 指定为“ <design_document>”或“ [<design_document>”，“ <index_name>”]
r|number| 可选|阅读仲裁所需的结果。 默认值为1，在这种情况下，将返回在索引中找到的文档。 如果将其设置为较高的值，则在返回结果之前，至少要从许多副本中读取每个文档。 与仅使用带有索引的本地存储文档相比，这可能会花费更多时间。 默认值：1
bookmark |string| 可选|一个字符串，使您可以指定所需的结果页面。 用于对结果集进行分页。 每个查询都会在书签键下返回一个不透明的字符串，然后可以将其传递回查询中以获取下一页结果。 如果选择器查询的任何部分在请求之间更改，则结果是不确定的。默认值：null
update |boolean| 可选|在返回结果之前是否更新索引。 默认为true
stable |boolean| 可选|视图结果是否应该从一组“稳定”的分片中返回
stale |string| 可选|update = false和stable = true选项的组合。 可能的选项：“ ok”，false（默认）
execution_stats |boolean| 可选|在查询响应中包括执行统计信息。 可选，默认：``false''


### 基本运算符说明 ###

操作符|说明|示例
---|---|---
$gt |大于|-
$lt|小于 |-
$eq|等于|-
$ne|不等于|-
$lte|小于或等于|-
$gte|大于或等于|-
$in|包含|-
$nin | 不包含 |-
$regex|正则表达式|-


### 组合运算符说明 ###
操作符|类型|用途
---|---|---
$and|数组||如果数组中的所有选择器都匹配，则匹配。
$or|数组|如果数组中的任何选择器匹配，则匹配。 所有选择器必须使用相同的索引。
$not|Selector|如果给定的选择器不匹配，则匹配。
$nor|数组|如果数组中的所有选择器都不匹配，则匹配。
$all|数组|如果它包含参数数组的所有元素，则匹配数组值。
$elemMatch|Selector|匹配并返回所有包含包含至少一个与所有指定查询条件匹配的元素的数组字段的文档。
$allMatch|Selector|匹配并返回所有包含数组字段且其所有元素均与所有指定查询条件匹配的文档。

### 统计信息说明###
字段|说明
---|---
total_keys_examined|检查的索引键数。 当前始终为0。
total_docs_examined|Number of documents fetched from the database / index, equivalent to using include_docs=true in a view. These may then be filtered in-memory to further narrow down the result set based on the selector.
total_quorum_docs_examined|Number of documents fetched from the database using an out-of-band document fetch. This is only non-zero when read quorum > 1 is specified in the query parameters.
results_returned|Number of results returned from the query. Ideally this should not be significantly lower than the total documents / keys examined.
execution_time_ms|Total execution time in milliseconds as measured by the database.

### 1. 简单查询 ###

- 字段查询

```
-- 查询 工作地为：“shenzhen”,且创建时间：“3”年的记录
{
   "selector": {
      "workCity": "shenzhen",
      "workTime": 3
   }
}
```


- 大于，小于，大于或等于，小于或等于

```
-- 查询创办时间大于3年
{
   "selector": {
      "workTime": {
         "$gt": 3
      }
   }
}

```
- between and 查询

```
--查询创办时间  1<workTime<10
{
   "selector": {
      "workTime": {
         "$gt": 1,
         "$lt": 10
      }
   }
}

```

- in 和 not in 查询 

```
-- 查询创办时间是：1年，4年的
{
   "selector": {
      "workTime": {
         "$in": [
            1,
            4
         ]
      }
   }
}
```

- 模糊查询like
couchDB模糊查询，需要使用正则表达来查询

```
--查询城市以“fu”开头公司
{
   "selector": {
      "workCity": {
         "$regex": "^fu"
      }
   }
}
```

- 逻辑“或”查询

```
-- 查询1977年出品，导演是“George Lucas”，或者是“Steven Spielberg"
{
   "selector": {
       "year": 1977,
       "$or": [
         { "director": "George Lucas" },
         { "director": "Steven Spielberg" }
    ]
   }
}
```

- 排序
```
-- 查询主演为“Robert De Niro”，并按照演员名升序，电影上映时间升序排列
{
    "selector": {"Actor_name": "Robert De Niro"},
    "sort": [{"Actor_name": "asc"}, {"Movie_runtime": "asc"}]
}
```

- 查询返回指定字段
```
-- 查询演员是：“Robert De Niro”的记录，并只返回演员名称，电影年份，主键
{
    "selector": { "Actor_name": "Robert De Niro" },
    "fields": ["Actor_name", "Movie_year", "_id", "_rev"]
}
```

-- 分页查询
```
-- 使用bookmark分页查询
```

### 2. 索引 ###
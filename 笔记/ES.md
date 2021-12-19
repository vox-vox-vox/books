# ES

## 文档

一个文档就是一个存储对象，如

```json
{
    "name":         "John Smith",
    "age":          42,
    "confirmed":    true,
    "join_date":    "2014-06-01",
    "home": {
        "lat":      51.5,
        "lon":      0.1
    },
    "accounts": [
        {
            "type": "facebook",
            "id":   "johnsmith"
        },
        {
            "type": "twitter",
            "id":   "johnsmith"
        }
    ]
}
```

文档有三个元数据：

- `_index` 一个 *index* 应该是因共同的特性被分组到一起的文档集合。 例如，你可能存储所有的产品在索引 `products` 中。这个名字必须小写，不能以下划线开头，不能包含逗号
- `_type` 数据可能在索引中只是松散的组合在一起，Elasticsearch 公开了一个称为 *types* （类型）的特性，它允许您在索引中对数据进行逻辑分区。不同 types 的文档可能有不同的字段，但最好能够非常相似。 例如，所有的产品都放在一个索引中，但是你有许多不同的产品类别，比如 "electronics" 、 "kitchen" 和 "lawn-care"。这些文档共享一种相同的（或非常相似）的模式：他们有一个标题、描述、产品代码和价格。他们只是正好属于“产品”下的一些子类。
- `_id` *ID* 是一个字符串，当它和 `_index` 以及 `_type` 组合就可以唯一确定 Elasticsearch 中的一个文档。 当你创建一个新的文档，要么提供自己的 `_id` ，要么让 Elasticsearch 帮你生成（自动生成的 ID 是 URL-safe、 基于 Base64 编码且长度为20个字符的 GUID 字符串。 这些 GUID 字符串由可修改的 FlakeID 模式生成，这种模式允许多个节点并行生成唯一 ID ，且互相之间的冲突概率几乎为零。如`AVFgSgVHUP18jI2wRx0w`）。
- 可用 `/{index}/{type}/{id}` 唯一标识一个文档

文档的其余部分：

- `_version` 在 Elasticsearch 中每个文档都有一个版本号。当每次对文档进行修改时（包括删除）， `_version` 的值会递增。



## 实战

私网地址 : es-cn-7mz2f2ngc003xjryo.elasticsearch.aliyuncs.com

账号：elastic

密码：hdmap!23

访问url：http://elastic:hdmap!23@es-cn-7mz2f2ngc003xjryo.elasticsearch.aliyuncs.com:9200/

```json
# 开启自动创建index，aliyun的es服务默认没有开启
curl --location --request PUT 'http://elastic:hdmap!23@es-cn-7mz2f2ngc003xjryo.elasticsearch.aliyuncs.com:9200/_cluster/settings' \
--header 'Content-Type: application/json' \
--data-raw '{
    "persistent": {
        "action.auto_create_index": "true"
    }
}'

# POST 创建新文档
curl --location --request POST 'http://elastic:hdmap!23@es-cn-7mz2f2ngc003xjryo.elasticsearch.aliyuncs.com:9200/website/blog/' \
--header 'Content-Type: application/json' \
--data-raw '{
  "title": "My first blog entry",
  "text":  "Just trying this out...",
  "date":  "2014/01/01"
}'

## 输入手动id，只需在type后面加入id即可
curl --location --request POST 'http://elastic:hdmap!23@es-cn-7mz2f2ngc003xjryo.elasticsearch.aliyuncs.com:9200/website/blog/123' \
--header 'Content-Type: application/json' \
--data-raw '{
  "title": "My first blog entry",
  "text":  "Just trying this out...",
  "date":  "2014/01/01"
}'


# PUT 
curl -X PUT "http://elastic:hdmap!23@es-cn-7mz2f2ngc003xjryo.elasticsearch.aliyuncs.com:9200/website/blog/123?pretty" -H 'Content-Type: application/json' -d'
{
  "title": "My first blog entry",
  "text":  "Just trying this out...",
  "date":  "2014/01/01"
}
'

# GET
curl --location --request GET 'http://elastic:hdmap!23@es-cn-7mz2f2ngc003xjryo.elasticsearch.aliyuncs.com:9200/website/blog/123?pretty' \
--header 'Content-Type: application/json'

```

es不允许一个索引有多个type


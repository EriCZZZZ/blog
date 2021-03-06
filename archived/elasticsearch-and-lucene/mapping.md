# mapping

*v7.1*

mapping用于定义一个文档和其中的字段是如何存储和被索引的。

可以被用作：

- 指定某个字段被当作全文 *(full text)* 来处理
- 说明每个字段的格式，如数字、日期或者地理位置等
- 一些自定义的规则用来控制动态mapping的插入

## Mapping Type

每个索引有且一种mapping，用来决定索引中的文档应该被如何索引。

> 从v6.0.0开始，一个索引应该只有一个mapping。随着版本的迭代，后续会不再支持一个索引多个mapping。
> [为什么要移除mapping的类型](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/removal-of-types.html)

### Mapping Type 有什么

#### Meta-fields 元数据

每个文档都包含一些相关的元数据。

1. 用于鉴别类型的
    1. \_index 所属的索引
    1. \_type deprecated type
    1. \_id 文档的id
1. 源文档的信息
    1. \_source 原文档
    1. \_size `_source` 占用的字节数
1. 索引相关
    1. \_field_names 所有非null的字段的名称
    1. \_ignored 因格式不正确所以被忽略的字段
1. 路由相关
    1. \_routing 自定义的路由值，用于指定存储的分片
1. 其他的
    1. \_meta 应用自定义的字段

#### Fields or properties 字段与属性

与要存储的的文档相关的字段与属性的列表

## 创建mapping要注意的点 / 创建mapping的最佳实践

### 避免mapping的过度膨胀

在一个mapping中定义过多的字段，会导致性能上的下降。

使用一些属性来避免过度膨胀：

- `index.mapping.total_fields.limit`
- `index.mapping.depth.limit`
- `index.mapping.nested_fields.limit`
- `index.mapping.nested_objects.limit`

### mapping的字段一但定义便无法修改

可以通过新建字段，并reindex来实现修改的目的。

### 动态mapping与声明式 *(explicit)* mapping

默认的，当插入新的文档时，如果有新的字段，将会自动在索引的mapping中，添加对应的字段。
这种mapping称为动态mapping。

某些时候，我们不想让ES自动生成和修改mapping，那么我们可以使用如下的方式解决：

1. 创建索引时定义mapping
1. 使用 `PUT` 接口修改mapping
1. 将 `dynamic` 属性设置为 `strict` 来禁用dynamic mapping
    ```
        PUT /index
        {
                "mappings": {
                        "properties": {
                                "dynamic": "strict"
                        }
                }
        }
    ```

## 字段的数据类型

每个字段都有固定的类型，有如下分类：

### 基础数据类型

#### 字符串 `text` 

用于存储全文索引的数据。

该类型字段的数据在被索引进倒排索引之前，会被 `analyzer` 进行分词。该类型字段不支持排序或聚合。

##### 参数

- `analyzer` [analyzer](#analyzer)
- `boost` [boost](#boost)
- `eager_gloabl_ordinals` [eager\_gloabl\_ordinals](#eager_gloabl_ordinals)
- `fielddata` [fielddata](#fielddata)
- `fielddata_frequency_filter` [fielddata\_frequency\_filter](#fielddata_frequency_filter)
- `fields` [fields](#fields)
- `index` [index](#index)
- `index_options` [index\_options](#index_options)
- `index_prefixes` [index\_prefixes](#index_prefixes)
- `index_phrases` [index\_phrases](#index_phrases)
- `norms` [norms](#norms)
- `position_increment_gap` [position\_increment\_gap](#position_increment_gap)
- `store` [store](#store)
- `search_analyzer` [search\_analyzer](#search_analyzer)
- `search_quote_analyzer` [search\_quote\_analyzer](#search_quote_analyzer)
- `similarity` [similarity](#similarity)
- `term_vector` [term\_vector](#term_vector)

#### 字符串 `keyword`

`keyword` 类型的字段用于存储用来 **精确** 匹配的数据。

##### 参数

- `boost` [boost](#boost)
- `doc_values` [doc\_values](#doc_values)
- `eager_gloabl_ordinals` [eager\_gloabl\_ordinals](#eager_gloabl_ordinals)
- `fields` [fields](#fields)
- `ignore_above` [ignore\_above](#ignore_above)
- `index` [index](#index)
- `index_options` [index\_options](#index_options)
- `norms` [norms](#norms)
- `null_value` [null\_value](#null_value)
- `store` [store](#store)
- `similarity` [similarity](#similarity)
- `normalizer` [normalizer](#normalizer)
- `split_queries_on_whitespace` 是否需要在查询这个字段的时候，按照空白字符来分割。默认 ```false``` 。

#### 数字

用于存储数字。

|支持的类型|范围&介绍|
|----------|---------|
|long|有符号，64位|
|integer|signed, 32-bit|
|short|signed, 16-bit|
|byte|signed, 8-bit|
|double|双精度浮点数，64位，IEEE定义|
|float|单精度浮点数，32位，IEEE定义|
|half_float|半精度浮点数，16位，IEEE定义|
|scaled_float|浮点数，可伸缩的|

e.g.

```curl
PUT my_index
{
  "mappings": {
    "properties": {
      "number_of_bytes": {
        "type": "integer"
      },
      "time_in_seconds": {
        "type": "float"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      }
    }
  }
}
```

##### 参数

- `coerce` 是否尝试进行转换
- `boost` [boost](#boost)
- `doc_values` [doc_values](#doc_values)
- `ignore_malformed` [ignore_malformed](#ignore_malformed)
- `index` [index](#index)
- `null_value` [null_value](#null_value)
- `store` [store](#store)

**scaled_float专用的参数**

`scaling_factor` 用于编码数据。

#### Date 日期

用来存储日期，毫秒精度。

在ES中，使用如下格式表示日期：
- 格式化的字符串
- `long` 毫秒时间戳
- `integer` 秒时间戳

在ES内部，一切时间都会被转换为UTC时间，并存储为 `long` 格式的毫秒时间戳。对于时间的查询将会被转换为 `long` 的范围查询。
查询或聚合的结果会被转换为字段对应类型的字符串。

日期字段的格式可以自定义，默认值取 `"strict_date_optional_time||epoch_millis"` 。

> strict_date_optional_time指的是日期必选时间可选的ISO时间格式， yyyyMMddTHHmmssZ。
> epoch_millis指的是毫秒时间戳。

##### 参数

- `boost` [boost](#boost)
- `doc_values` [doc_values](#doc_values)
- `format` [format](#format)
- `local` 用于解析日期，默认值为 `ROOT locale` 。
- `ignore_malformed` [ignore_malformed](#ignore_malformed)
- `index` [index](#index)
- `null_value` [null_value](#null_value)
- `store` [store](#store)

#### Date nanoseconds

纳秒精度的日期。

#### boolean 布尔值

布尔值，接受 `true` 和 `false` 和 字符串 `"true"` 或 `"false"`。

需要注意的点：
- 聚合接口，使用 `0` 和 `1` 作为 `key` ， 使用 `"true"` 和 `"false"` 作为 `key_as_string`
- 在脚本中时，被认为是数字 `0` 和 `1`

##### 参数

- `boost` [boost](#boost)
- `doc_values` [doc_values](#doc_values)
- `index` [index](#index)
- `null_value` [null_value](#null_value)
- `store` [store](#store)

#### binary data 二进制

存储经过Base64编码的二进制数据，默认不会被索引，即不能被搜索。

**存储的数据不能包含 `\n`**

##### 参数

- `doc_values` [doc_values](#doc_values)
- `store` [store](#store)

#### range

用于存储一个范围。
支持整数、浮点数、long、双精度浮点数、日期和ip。

- `interger_range`
- `float_range`
- `long_range`
- `double_range`
- `date_range`
- `ip_range`

e.g. 创建一个包含日期范围的字段的mapping，并插入一个文档。

```curl
PUT range_index
{
  "settings": {
    "number_of_shards": 2
  },
  "mappings": {
    "properties": {
      "expected_attendees": {
        "type": "integer_range"
      },
      "time_frame": {
        "type": "date_range", 
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }
    }
  }
}

PUT range_index/_doc/1?refresh
{
  "expected_attendees" : { 
    "gte" : 10,
    "lte" : 20
  },
  "time_frame" : { 
    "gte" : "2015-10-31 12:00:00", 
    "lte" : "2015-11-01"
  }
}
```

#### 参数

- `coerce` 是否尝试进行转换
- `boost` [boost](#boost)
- `index` [index](#index)
- `store` [store](#store)

### 复合数据类型

#### Object 对象

对象类型可以看作是JSON对象。JSON格式的数据自身就是分层的。每个文档可能会包含一个JSON对象，如此递归下去。
就像这样：

```crul
PUT my_index/_doc/1
{
  "region": "US",
  "manager": {
    "age":     30,
    "name": {
      "first": "John",
      "last":  "Smith"
    }
  }
}
```
文档自身就是一个JSON对象，其中的 `manager` 字段也是一个JSON对象，显然， `name` 也是一个JSON对象。

在ES内部，整个文档会被存储为一个平铺的kv对列表，多层的对象会被转换为：

```json
{
  "region":             "US",
  "manager.age":        30,
  "manager.name.first": "John",
  "manager.name.last":  "Smith"
}
```

##### 如何声明一个对象类型的字段

```curl
PUT my_index
{
  "mappings": {
    "properties": {
      "region": {
        "type": "keyword"
      },
      "manager": {
        "properties": {
          "age":  { "type": "integer" },
          "name": {
            "properties": {
              "first": { "type": "text" },
              "last":  { "type": "text" }
            }
          }
        }
      }
    }
  }
}
```

与其他字段类型不同，Object字段类型不需要显示的用 `type` 字段来声明。

##### 参数

- `dynamic` [dynamic](#dynamic)
- `enabled` [enabled](#enabled)
- `properties` [properties](#enabled)

#### Nested 支持对Object数组中的每个Object进行独立搜索的字段类型

##### 什么是对每个Object进行独立的搜索？

抽象的讲，就是对Object数组进行搜索，会根据Object的字段是否同属于一个Object实例进行搜索，就叫做独立的搜索。

举个例子，mapping中有一个对象的数组，每个对象有两个字段 `foo` 和 `bar` ，我们希望能够搜索到数组中包含 `foo=1 and bar=2` 的对象的文档，这就叫独立的搜索。

与之相对的，如果搜索到的是该对象数组中，由任意两个Object，一个满足 `foo=1` ，另一个满足 `bar=2` ，这就不是独立的搜索。

##### Object为什么不支持独立的搜索？

由于Lucene没有对象的概念，所以EX在索引Object的时候，会把Object的层级结构平摊成一个kv列表。
而Object数组则会在平摊的过程中，失去 **位于哪一个对象** 的信息，这导致无法按照每个Object进行独立的搜索。

举个例子， `user` 字段是一个Object数组：

```curl
PUT my_index/_doc/1
{
  "group" : "fans",
  "user" : [
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

在ES内部会被平摊成这样：

```json
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```

显然，被索引的信息，失去了 `alice` 和 `white` 是属于同一个Object这样的信息，这就导致了错误的查询结果：

```curl
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}
```

其实并没有一个叫 `Alice Smith` 的用户，结果却会返回对应的数据。

##### nested原理与注意事项

Nested类型，会将数组中的每个Object存储为一个独立的文档（假如数组由10个Object，将会有1个父文档和10个子文档），这将会带来一些性能上的风险。

- `index.mapping.nested_fields.limit` 用来限制一个索引中，最多有多少个 `nested` 字段，默认为50个。
- `index.mapping.nested_objects.limit` 用来限制每个 `nested` 字段中，有多少个Object，默认为10000个。

##### 如何创建与使用

创建时，需要显式的声明字段为 `nested` 类型。

```curl
PUT my_index
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested"
      }
    }
  }
}
```

插入数据和搜索都与普通的搜索方法类似。

这个搜索会返回，拥有一个同时满足 `first` 和 `last` 的 `user` 的文档。

```curl
GET my_index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "Smith" }}
          ]
        }
      }
    }
  }
}
```

##### 参数

- `dynamic` [dynamic](#dynamic)
- `properties` [properties](#enabled)

### 地理位置数据类型

#### 地理点 Geo-Point

用于存储经纬度，用来表示地理位置。
可以用来：
- 寻找在指定多边形的中的点或寻找距某个点特定距离内的点
- 在地理上和距离上进行聚合
- 将地理上的距离与文档的得分关联起来
- 根据距离排列文档

接受如下格式的数据：
- 一个对象，有 ```lat``` 和 ```lon``` 两个字段
- 一个字符串，逗号分割的经纬度
- 字符串，geo hash
- 两个浮点数的数组

##### 浮点数

- `ignore_malformed` [ignore_malformed](#ignore_malformed)
- `ignore_z_value` （默认为）true时，接受第三个元素作为z轴，但不检索。如果为 `false` ，如果有除了两个元素之外的元素，会返回异常。
- `null_value` [null_value](#null_value)

#### 地理区域 Geo-shape

用于表示一个地理区域

### 特殊用途的数据类型

#### IP

用于存储ip地址，支持ipv4和ipv6。

可以用 [CIDR notation](https://www.baidu.com/s?wd=CIDR%20notation) 格式来进行查询。

```curl
GET my_index/_search
{
  "query": {
    "term": {
      "ip_addr": "192.168.0.0/16"
    }
  }
}
```

##### 参数

- `boost` (boost](#boost)
- `doc_values` (doc_values)[#doc_values]
- `index` (index)[#index]
- `null_value` (null_value)[#null_value]
- `store` (store)[#store]

#### 自动补全

用于提供自动补全支持的字段类型

#### 词数统计

用于统计一个字符串中有多少（经过分词之后产生的）词

#### mapper-murmur3

todo

#### mapper-annotated-text

todo

#### percolator

todo

#### Join

`Join` 类型是特别用来在一个索引中建立文档之间的父子关系。

##### 定义

在定义mapping时，预定义若干个父子关系。如：

```curl
PUT my_index
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": "answer"
        }
      }
    }
  }
}
```

##### 插入文档

1. 插入父文档
    
    ```curl
    PUT my_index/_doc/1?refresh
    {
      "text": "This is a question",
      "my_join_field": {
        "name": "question"
      }
    }
    
    PUT my_index/_doc/2?refresh
    {
      "text": "This is another question",
      "my_join_field": "question"
    }
    ```
    
    作为父文档，插入时在 `join` 类型的字段，传递 `name` 字段来体现是哪个父子关系。

1. 插入子文档

    ```curl
    PUT my_index/_doc/3?routing=1&refresh
    {
      "text": "This is an answer",
      "my_join_field": {
        "name": "answer",
        "parent": "1"
      }
    }
    
    PUT my_index/_doc/4?routing=1&refresh
    {
      "text": "This is another answer",
      "my_join_field": {
        "name": "answer",
        "parent": "1"
      }
    }
    ```
    
    插入子文档时，需要同时指明所属的父子关系，和所属的父文档的id。

    **特别的，子文档必须要与父文档存储在同一个分片上，故 `routing` 参数是必须指定的。**

一些限制

1. 一个索引只能有一个 `join` 类型的字段。
1. 父子文档必须要被存储在同一个分片上。
1. 每个元素能有多个子文档，但只能有一个父文档。
1. 可以向一个 `join` 字段增加新的父子关系。
1. 可以向一个父文档追加子文档。

##### 一些特别的

1. 使用 `{join_field_name}#{join_parent_name}` 来筛选特定的 `join` 字段。

1. 可以定义多级 `join` 字段。

1. 一个父类型可以有多个子类型。

    ```curl
    PUT my_index
    {
      "mappings": {
        "properties": {
          "my_join_field": {
            "type": "join",
            "relations": {
              "question": ["answer", "comment"]
            }
          }
        }
      }
    }
    ```

1. 有特殊的查询和聚合方法，如： `has_child query` 和 `has_parent query` 等。

#### Alias 别名

用于给索引中的一个字段增加一个别名。

别名可以修改指向。

- 一些限制
    - 别名只能指向具体的字段，而不能是别名
    - 别名只能指向已经存在的字段
    - 别名只能指给一个字段
    - nested的字段的别名，要和它指向的对象在同一层上
- 写入接口均不支持别名
    - 因为别名不会存储在文档的source中

#### Rank feature

todo

#### Rank features

todo

#### Dense vector

todo

#### Sparse vector

todo

### 数组

es中没有专门的数组类型，每一个字段都可被看作是一个数组。但一个数组中只能有一种类型的值。

**对象数组无法被分别搜索，如果需要，请参考 [`nested`](#Nested 支持对Object数组中的每个Object进行独立搜索的字段类型)。**

### multi-fields 复合字段

对于一个字段来说，经常会遇到一份数据，按照多种方式解析的情景。复合字段就是用来处理这种情况的。

## Mapping的字段类型的参数

不同的字段类型，支持不同的参数。

### dynamic

用于控制索引或 `object` 是否允许动态的插入新的字段，以及不允许时，新增的字段会被如何处理。

**特别的，`子Object` 会从 `父Object` 或 `mapping` 继承这个属性。**

- `true` 默认值，在索引新的文档时，如果有新的字段，会自动的插入到 `mapping` 中。
- `false` 忽略新增的字段，不会被索引，即无法被搜索。但是仍然会被存储，并在 `_source` 中返回。
- `strict` 严格模式，如果有新的字段，会抛出异常。


> 该属性可被修改

### enabled

用于控制字段是否需要被索引。**只有 `mapping` 或 `Object` 支持。**

- `true` 会被索引
- `false` 不会被索引，但仍然会被存储，可以在 `_source` 中查询到。

### properties

用于表明 `mapping` 和 `Object` 和 `nested` 下的字段。

在搜索或其他的接口中，使用 `obj1.obj2.target_field` 来指向目标字段。

### store

用来标记字段是否为 `stored`。

大部分场景下， `_source` 字段和查询接口中的过滤器都能够实现 `store` 属性的功能。

#### 用途

1. 可以在查询接口中使用 `stored_fields` 字段来指定需要返回的数据。
    ```curl
        GET my_index/_search
        {
          "stored_fields": [ "title", "date" ] 
        }
    ```
1. 可以用于没有 `_source` 存在的接口。

### doc_values

用来实现排序、聚合和在脚本中访问字段的值。

使用倒排索引能够快速查询到拥有某个或某些短语的文档。
但是对于聚合、排序和脚本中的访问的情形，还需要 **通过查找到文档，然后查看文档中的字段包含的短语** 的能力。

将doc_values设置为true的字段，会在文档被索引的时候，同时构建一份与 `_source` 内容相同的 `Doc values` 。 `Doc values` 是存储在硬盘上的数据结构，提供了这种能力。与倒排索引不同的是， `Doc values` 是以按列的方式进行存储的。这种存储方式能够更加高效的进行排序和聚合。

大部分字段类型都支持 `doc_values` ，需要特别指出的是， `analyzed string` 字段类型不支持。

### null_value

提供让null值可以被搜索的能力。

默认的，空值在插入文档时不会被索引，即不能被搜索。

为了能够搜索到某个字段为空的文档，可以将该字段的 `null_value` 设为一个特定的值，作为空值的具体表现。
特定的值必须与字段的类型一致。比如一个 `long` 类型的字段的 `null_value` 就不能设置成字符串。
设定好特定的值后，就可以在搜索接口，通过传递这种值作为NULL的代指来搜索该字段为空的文档。

**如果一个文档的此字段的值被设置为了 `null_value` ，那么这个会被认为是空而不是这个值。**

设置了 `null_value` 只意味着创建了一个“别名”，而不会将 `null_value` 存储到 `_source` 中。

### boost

字段的权重值。浮点数，默认为1.0。

只有 `term` 查询有效。

**不建议在索引中设置boost，作为替代品，使用查询时设置boost**

### format

ES有一些内置的 [时间格式](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/mapping-date-format.html#built-in-date-formats) 
，也可用 `yyyy-MM-dd HH:mm:ss` 这种形式自定义时间的格式。

```curl
PUT my_index
{
  "mappings": {
    "properties": {
      "a_date": {
        "type": "date",
        "formate": "yyyy-MM-dd"
      }
    }
  }
}
```

### ignore_malformed

提供配置处理异常输入的策略。
该字段设为 `true` 时，新的文档被插入时，如果该字段的格式与设定不同，将会跳过该字段，不会被索引，文档的其他字段会正常插入。
默认的，该属性被设置为 `false` ，遇到错误的格式，将会抛出异常，整个文档会被拒绝插入。

特别的，该属性对于 `nested, object, range` 无效。

### index

该属性用来配置字段是否会被索引，代表着是否可以被搜索。

### eager_gloabl_ordinals

用于控制什么时候建立 `全局序列` 。

#### 全局序列

为了支持聚合和其他需要按照文档级别进行搜索的操作，ES使用了 (doc_values)[#doc_values) 来存储了文档。
类似于 `keyword` 等基于短语搜索的字段，使用 `有序图(ordianl mapping)` 实现压缩率更高的方式存储相应的文档。
`有序图` 通过给每个短语指定一个单调增长的id或根据字符顺序作为相应的顺序。

### fields

用于定义 `multi-fields` 。用于：

1. 对于一个字段，可以有多种存储的格式，来满足不同的查询需求。
1. 另外，可以用于使用不同的分词器来对同一个字段进行分析，满足多种查询需求。

#### 例子

1. 一个有多个存储格式的字段

    1. 创建一个 `multi-fields` 字段 `city` ，该字段有两个存储格式，分别为 `text` 和 `keyword` 。可以分别进行全文搜索和精确搜索。
    
        ```curl
        PUT my_index
        {
          "mappings": {
            "properties": {
              "city": {
                "type": "text",
                "fields": {
                  "raw": {
                    "type":  "keyword"
                  }
                }
              }
            }
          }
        }
        
        PUT my_index/_doc/1
        {
          "city": "New York"
        }
        ```
    
    1. 利用两种存储格式分别进行了全文搜索、排序和聚合。
    
        ```curl
        GET my_index/_search
        {
          "query": {
            "match": {
              "city": "york"
            }
          },
          "sort": {
            "city.raw": "asc"
          },
          "aggs": {
            "Cities": {
              "terms": {
                "field": "city.raw"
              }
            }
          }
        }
        ```

1. 一个字段有多个分词器
    
    1. 创建mapping

    ```curl
    PUT my_index
    {
      "mappings": {
        "properties": {
          "text": {
            "type": "text",
            "fields": {
              "english": {
                "type":     "text",
                "analyzer": "english"
              }
            }
          }
        }
      }
    }

    PUT my_index/_doc/1
    { "text": "quick brown fox" }
    ```

    1. 查询

    ```curl
    GET my_index/_search
    {
      "query": {
        "multi_match": {
          "query": "quick brown foxes",
          "fields": [
            "text",
            "text.english"
          ],
          "type": "most_fields"
        }
      }
    }
    ```

### ignore\_above

用于忽略超过某个长度之后的数据。超过的数据会被忽略，即不会被索引也不会被额外存储，但是会被存储在\_source中。

**特别的，对于数组来说，会对每个元素进行检查。**

**传递的值是按照字节进行计算的，utf8等格式的字符需要自行变换**

### index\_options

用于控制什么信息需要被存储到倒排索引中，用于搜索和高亮。

接受以下的参数：

1. `docs` 只存储对应的文档的id，用来满足“有哪些文档包含这个短语”。 *其他字段默认*
1. `freqs` 存储文档id和短语（在文档中）出现的频率，文档的得分与短语的频率成正相关。
1. `positions` 文档id、频率和短语所在的位置。位置用于 ```proximity or phrase queries``` 。 *analyzed string默认*
1. `offsets`  文档id、频率、短语所在位置和起止字符的偏移量，提供将短语映射到原数据的能力，用于加速高亮和 ```unifiedhighlighter``` 。

### norms

用于控制字段内容的长度是否会影响文档的得分。接受 ```true``` 和 ```false``` (default)作为参数。

开启该功能，会增加磁盘占用，如果不需要该功能的话，可以将该功能关闭。

### similarity

用于设置使用什么相似度算法或相似度，默认为 ```BM25``` 。

自带的相似度算法：

1. ```BM25``` 
1. ```classic```
1. ```boolean``` 简单的二极区分，匹配或未匹配。

### normalizer

用于在应用分词器之前，用于保证生成归一化的token。

### analyzer

用于配置字段的分析器。

Elasticsearch提供了一些开箱即用的预定义分析器。
也有一些字符过滤器( *character filters* )，分词器( *tokenizers* )和短语过滤器( *Token Filters* )，可以用来组成一系列自定义的分析器。

分析器可以在每次查询、每个字段或每个索引中被分别定义。

#### 获取要应用的分析器的顺序

- 索引一个文档时
    1. 定义在该字段上的 `analyzer`
    1. 定义在该索引上的 `default`
    1. 标准分析器

- 搜索时
    1. 在全文搜索查询中配置的 `analyzer`
    1. 在该字段中定义的 `search_analyzer`
    1. 在该字段中定义的 `analyzer`
    1. 在该索引上的 `default_search`
    1. 在该索引上的 `default`
    1. 标准分析器

#### `search_quote_analyzer`

todo

### fielddata

`fielddata` 是一个内存数据结构，用于给 `text` 字段增加根据文档标志查找文章的字段有什么内容的能力。

`fielddata` 会在该字段第一次被用于聚合、排序或脚本访问时构建。构建是通过遍历每个段的全部倒排索引，然后存储在JAVA堆上。

**默认的，因为构建的耗费很大和暂用内存多且时间长，fielddata是false状态。**

#### `fielddata_frequency_filter`

用于控制什么数据会被加载到 `fielddata` 中。

### index_prefixes

用于控制是否将前缀存储在另外的字段中以加速前缀搜索。

### index_phrases

用于控制是否将短语存储到另外的字段中，用于加速短语搜索。

### position_increment_gap

todo

### search_analyzer

用于指定搜索用的分析器。

### search_quote_analyzer

todo

### term_vector

用于指定 `analyzed` 字段的什么额外的分词信息需要被存储。

|可以指定的值|含义|
|----------|-----------|
|no|默认值，不需要存储。|
|yes|经过分词的词汇|
|with_positions|词汇和位置|
|with_offsets|词汇和词汇的偏移量|
|with_positions_offsets|词汇、位置、偏移量|
|with_positions_payloads|词汇、位置、payloads|
|with_positions_offsets_payloads|词汇、位置、偏移量、payloads|


1. 位置，指的多个词汇在一个数据中的顺序。
1. payloads，自定义的权重值。






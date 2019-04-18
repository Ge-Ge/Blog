# Graphql实践——分页
----
## Graphql分页
graphql实现分页有以下两种方式：
1. 基于偏移量,需要提供第几页， 每页的数量
2. 基于游标或者id，提供每页数量，与 游标/id。  
对于游标分页[Relay](https://github.com/facebook/relay)(Facebook家的Graphql库) 定了一套规范 [Relay-style cursor pagination](https://facebook.github.io/relay/graphql/connections.htm)  

**基于偏移量的分页实现简单，但存在以下问题：**
 - 性能问题，虽然可以使用 “延迟关联” 解决，但会使sql语句变得复杂
    ```sql
    # 假设 有一个 product商品表，当商品表数量足够多时，这个查询会变得非常缓慢，
    SELECT id, name FROM product LIMIT 1000, 20;
    # 如果我们提供一个边界值，比如id，无论翻页到多么后面，其性能都会很好
    SELECT id, name FROM product WHERE id > 1000 LIMIT 20;
    ```

- 删除列表数据时，导致获取下一页的数据缺失
    ```sql
    # 假设 总共有11条数据，一页显示10条，总页数为 2 页。
    # 当调用接口删除 第 1 页的 1 条数据，然后进行翻页时，因为只剩下10条数据，所以下面的sql会查不到数据。
    SELECT id, name FROM product LIMIT 10, 10;
    ```
 
**基于游标/ID 的分页，也存在硬伤：**  
- 如何实现跳往第 n 页的功能  
    难道要获取 相应的游标再进行翻页？:joy: 所以它更适用于无限加载，或者只有 上一页/下一页 的情景上,对于跳往第n页还是需要用到基于偏移量的分页。  

**所以我们需要同时支持这两种分页。**  
![](../.gitbook/assets/graphql/我全都要.jpeg)
### Relay 式的游标分页
Relay 定义了 PageInfo，Edges，Edge Types，Node，Cursor等对象 用于实现灵活的分页。👇是Relay给出的一个query例子。
```
{
  user {
    id
    name
    friends(first: 10, after: "opaqueCursor") {
      edges {
        cursor
        node {
          id
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
}
```
friends 连接会返回一个对象，这个对象的名称会以 `Connection`结尾，如`friendConnection`, `Connection`中必需包含`PageInfo`，`Edges`。
#### PageInfo
Relay 在返回的游标连接上提供了一个 **PageInfo** 对象，其必需包含 hasPreviousPage， hasNextPage。
- startCursor : 列表中 第一个项的游标id，字符串
- endCursor : 最后一个项的游标id
- hasPreviousPage : 是否有上一页，布尔值
- hasNextPage :是否有下一页

> 游标是不透明的，并且它们的格式不应该被依赖，建议用 base64 编码它们。

#### Edges
`Edges`：类型为 `LIST` ,必需包含`Edge Types`  
`Edge Types`：类型为 `Object`,必需包含 `Node`，`Cursor`  
`Node`: 类型可以为 标量，枚举，对象，接口，联合类型,此字段无法返回列表。
`Cursor`: 类型为`String`  

通过Edges，列表数据中每一项都包含一个Cursor、Node,但我们基本很少需要Cursor。

#### 上一页/下一页
下一页分页，需要两个参数。
- first 页数大小，必需为一个非负整数。
- after 游标id。
服务器最多返回 first 条 游标id之后（不包括游标id） 的数据。

上一页分页，需要两个参数。
- last 页数大小，必需为一个非负整数。
- before 游标id。
服务器最多返回 last 条 游标id之前（不包括游标id） 的数据。

> first跟last不应该同时使用，这会使判断 上一页/下一页 变得麻烦。

### 支持基于偏移量的分页
我们需要在query时，把跳过多少条记录 这个参数给到 service 端，后端根据这个值 是否 存在 去使用不同分页方式。[Prisma](https://github.com/prisma/prisma)把这个参数命名为 `skip`，这里我们与其保持一致。
```
{
  user {
    id
    name
    friends(first: 10, after: "opaqueCursor", skip: 1) {
      edges {
        cursor
        node {
          id
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
}
```
### 基于egg.js实现基于偏移量 与 Relay 式的游标分页
👉[分页例子：包含前后端](https://github.com/Ge-Ge/practice.git)  
后端项目目录结构如👇
```
./app/
├── extend
├── graphql
│   ├── common                  # 定义公用的 Schema 和类型，如pageInfo
│   │   ├── resolver.js
│   │   ├── scalars
│   │   │   └── cursor.js       # 定义cursor数据类型
│   │   └── schema.graphql
│   ├── mutation
│   │   └── schema.graphql
│   ├── query
│   │   └── schema.graphql
│   └── user                    # user
│       ├── connector.js
│       ├── resolver.js
│       └── schema.graphql
```
定义 `PageInfo`对象,`Cursor`标量 。
```graphql
# graphql/common/schema.graphql
scalar Cursor
type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: Cursor
    endCursor: Cursor
}
```
`PageInfo`的`resolver`层
```js
// graphql/common/resolver
'use strict';
module.exports = {
  Cursor: require('./scalars/cursor'), // eslint-disable-line
  PageInfo: {
    hasNextPage(root) {
      // 在Connector层（如UserConnector）返回PageInfo对象时，我们可以返回 function 或 boolean，function能够支持更加复杂的判断
      if (typeof root.hasNextPage === 'function') return root.hasNextPage();
      return root.hasNextPage;
    },
    hasPreviousPage(root) {
      if (typeof root.hasPreviousPage === 'function') return root.hasPreviousPage();
      return root.hasPreviousPage;
    },
  },
};

```

+++
title = "使用 Pydantic 统一 Python 和 RESTful API 的命名规范"
date = 2025-02-05
description = "Python 后端用 snake_case，前端 API 要 camelCase，手动转换既累又容易出错。用 Pydantic 的 alias_generator 和 to_camel 可以自动搞定，GET 和 POST 请求都能覆盖。"
tags = ["Python", "Pydantic", "FastAPI", "RESTful API", "命名规范"]

[extra.comments]
issue_id = 5

[[extra.faq]]
question = "为什么前后端的命名规范会不一样？"
answer = "JavaScript 社区用 camelCase（比如 itemId），Python 的 PEP 8 规范用 snake_case（比如 item_id）。两边都是各自语言的标准写法，硬改哪边都不合适，所以需要一个中间层来自动转换。"

[[extra.faq]]
question = "Pydantic 的 alias_generator 和手动写 alias 有什么区别？"
answer = "手动 alias 是一个字段一个字段地写映射，适合 GET 请求里少量参数的场景。alias_generator 是在 Model Config 里统一设置转换规则（比如 to_camel），所有字段自动生效，适合 POST 请求体里字段多的情况。两种可以混着用。"

[[extra.faq]]
question = "populate_by_name = True 是什么意思？"
answer = "默认情况下，设了 alias 之后只能用 alias 名称来填充字段。设了 populate_by_name = True，就可以同时用原始字段名（snake_case）和 alias（camelCase）来传值，两种都认。后端测试和前端调用都方便。"

[[extra.faq]]
question = "这种方案对 OpenAPI 文档有影响吗？"
answer = "有，而且是好的影响。FastAPI 自动生成的 OpenAPI 文档会展示 camelCase 的字段名，前端开发者看到的文档直接就是他们需要用的字段名，不需要再做心理转换。"

[[extra.faq]]
question = "Pydantic V1 和 V2 的写法有什么不同？"
answer = "Pydantic V2 推荐用 model_config = ConfigDict(...) 替代 class Config，alias_generators 模块也从 pydantic.alias_generators 导入。本文的写法兼容 Pydantic V2，如果你还在用 V1，需要把 ConfigDict 改成 class Config 的写法。"
+++

写 Python 后端的人都知道一个别扭的事：你在代码里写 `item_id`，前端拿到的 JSON 也是 `item_id`，但前端同事会过来问你——"能不能改成 `itemId`？我们 JavaScript 都是 camelCase。"

改？每个字段都手动映射一遍，累。不改？前端代码里全是下划线，不伦不类。

Pydantic 可以一行配置解决这个问题。

<!--more-->

## 问题在哪

两套命名规范各有各的道理：

- **Python (PEP 8)**：`snake_case` —— `item_id`, `search_query`, `created_at`
- **JavaScript / REST API**：`camelCase` —— `itemId`, `searchQuery`, `createdAt`

都是各自社区的标准，谁也说服不了谁。硬要前端写 `item_id` 或者后端写 `itemId`，都是在给一边找不痛快。

正确的做法是：**后端代码里继续写 snake_case，API 对外暴露 camelCase，中间让 Pydantic 自动转换。**

<img src="/images/pydantic-naming-flow.svg" alt="Python snake_case 到 API camelCase 的转换流程" style="width:100%;max-width:900px;" />

## GET 请求：用 Query 的 alias 参数

GET 请求的参数在 URL 查询字符串里，数量通常不多。这种场景用 `Query` 的 `alias` 参数逐个映射就行：

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def get_item(
    item_id: int = Query(..., alias="itemId"),
    search_query: str | None = Query(None, alias="searchQuery"),
):
    return {"item_id": item_id, "search_query": search_query}
```

前端这样调：

```
GET /items/?itemId=42&searchQuery=example
```

后端函数里拿到的就是 `item_id=42`、`search_query="example"`，完全符合 PEP 8。

响应结果：

```json
{
  "item_id": 42,
  "search_query": "example"
}
```

这里有个细节——**响应的 JSON 字段名还是 snake_case**。如果你想让响应也变成 camelCase，需要用 `response_model` 配合 Pydantic 模型，下一节就会讲到。

## POST 请求：用 alias_generator 批量转换

POST 请求的 body 通常字段更多，一个一个写 `alias` 太麻烦。Pydantic 提供了 `alias_generator`，在模型配置里写一次，所有字段自动转换：

```python
from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict, Field
from pydantic.alias_generators import to_camel

app = FastAPI()

class ItemCreate(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
    )

    item_id: int = Field(..., description="The ID of the item")
    search_query: str | None = Field(None, max_length=50, description="A search query")
    created_at: str | None = Field(None, description="Creation timestamp")

@app.post("/items/")
async def create_item(item: ItemCreate):
    return item
```

三个关键配置：

1. **`alias_generator = to_camel`**：自动把 `item_id` 映射为 `itemId`，`search_query` 映射为 `searchQuery`，所有字段一步到位。
2. **`populate_by_name = True`**：允许同时使用 snake_case 和 camelCase 来传值。没有这个配置，只能用 camelCase。
3. **`to_camel`**：Pydantic 内置的转换函数，从 `pydantic.alias_generators` 导入。

前端发 POST 请求：

```json
POST /items/
Content-Type: application/json

{
  "itemId": 42,
  "searchQuery": "example",
  "createdAt": "2025-02-05T10:00:00Z"
}
```

后端收到后，`item.item_id` 就是 42，`item.search_query` 就是 `"example"`——Python 代码里始终是 snake_case。

响应结果（因为 `alias_generator` 也影响序列化，会自动用 camelCase 输出）：

```json
{
  "itemId": 42,
  "searchQuery": "example",
  "createdAt": "2025-02-05T10:00:00Z"
}
```

## 让 GET 的响应也变成 camelCase

前面说了，GET 请求直接 `return {"item_id": ...}` 的话，响应是 snake_case。要统一的话，加一个 `response_model`：

```python
class ItemResponse(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
    )

    item_id: int
    search_query: str | None = None

@app.get("/items/", response_model=ItemResponse)
async def get_item(
    item_id: int = Query(..., alias="itemId"),
    search_query: str | None = Query(None, alias="searchQuery"),
):
    return ItemResponse(item_id=item_id, search_query=search_query)
```

这样 GET 请求的响应也会自动序列化为 camelCase：

```json
{
  "itemId": 42,
  "searchQuery": "example"
}
```

## 抽个基类，别重复写

如果项目里很多模型都需要 camelCase 转换，每个都写一遍 `model_config` 太啰嗦。抽一个基类：

```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

class CamelModel(BaseModel):
    """所有需要 camelCase API 风格的模型都继承这个。"""
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
    )

# 之后直接继承就行
class ItemCreate(CamelModel):
    item_id: int
    search_query: str | None = None
    created_at: str | None = None

class UserProfile(CamelModel):
    user_name: str
    email_address: str
    is_active: bool = True
```

所有继承 `CamelModel` 的模型自动就有 camelCase 的 alias。干净。

## 注意事项

几个容易踩的坑：

1. **`by_alias` 参数**：如果你手动调用 `model.model_dump()` 序列化，默认输出 snake_case。要输出 camelCase，需要 `model.model_dump(by_alias=True)`。FastAPI 的 `response_model` 会自动处理这个。

2. **嵌套模型**：`alias_generator` 只影响当前模型的字段名，不会递归转换嵌套模型。嵌套的模型也需要配置 `alias_generator`，所以用基类继承的方式最省事。

3. **和 ORM 配合**：如果你用 SQLAlchemy 之类的 ORM，ORM 模型和 Pydantic 模型是分开的。ORM 用 snake_case，Pydantic 模型负责转换，不要在 ORM 层做命名转换。

4. **OpenAPI 文档**：配了 `alias_generator` 之后，FastAPI 自动生成的 Swagger 文档里会显示 camelCase 的字段名，前端看文档直接就是他们需要的字段名。

## 小结

思路就一句：Python 代码里写 snake_case，API 对外暴露 camelCase，中间让 Pydantic 桥接。

具体怎么落地：

- GET 请求参数少，用 `Query(alias="camelCase")` 逐个映射
- POST 请求体字段多，用 `alias_generator = to_camel` 批量转换
- 加 `populate_by_name = True` 让两种命名都能传值
- 抽一个 `CamelModel` 基类，全项目复用

这样前后端各写各的风格，谁都不用迁就谁。

+++
title = "使用 Pydantic 统一 Python 和 RESTful API 的命名规范"
date = 2025-02-05

[extra.comments]
issue_id = 6
+++

在现代 Web 开发中，API 风格和命名规范是确保前后端顺畅协作的重要方面。尤其是前后端分离的架构，前端通常使用 `camelCase` 风格，而后端（尤其是 Python）则遵循 PEP 8 风格，即 `snake_case`。为了在前后端之间保持一致性，我们可以使用 Pydantic 这个模块来自动转换和映射命名风格。本文将介绍如何使用 Pydantic 统一 Python 后端和 RESTful API 的命名规范，并在 **GET** 和 **POST** 请求中实现参数和请求体的命名统一。

<!--more-->

### 1. **统一命名规范的必要性**

通常情况下：

- **前端**：JavaScript 和 TypeScript 的社区更倾向于使用 `camelCase`，因为这是 JavaScript 的命名约定。
- **后端**：Python 更倾向于使用 `snake_case`，这是 PEP 8 风格指南的推荐。

这种命名风格上的差异，往往会导致前后端之间的协作复杂化。为了确保规范性和一致性，我们可以通过以下方式来进行处理：

- **前端**：保持 `camelCase` 风格的参数。
- **后端**：使用 `snake_case` 风格的参数，并且能够自动转换成前端使用的命名风格。

### 2. **使用 Pydantic 实现命名规范化**

Pydantic 是一个非常强大的 Python 数据验证和序列化工具。通过 Pydantic，我们可以灵活地定义数据模型，并使用 `alias_generator` 和 `populate_by_name` 来自动处理命名风格转换。

#### 实现方式：

- **GET 请求**：使用 `Query` 接收查询参数，并通过 `alias` 映射 `camelCase` 参数到后端使用的 `snake_case` 参数。
- **POST 请求**：使用 Pydantic 模型，并通过 `alias_generator` 和 `to_camel` 将请求体参数从 `snake_case` 转换为 `camelCase`。

### 3. **GET 请求中的参数转换**

在 **GET** 请求中，参数通常通过查询字符串传递。我们可以通过 Pydantic 的 `Query` 来接收查询参数，并在其中使用 `alias` 将 `camelCase` 转换为 `snake_case`，从而确保后端遵循 Python 的命名约定。

#### 示例代码：GET 请求中的 `alias` 转换

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def get_item(
    item_id: int = Query(..., alias="itemId"),  # 'itemId' 是前端传递的参数
    q: str | None = Query(None, alias="searchQuery")  # 'searchQuery' 是前端传递的参数
):
    return {"item_id": item_id, "q": q}
```

### 代码解释：

- **前端使用 `camelCase`**：前端传递的查询参数是 `itemId` 和 `searchQuery`。
- **后端使用 `snake_case`**：后端处理时，自动将查询参数映射到 `item_id` 和 `q`，符合 PEP 8 规范。

### 4. **POST 请求中的请求体转换**

对于 **POST** 请求，我们可以使用 Pydantic 模型，在后端使用 `snake_case` 参数，但通过 `alias_generator` 和 `to_camel` 来支持前端传递的 `camelCase` 参数。

#### 示例代码：POST 请求中的请求体转换

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field
from pydantic.alias_generators import to_camel

app = FastAPI()

class Item(BaseModel):
    item_id: int = Field(..., description="The ID of the item to get")
    q: str | None = Field(None, max_length=50, description="A search query")

    class Config:
        alias_generator = to_camel  # 使用 to_camel 来转换字段名
        populate_by_name = True  # 允许使用字段名称而不是别名

@app.post("/items/")
async def create_item(item: Item):
    return {"item_id": item.item_id, "q": item.q}
```

### 代码解释：

- **`alias_generator = to_camel`**：Pydantic 提供了一个 `to_camel` 别名生成器，自动将字段名从 `snake_case` 转换为 `camelCase`。
- **`populate_by_name = True`**：允许通过字段名称而不是别名来填充字段，这样就可以支持前端使用 `camelCase` 参数名。

### 5. **测试和运行**

启动 FastAPI 应用后，我们可以通过以下 URL 来测试：

#### GET 请求：

```json
http://127.0.0.1:8000/items/?itemId=42&searchQuery=example
```

响应将是：

```json
{
  "item_id": 42,
  "q": "example"
}
```

![image-20250205174439243](https://fullstackjam-1257718633.cos.ap-nanjing.myqcloud.com/imgs/202502051744911.png)

#### POST 请求：

请求：

```json
POST http://127.0.0.1:8000/items/
Content-Type: application/json

{
  "itemId": 42,
  "searchQuery": "example"
}
```

响应将是：

```json
{
  "item_id": 42,
  "q": "example"
}
```

![image-20250205174556012](https://fullstackjam-1257718633.cos.ap-nanjing.myqcloud.com/imgs/202502051745453.png)

### 6. **总结**

通过使用 Pydantic 的 `alias` 和 `alias_generator` 功能，我们可以高效地实现以下目标：

1. **统一命名规范**：确保前后端之间的命名风格一致，前端使用 `camelCase`，后端使用 `snake_case`，从而避免了不同风格带来的问题。
2. **遵循 PEP 8 标准**：后端代码保持符合 Python 的 PEP 8 风格指南，提升代码的可读性和一致性。
3. **无缝协作**：通过 Pydantic 自动转换功能，前后端能够轻松对接，无需手动转换参数名，从而提高开发效率并减少出错的风险。

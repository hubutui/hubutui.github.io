---
title: "如何让 LLM 稳定地返回结构化输出"
date: 2025-06-02T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - vllm
  - LLM
  - 结构化输出
  - json
  - json_object
  - json_schema
---

## 前言

目前，LLM 的输出都是纯文本，但是有的时候我们希望大模型能够输出结构化的结果，特别是 JSON 结果，方便我们在后续的步骤中使用．

这里，我们介绍一下 vllm 的支持，以及 openai 接口的支持．

## vllm 的支持

vllm 支持以下配置：

1. `guided_choice`：模型输出为指定选项之一．
2. `guided_regex`：模型输出满足指定的正则表达式．
3. `guided_json`：模型输出满足指定的 JSON Schema．
4. `guided_grammar`：模型的输出满足指定的语法规则．这个我们暂时不用．
5. `structural_tag`：暂时不用．有兴趣查询官方文档．

以上的这些配置都不在 openai 接口的参数里，一般都要通过 `extra_body` 参数来设置．

注意，你必须确定模型是用 vllm 服务启动的，才可以使用这些配置．此外，如果你的请求经过 new-api 等类似的 API 管理服务的中转，`extra_body` 参数可能不会被正确传递．

## 代码示例

下面我们用代码来举例，使用模型为：`Qwen/Qwen2.5-32B-Instruct`，基础代码如下：

```python
from enum import Enum
from typing import List

from openai import OpenAI
from openai.lib._pydantic import to_strict_json_schema
from pydantic import BaseModel

model = "Qwen/Qwen2.5-32B-Instruct"
base_url = "your-base-url"
api_key = "sk-your-key"

client = OpenAI(api_key=api_key, base_url=base_url)
```

### guided_choice

`guided_choice` 测试，也就是要求模型输出的结果仅限于给定的选择．

```python
completion = client.chat.completions.create(
    model=model,
    messages=[{"role": "user", "content": "李白是唐代诗人吗？"}],
    extra_body={"guided_choice": ["是", "否"]},
)
print(completion.choices[0].message.content)
```

如果一切正常，模型应该会输出"是"．如果你无法控制你的请求经过哪些中间层的转发处理，甚至也无法确定模型是不是用 vllm 部署，则可能会有额外的输出内容．例如，我们把模型改成调用硅基流动的 deepseek-v3，则可能输出更多的信息，模型会对结果进行详细的解释．

### guided_regex

也就是让模型输出满足正则表达式的结果．

```python
completion = client.chat.completions.create(
    model=model,
    messages=[
        {
            "role": "user",
            "content": "帮我随机生成一个电子邮件地址．例如: alan.turing@enigma.com\n",
        }
    ],
    extra_body={"guided_regex": r"\w+@\w+\.com\n"},
)
print(completion.choices[0].message.content)
```

### guided_json

这里是要求模型输出的内容要符合指定的 JSON Schema．其中 JSON Schema 可以自己全部手写，也可以从 `pydantic.BaseModel` 生成．这里我们演示的是从 `pydantic.BaseModel` 生成 JSON Schema 的方式．这通常是比较简单直接的做法．

```python
class Step(BaseModel):
    explanation: str
    output: str


class MathResponse(BaseModel):
    steps: list[Step]
    final_answer: str

# pydantic basemodel 转换为 json schema
# 推荐使用后者
json_schema = MathResponse.model_json_schema()
# json_schema = to_strict_json_schema(MathResponse)


completion = client.chat.completions.create(
    model=model,
    messages=[
        {"role": "system", "content": "你是一个聪明的数学助手．"},
        {"role": "user", "content": "求解方程 8x + 31 = 2."},
    ],
    extra_body=dict(guided_json=json_schema),
)

content = completion.choices[0].message.content
answer = json.loads(content)
```

这里输出的结果可以直接用 `json.loads` 反序列化为一个对象来使用．实际上这里等价于设置 `response_format` 为 `json_schema`．等价代码如下：

```python
class Step(BaseModel):
    explanation: str
    output: str


class MathResponse(BaseModel):
    steps: list[Step]
    final_answer: str


json_schema = to_strict_json_schema(MathResponse)
response_format = {
    "type": "json_schema",
    "json_schema": {
        "schema": to_strict_json_schema(MathResponse),
        "name": MathResponse.__name__,
        "strict": True,
    },
}
completion = client.chat.completions.create(
    model=model,
    messages=[
        {"role": "system", "content": "你是一个聪明的数学助手．"},
        {"role": "user", "content": "求解方程 8x + 31 = 2."},
    ],
    response_format=response_format,
)

content = completion.choices[0].message.content

answer = json.loads(content)
```

这样子写起来有些麻烦，openai-python 新增了一个接口，方便我们直接传入 `pydantic.BaseModel`，他自己帮我们做好 JSON Schema 的生成，返回的结果也是一个对象，不用再自己做 JSON 的反序列化．等价代码如下：

```python
class Step(BaseModel):
    explanation: str
    output: str


class MathResponse(BaseModel):
    steps: list[Step]
    final_answer: str

completion = client.beta.chat.completions.parse(
    model=model,
    messages=[
        {"role": "system", "content": "你是一个聪明的数学助手．"},
        {"role": "user", "content": "求解方程 8x + 31 = 2."},
    ],
    response_format=MathResponse,
)
# completion.choices[0].message.parsed 是一个 MathResponse 对象，可以直接使用
print(completion.choices[0].message.parsed)
```

这里返回的结果就直接是一个 `MathResponse`，非常方便使用．

### JSON mode

有的模型并不支持 JSON Schema，而是支持 JSON mode．例如 gpt-3.5-turbo 支持的就是 JSON mode．JSON mode 只是保证返回的内容为有效的 JSON 字符串，但是不保证它遵循预定的 JSON Schema．当然，你也可以在提示词中详细地描述需要生成的 JSON 数据的信息，确保得到更加可靠的 JSON 结果．

此外，你还要必须在提示词中显式要求模型返回 JSON 输出．

```python
completion = client.chat.completions.create(
    model=model,
    messages=[
        {
            "role": "user",
            "content": "介绍一下李白的生平，以JSON格式返回结果",
        }
    ],
    response_format={"type": "json_object"},
)
print(completion.choices[0].message.content)
```

例如上述例子，我们指定了 `response_format={"type": "json_object"}`，返回的输出会是有效的 JSON 输出，但是并不确保其遵循特定的 JSON Schema．

好在现在多数模型都是支持的 JSON Schema，仅支持 JSON mode 的也较少使用了．

## 大一统

实际上，vllm 的 `guided_choice`, `guided_regex`, `guided_json` 都可以统一到 `response_format` 为 `json_schema` 中来．特别提一下，JSON Schema 里是可以写正则表达式的．openai 接口支持的 JSON Schema 规范可以参考 [Supported schemas](https://platform.openai.com/docs/guides/structured-outputs#supported-schemas)．

因此，考虑到中间的转发服务可能丢弃 `extra_body` 参数，最终的提供模型服务的也可能不是 `vllm`，我们建议优先使用 `response_format` 为 `json_schema` 的方式来获得 LLM 的结构化输出．

总之，我们建议使用 `response_format` 为 `json_schema`．

此外，尽管不是必须的，我们在提示词里明确要求模型返回 JSON 输出，并且描述 JSON 的各个字段如何填充数据，或者直接给出示例输出，往往能够得到更加稳定的结果．

## 总结

要让支持 JSON Schema 的 LLM 稳定地输出结构化的结果，特别是 JSON 输出，我们可以：

1. 编写合适的提示词，要求模型返回 JSON 输出，并且描述输出的各个字段如何填充或者给出示例。
2. 创建合适的 `pydantic.BaseModel`，优先使用 `completion = client.beta.chat.completions.parse` 来发起请求，注意设置 `response_format` 为你的 `pydantic.BaseModel`，然后 `completion.choices[0].message.parsed` 就是返回的满足 `pydantic.BaseModel` 的对象。
3. 处理异常情况，有的模型不能正确返回 JSON 输出，以上调用就会抛出异常。这个时候就需要我们捕获并且处理。

## 参考

1. [openai 文档](https://platform.openai.com/docs/guides/structured-outputs)
2. [vllm 文档](https://docs.vllm.ai/en/stable/features/structured_outputs.html)

---
title: "详解 Dify 的 LLM 节点的结构化输出"
date: 2025-06-01T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - dify
  - json
  - llm
  - 结构化输出
  - 智能体
---

## 前言

dify 1.3.0 新增特性，LLM 节点支持结构化输出了．简单地说，就是 LLM 节点可以输出一个 JSON 对象了，方便后续的节点直接使用．详细见 [1.3.0 版本更新日志](https://github.com/langgenius/dify/releases/tag/1.3.0)．

不过，我们建议使用 1.4.0，因为 dify 1.4.0 修复了 LLM 节点输出结构化变量的 bug．

## 具体如何使用

1. 打开设置->模型供应商->OpenAI-API-compatible->添加模型，按照提示添加一个 LLM 模型．假设我们这里添加的模型为 deepseek-v3，Structured Output 这里可以勾选支持也可以勾选不支持，具体要根据实际模型是否支持 JSON Schema 来选择．甚至对于支持的模型来说，这里也可以勾选不支持．区别只在于 dify 会如何发送请求到 LLM．
2. 新建一个 chatflow，启用 LLM 节点的输出变量->结构化输出，点开输出变量，点击配置，在打开的配置窗口中配置好合适的 JSON Schema．例如，我想要 LLM 帮我生成虚构的人物信息，我的 JSON Schema 如下：

```json
{
  "properties": {
    "name": {
      "title": "Name",
      "type": "string"
    },
    "age": {
      "title": "Age",
      "type": "integer"
    },
    "sex": {
      "title": "Sex",
      "type": "string",
      "enum": ["男", "女", "未知", "其他"]
    }
  },
  "required": ["name", "age", "sex"],
  "title": "Person",
  "type": "object",
  "additionalProperties": false
}
```

如果我们前面添加模型的时候，Structured Output 设置为不支持，这里会有一个黄色感叹号，并且提示当前模型不支持此功能，将会自动降级为提示注入．也就是说，dify 会自动在发送给模型的提示词里注入要求模型输入 JSON 结果的提示词．具体注入的提示词内容可以看 [prompts.py#L231-L334](https://github.com/langgenius/dify/blob/1.4.0/api/core/llm_generator/prompts.py#L231-L334)．我们如果在别的地方有类似的需求，也可以参考这个来写．

3. LLM 的系统提示词请根据需要书写，建议提示词里要明确要求 LLM 返回 JSON 输出，并且给出实例．例如，我这里的提示词为：

````text
```xml
<instruction>
根据用户要求，生成一个虚构人物的信息，并以JSON格式返回．请按照以下步骤完成任务：
1. 随机生成一个虚构人物的姓名、年龄和性别．
2. 姓名可以是中文或英文，年龄范围为1至100岁，性别为“男”或“女”．
3. 确保生成的信息符合逻辑（例如，年龄与姓名风格匹配）．
4. 将生成的信息整理为JSON格式，包含以下字段：name（姓名）、age（年龄）、sex（性别）．
5. 输出时仅返回JSON内容，不要包含任何额外的文本或XML标签．
6. 如果用户未提供具体要求，则完全随机生成；若用户提供了部分信息（如性别），则根据要求补充其他字段．
</instruction>

<examples>
<example>
输入：生成一个女性角色
输出：
```json
{"name": "李梅", "age": 28, "sex": "女"}
```
</example>

<example>
输入：生成一个英文名的年轻男性
输出：
```json
{"name": "John Smith", "age": 22, "sex": "男"}
```
</example>

<example>
输入：生成一个虚构人物，年龄50岁以上
输出：
```json
{"name": "王建国", "age": 65, "sex": "男"}
```
</example>
</examples>

<notes>
1. 所有字段均为必填，不可遗漏．
2. 姓名应避免使用现实中的知名人物或敏感词汇．
3. 年龄需为整数，性别仅限“男”或“女”．
4. 若用户未指定语言，优先使用中文姓名．
5. 输出必须为严格规范的JSON格式，无需额外说明．
</notes>
```
````

经过以上步骤，就已经设置好了．

补充：有的用户可能会注意到 LLM 节点这里点开模型设置，还有回复格式，JSON Schema 的设置．实际上不需要用，一般我们用 LLM 节点设置的输出变量->结构化输出即可．

如果你好奇 dify 是怎么从 LLM 返回的文本中提取到 JSON 结果的，你可以看看 [node.py#L800-L817](https://github.com/langgenius/dify/blob/1.4.0/api/core/workflow/nodes/llm/node.py#L800-L817)．其实非常简单粗暴，先尝试 `json.loads`，失败了就再试试 `json_repair.loads`，不行就是失败了．

**注意**：启用 LLM 节点的输出变量->结构化输出的作用在于让 LLM 节点输出一个变量 `structured_output`，并且它是符合我们指定的 JSON Schema 的．如果没有启用这个，我们依然可以通过修改提示词的方式要求 LLM 返回一个 JSON 输出，然后再用代码节点去从 LLM 返回的文本中解析得到一个 JSON 对象．显然，直接让 LLM 节点给我们返回 JSON 对象更加简单快捷．

### 更多细节

根据代码 [node.py#L574-L579](https://github.com/langgenius/dify/blob/1.4.0/api/core/workflow/nodes/llm/node.py#L574-L579)，dify 首先使用方法 [\_check_model_structured_output_support](https://github.com/langgenius/dify/blob/1.4.0/api/core/workflow/nodes/llm/node.py#L1187-L1211) 检查是否支持结构化输出．根据该方法的定义，如果 LLM 节点没有打开输出变量->结构化输出，则返回 `disabled`，再根据 dify 添加模型的时候 Structured Output 选择的是支持还是不支持返回 `supported` 或者 `unsupported`．也就是说，这个结果的返回值是三种状态之一．

如果是 `disabled`，那就是禁用了结构化输出．后续正常调用 LLM 即可．

如果是 `supported`，则调用 [\_handle_native_json_schema](https://github.com/langgenius/dify/blob/1.4.0/api/core/workflow/nodes/llm/node.py#L1053-L1073) 方法进行来获取模型参数．根据该方法的定义，`response_format` 参数会被设置为 `json_schema`．所以前面说了，在 LLM 节点的设置里开启了输出变量->结构化输出之后，LLM 节点里的模型设置里的回复格式参数设置是无效的，反正都会被覆盖．

如果是 `unsupported`，则调用 [\_set_response_format](https://github.com/langgenius/dify/blob/1.4.0/api/core/workflow/nodes/llm/node.py#L1110-L1122) 方法，他会去设置 `response_format` 参数为 `json` 或者 `json_object`．实际上设置为 `json` 或者 `json_object`结果是一样的，只是不同的模型用的不同的名字．而根据 [\_fetch_prompt_messages](https://github.com/langgenius/dify/blob/1.4.0/api/core/workflow/nodes/llm/node.py#L619-L798) 方法，dify 又会调用 [\_handle_prompt_based_schema](https://github.com/langgenius/dify/blob/1.4.0/api/core/workflow/nodes/llm/node.py#L1075-L1108) 方法，注入要求 LLM 返回 JSON 输出的提示词．

LLM 节点里的模型配置里的回复格式参数，只会在添加到 dify 的模型设置 Structured Output 为支持，同时 LLM 节点设置的输出变量->结构化输出未启用的时候生效．一般来说，我们也很少会去使用．

## 测试

### deepseek-v3 + Structured Output 设置为不支持

输入 `武林高手`，输出：

````json
{
  "text": "```json\n{\"name\": \"张无忌\", \"age\": 25, \"sex\": \"男\"}\n```",
  "usage": {
    "prompt_tokens": 625,
    "prompt_unit_price": "0",
    "prompt_price_unit": "0",
    "prompt_price": "0",
    "completion_tokens": 23,
    "completion_unit_price": "0",
    "completion_price_unit": "0",
    "completion_price": "0",
    "total_tokens": 648,
    "total_price": "0",
    "currency": "USD",
    "latency": 2.391938462969847
  },
  "finish_reason": "stop",
  "structured_output": {
    "name": "张无忌",
    "age": 25,
    "sex": "男"
  },
  "files": []
}
````

可以看到 `structured_output` 按照我们的要求给出了．此外，查询后台记录，我们发现 dify 实际发送给 LLM 的信息为：

````json
[
  {
    "role": "system",
    "content": [
      {
        "text": "You’re a helpful AI assistant. You could answer questions and output in JSON format.\nconstraints:\n    - You must output in JSON format.\n    - Do not output boolean value, use string type instead.\n    - Do not output integer or float value, use number type instead.\neg:\n    Here is the JSON schema:\n    {\"additionalProperties\": false, \"properties\": {\"age\": {\"type\": \"number\"}, \"name\": {\"type\": \"string\"}}, \"required\": [\"name\", \"age\"], \"type\": \"object\"}\n\n    Here is the user's question:\n    My name is John Doe and I am 30 years old.\n\n    output:\n    {\"name\": \"John Doe\", \"age\": 30}\nHere is the JSON schema:\n{\"properties\": {\"name\": {\"title\": \"Name\", \"type\": \"string\"}, \"age\": {\"title\": \"Age\", \"type\": \"integer\"}, \"sex\": {\"title\": \"Sex\", \"type\": \"string\", \"enum\": [\"男\", \"女\", \"未知\", \"其他\"]}}, \"required\": [\"name\", \"age\", \"sex\"], \"title\": \"Person\", \"type\": \"object\", \"additionalProperties\": false}\n\n\n```xml\n<instruction>\n根据用户要求，生成一个虚构人物的信息，并以JSON格式返回．请按照以下步骤完成任务：\n1. 随机生成一个虚构人物的姓名、年龄和性别．\n2. 姓名可以是中文或英文，年龄范围为1至100岁，性别为“男”或“女”．\n3. 确保生成的信息符合逻辑（例如，年龄与姓名风格匹配）．\n4. 将生成的信息整理为JSON格式，包含以下字段：name（姓名）、age（年龄）、sex（性别）．\n5. 输出时仅返回JSON内容，不要包含任何额外的文本或XML标签．\n6. 如果用户未提供具体要求，则完全随机生成；若用户提供了部分信息（如性别），则根据要求补充其他字段．\n</instruction>\n\n<examples>\n<example>\n输入：生成一个女性角色\n输出：\n```json\n{\"name\": \"李梅\", \"age\": 28, \"sex\": \"女\"}\n```\n</example>\n\n<example>\n输入：生成一个英文名的年轻男性\n输出：\n```json\n{\"name\": \"John Smith\", \"age\": 22, \"sex\": \"男\"}\n```\n</example>\n\n<example>\n输入：生成一个虚构人物，年龄50岁以上\n输出：\n```json\n{\"name\": \"王建国\", \"age\": 65, \"sex\": \"男\"}\n```\n</example>\n</examples>\n\n<notes>\n1. 所有字段均为必填，不可遗漏．\n2. 姓名应避免使用现实中的知名人物或敏感词汇．\n3. 年龄需为整数，性别仅限“男”或“女”．\n4. 若用户未指定语言，优先使用中文姓名．\n5. 输出必须为严格规范的JSON格式，无需额外说明．\n</notes>\n```",
        "type": "text"
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "text": "武林高手",
        "type": "text"
      }
    ]
  }
]
````

可以看到，dify 在我们的提示词之前注入了他自己的提示词．

### deepseek-v3 + Structured Output 设置为支持

输入 `武林高手`，输出：

````json
{
  "text": "```json\n{\"name\": \"张无忌\", \"age\": 30, \"sex\": \"男\"}\n```",
  "usage": {
    "prompt_tokens": 985,
    "prompt_unit_price": "0",
    "prompt_price_unit": "0",
    "prompt_price": "0",
    "completion_tokens": 30,
    "completion_unit_price": "0",
    "completion_price_unit": "0",
    "completion_price": "0",
    "total_tokens": 1015,
    "total_price": "0",
    "currency": "USD",
    "latency": 1.5282385630416684
  },
  "finish_reason": "stop",
  "structured_output": {
    "name": "张无忌",
    "age": 30,
    "sex": "男"
  },
  "files": []
}
````

同样查看后台记录，dify 发送给 LLM 的内容为：

````json
[
  {
    "role": "system",
    "content": [
      {
        "text": "```xml\n<instruction>\n根据用户要求，生成一个虚构人物的信息，并以JSON格式返回．请按照以下步骤完成任务：\n1. 随机生成一个虚构人物的姓名、年龄和性别．\n2. 姓名可以是中文或英文，年龄范围为1至100岁，性别为“男”或“女”．\n3. 确保生成的信息符合逻辑（例如，年龄与姓名风格匹配）．\n4. 将生成的信息整理为JSON格式，包含以下字段：name（姓名）、age（年龄）、sex（性别）．\n5. 输出时仅返回JSON内容，不要包含任何额外的文本或XML标签．\n6. 如果用户未提供具体要求，则完全随机生成；若用户提供了部分信息（如性别），则根据要求补充其他字段．\n</instruction>\n\n<examples>\n<example>\n输入：生成一个女性角色\n输出：\n```json\n{\"name\": \"李梅\", \"age\": 28, \"sex\": \"女\"}\n```\n</example>\n\n<example>\n输入：生成一个英文名的年轻男性\n输出：\n```json\n{\"name\": \"John Smith\", \"age\": 22, \"sex\": \"男\"}\n```\n</example>\n\n<example>\n输入：生成一个虚构人物，年龄50岁以上\n输出：\n```json\n{\"name\": \"王建国\", \"age\": 65, \"sex\": \"男\"}\n```\n</example>\n</examples>\n\n<notes>\n1. 所有字段均为必填，不可遗漏．\n2. 姓名应避免使用现实中的知名人物或敏感词汇．\n3. 年龄需为整数，性别仅限“男”或“女”．\n4. 若用户未指定语言，优先使用中文姓名．\n5. 输出必须为严格规范的JSON格式，无需额外说明．\n</notes>\n```",
        "type": "text"
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "text": "武林高手",
        "type": "text"
      }
    ]
  }
]
````

可以看到，LLM 接收到的信息是没有额外的提示词注入的．

## 小结

以上就是 dify 的 LLM 节点的结构化输出的相关内容了．简单地说，我们只需要：

1. 添加一个 LLM 模型，最好是原生支持 JSON Schema．
2. 在 LLM 节点里勾选并设置结构化输出的 JSON Schema．
3. 编写合适的提示词，要求 LLM 返回 JSON 输出．

## 参考

1. [LLM 节点说明](https://docs.dify.ai/zh-hans/guides/workflow/node/llm#json-schema)
2. [结构化输出](https://docs.dify.ai/zh-hans/guides/workflow/structured-outputs)

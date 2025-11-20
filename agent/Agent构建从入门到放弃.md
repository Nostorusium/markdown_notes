# Agent构建从入门到放弃

## 从零开始的MCP

原教旨主义的LLM是一个根据输出给出符合现实语义信息分布输出的封闭的孤立系统，只能依靠已有的知识回答问题，无法获得实时数据，与外部系统交互。

OPEN AI在2023年提出了一个重要的概念: *Function Call*，函数调用。本质上给LLM提供了与外部系统交互的能力，允许他主动调用预设的函数。而开发者想实现Function Call的成本是很高的。

想要实现Function Call，首先模型本身要能稳定的支持Function Call的调用。LLM要能稳定的，以一定格式给出调用function call的输出。在模型训练**微调**时，标准的sharegpt风格的数据集中提供了用于function call训练的特殊字段:

```
[
  {
    "conversations": [
      {
        "from": "human",
        "value": "人类指令"
      },
      {
        "from": "function_call",
        "value": "工具参数"
      },
      {
        "from": "observation",
        "value": "工具结果"
      },
      {
        "from": "gpt",
        "value": "模型回答"
      }
    ],
    "system": "系统提示词（选填）",
    "tools": "工具描述（选填）"
  }
]
```

这也意味着只有经过特殊function call微调的模型才能稳定支持这种能力。

另一方面，function call提出伊始并没有想让它成为一项标准。尽管后续很多模型也支持了function call的调用，但其实现方式各不相同。这意味着想要开发一个function call工具，需要为不同的模型做适配。比如:参数格式，触发逻辑，返回结构等。

### Model Context 


## Prompt最佳实践

## 上下文管理
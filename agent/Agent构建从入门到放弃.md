# Agent构建从入门到放弃

本指南旨在提供一个自底向上的Agent构建指南，给出必要的理论内容，以及需要补充的网络编程知识、python语法等技术细节，并最终导向可实现性强的代码实践。

## 从零开始的MCP

原教旨主义的LLM是一个根据输入给出符合现实语义信息分布输出的封闭的孤立系统，只能依靠已有的知识回答问题，无法获得实时数据，与外部系统交互。

OPEN AI在2023年提出了一个重要的概念: *Function Call*，函数调用。本质上给LLM提供了与外部系统交互的能力，允许他主动调用预设的函数。而开发者想实现Function Call的成本很高。

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

### Model Context Protocol

**MCP**，模型上下文协议是由 *Anthropic* 公司(开发了Claude)提出的标准协议，以解决AI模型与外部数据源、工具交互的难题。MCP作为一个通用插头，制定了统一的规范，让模型与外部工具的交互更加标准化、可复用。

MCP采用 **client-host-server** 架构。每个host可以运行多个client实例。信息格式与交互基于JSON-RPC构建，提供一种有状态的会话协议，专注于上下文交换，clients与servers之间的协调。

1. Host
   充当协调者，用于管理client实例，控制client的连接、生命周期，管理client的上下文聚合。
2. Client
   client由host创建，并保持一个独立的server连接。负责与server建立有状态的会话，处理通信等。client与server有一对一关系。
3. Server
   server提供专业的上下文和能力。通过MCP原语暴露resources,tools,prompts。通过client提供的接口请求sampling。可以是本地进程，也可以是远程服务。

Agent负责协调LLM与MCP Client。而对于MCP Server，我们可以认为它是一个远端的存在，使用固定通信模式与Agent维护的Client交互，哪怕它存在于本地上。

Agent所关注的只有MCP Client和LLM的协调交互等，而MCP Server的实现应当独立运作。

### JSON-RPC协议

MCP协议就好比是一个为LLM服务的应用层的**网络协议**。它采用的**JSON-RPC**协议是一个基于JSON格式的轻量级**远程过程调用**(RPC)协议，基于请求与响应模型。它规定了双方通信的格式与交互流程。

request与response均为JSON格式，由客户端发起，服务器处理后返回结果。应包含以下字段:

1. jsonrpc 协议版本
2. method 调用的远程方法名称
3. params 方法参数
4. id 用于匹配请求和响应。为null表示通知

```
request = {
   "jsonrpc": "2.0",
   "method": "subtract",
   "params": [42, 23],
   "id": 1
}
response = requests.post("http://example.com/jsonrpc", json=request)
print(response.json())
```

JSON-RPC协议是一种**消息格式规范**与**交互流程约定**，并不规定传输方式，比如，在HTTP报文的body中携带基于JSON-RPC的JSON文件是一个常见的做法。

JSON-RPC 2.0规范定义了三种基本类型的消息:

1. Requests
   需包含唯一的id和方法名称
2. Responses
   需包含与request对应的id
3. Notifications
   不包含id,认为是通知。

双方的通信流程基于一应一答的request与response。以client向server发送请求为例:

```
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {
    "cursor": "optional-cursor-value"
  }
}
```

server对其响应为:

```
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "Get current weather information for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name or zip code"
            }
          },
          "required": ["location"]
        }
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

client与server的id应对应。client在method中写明希望调用的tool,server负责调用该tool，并构建该response返回，返回server支持的tool:get_weather。

### MCP Server

MCP Server的功能划分如下。

1. Tools
   Server暴露可执行功能，供LLM调用
2. Resources
   Server暴露数据和内容供client读取并作为LLM的上下文
3. Prompts
   Server定义可复用的prompt，引导LLM交互

与MCP Server的交互全部遵循JSON-RPC协议。

### MCP Client

除了基本的连接通信功能之外，Client可以给服务器提供额外的功能：

1. Sampling
   允许server反向请求Host的LLM的能力，让Server也能使用AI
2. Roots
   client提供给server指定的一些地址，告诉服务器应该关注那些资源。
3. Elicitation
   允许server向user发送请求，来获得更多信息
4. Logging
   允许server向client发送日志信息

### MCP Host

在MCP的架构中，Host负责client的实现、LLM的集成、会话的管理等等。LLM仅仅是Host的一个组成部分。类似于 *Claude Desktop* 的应用是一个完整的Host实现。

在仅拥有一个LLM api的前提下，我们仍需要手写一个完整的Host并完成：

1. Client实现
   比如，它应如何与server建立连接
2. Rresponse处理
   如何处理接收到的响应?
3. LLM集成
   基于api，或是本地部署?
4. 上下文历史管理
   对context的维护，比如**滑动窗口**
5. 工作流安排
   比如，何时调用tool
6. 其他
   你喜欢的其他功能

在这个意义上，不依赖如 *Claude Desktop* 的应用从零实现一个Host(Agent)是一个复杂的工作。

### 我们应当如何看待MCP

MCP有两个层级，数据层和传输层。
在数据层面，我们采用JSON-RPC，规定了client和server交互的信息结构和语义。
1. 生命周期管理
   如何初始化连接，如何关闭。
2. Server功能
   让Server提供必要的tool给AI使用、暴露必要的数据、提供必要的prompt
3. Client功能
   允许Server向Client反向交互，请求更多用户信息，或者调用LLM等。
4. 其他实用的功能
   比如利用notification实现实时更新等。

在传输层面管理client与server之间的沟通信道。
1. stdio传输
   使用标准I/O，与本地的进程直接通信。
2. streamable HTTP 传输
   使用HTTP POST交互方式

## 异步网络通信与上下文管理

在正式编写代码之前，我们需要补充一些开发知识。
由于Agent归根结底需要网络通信，为了高效的处理I/O密集型操作，我们需要学习编写**异步I/O代码**以及一些程序细节。

### 协程 Coroutine

协程可以视为一个特殊的函数，可在执行过程中暂停，并稍后恢复执行。使用async def关键字定义。

```
import asyncio
async def hello_world():
    print("hello")
    await asyncio.sleep(1)
    print("world!)
```

使用 await 关键字暂停执行，等待异步操作的完成。

---

asyncio中，使用**事件循环**(event loop)调度和执行协程。循环中的协程时而阻塞，时而被执行。它不停地查询是否有任务要执行，并在任务完成后调用回调函数。这也是一个典中典的网络通信情境下对事件机制的应用。

```
async def main():
    task = asyncio.create_task(hello_world()) # 注册回调函数为hello_world()
    await task
asyncio.run(main())
```

asyncio.run()会创建一个事件循环，并运行指定的协程。

### 上下文管理协议

在介绍上下文管理器之前，先引入python的一个语法:with

```
file = open('file.txt')
for line in f:
  # do something
f.close()
```
这是一个打开文件的例子。这个写法存在一个问题：在打开文件后，对读取到的内容进行操作的过程中如果发生异常，会导致**文件句柄**(File Handle,操作系统管理该文件的一个数据结构,包含fd等)无法被正常释放，造成资源泄露。

> 说白了就是没close
> 不要以为用了python就不用管理资源泄露了
> ~~内存管理是跑不了的~~

于是我们引入异常管理:

```

file = open('file.txt')
try:
   for line in f:
       # do something
finally:
   f.close()
```

但是这样写很繁琐。于是with语法诞生了:

```
with open('file.txt') as file:
  for line in file:
    # do something
```

with语法块可以完成相同的功能。

---

```
with context_expression [as target(s)]:
    with-body
```

with的语法十分简单，只需要with表达式就可以执行自定义的业务逻辑。想要使用with，with后的对象需要实现**上下文管理器协议**。

在python中，只要一个类实现了以下方法，就实现了上下文管理器协议:
- \_\_enter\_\_
  在进入with语法块之前调用,返回值赋给with的target
- \_\_exit\_\_
  在退出with语法块(**作用域**)、发生异常、return等情况时调用。等价于finally，能保证__exit__的执行

一个file可能类似于这样:

```
# 伪代码
class File:
  def __enter__(self):
    return file_obj
  
  # exc是exception
  def __exit__(self,exc_type,exc_value,exc_tb):
    file_obj.close()
    # 发生了exception异常!
    if exc_type is not None:
      raise exception
```

因此with非常适合需要对上下文处理的情景，比如**文件操作/socket通信**等，它们都需要在执行完业务逻辑后释放资源。就好比C++的网络代码中塞满了shared_ptr智能指针，python中也到处充斥着with.

> 我们可以认为with的意思是:with 这个类定义的操作,执行以下操作
> 又或者参考C++中RAII的概念。
> __enter__就好比是构造函数 __exit__就好比是析构函数
> 上下文管理确保开头和结尾一定能执行必要的操作

python标准库提供的contextlib可以进一步简化我们的代码。该模块允许以装饰器的方式进行上下文管理。

```
from contextlib import contextmanager
@contextmanager
def test():
    print('before')
    yield 'hello'
    print('after')

with test() as t:
    print(t)
```

其输出以yield为界:

```
# Output:
# before
# hello
# after
```

### Exit Stack

contextlib提供了一个用于管理上下文的容器:exit stack

```
with open('book1.txt') as book1:
  with open('book2.txt') as book2:
    # read book1
    # read book2
# exit...
```

当我们一次性有多个操作需要上下文管理，为了避免上述这种套娃带来的繁琐，我们提出exit stack作为一个栈类型的容器。

```
from contextlib import ExitStack
file_list = ['a.txt','b.txt']

with ExitStack() as stack:
  files = [stack.enter_context(open(f)) for file in file_list]

  for file in files:
    print(file.read())
# exit
```

1. enter_context()
   将上下文管理器加入栈结构。
2. callback(func,*args,**kwargs)
   给普通函数也注册清理功能，在退出时按栈后进先出执行

其核心思想是将多个操作收集到一个列表，退出时按栈逻辑倒序执行，以模拟嵌套with的行为。

其底层实现为:维护一个函数列表作为栈使用。
1. enter_context时调用__enter__，将__exit__压入栈
2. callback时将指定函数包装并压入栈

---

上下文管理器功能提供了在进入和退出作用域时执行的操作，这需要使用with语法。因为一个with语法块就好比是一个**局部的**、作用在栈内存的**作用域**，很容易类比到C++中的RAII思想。

而Stack作为容器，我们可以将许多支持上下文管理器的对象加入其中，并利用with的作用域功能在退出时倒序出栈逐一释放资源。

在不使用with的情况下，我们仍然可以手动管理ExitStack以更灵活的管理对象的生命周期，但需要保证ExitStack.close()能够最终执行。


### AsyncExitStack

ExitStack是同步的，AsyncExitStack是它的异步版本。

```
async with AsyncExitStack() as stack:
    conn = await stack.enter_async_context(async_database_connect())
```

enter_context则改名为enter_async_context
简单来说，它只是在ExitStack的基础上加入了async/await支持。

## MCP Server

MCP Server的**主要**功能如下。

### Resources

```
@mcp.resource("file://documents/{name}")
def read_document(name: str) -> str:
    """Read a document by name."""
    # This would normally read from disk
    return f"Content of {name}"
```

resource的使用类似于REST API中的GET，目的是暴露服务器可用的数据。它不应当承担运算功能，也不应该产生任何副作用。它由应用本身控制，而不由LLM控制。我们使用url来确定一个资源。

### Tools

```
@mcp.tool()
def sum(a: int, b: int) -> int:
    """Add two numbers together."""
    return a + b
```

tools用于让LLM做出决策。不同于resources，tools应当承担运算功能，并伴随副作用。

```
{
  name: "searchFlights",
  description: "Search for available flights",
  inputSchema: {
    type: "object",
    properties: {
      origin: { type: "string", description: "Departure city" },
      destination: { type: "string", description: "Arrival city" },
      date: { type: "string", format: "date", description: "Travel date" }
    },
    required: ["origin", "destination", "date"]
  }
}
```

MCP使用JSON Schema定义一个tool，每个tool应当拥有清晰的输入和输出。
协议本身提供了两个操作：
1. tools/list
   列出可用tool
2. tools/call
   调用某个tool

### Prompts

prompts主要用于定义结构化的输入和交互模式。

```
{
  "name": "plan-vacation",
  "title": "Plan a vacation",
  "description": "Guide through vacation planning process",
  "arguments": [
    { "name": "destination", "type": "string", "required": true },
    { "name": "duration", "type": "number", "description": "days" },
    { "name": "budget", "type": "number", "required": false },
    { "name": "interests", "type": "array", "items": { "type": "string" } }
  ]
}
```

比如这是一个"做一个假期计划"的prompt。比起非结构化的自然语言输入，prompt系统允许选择"做一个假期计划"作为模板并以该模式交互。

协议规定了两个操作:
1. prompts/list
   用于发现可用的prompt
2. prompts/get
   用于获得prompt

prompt的使用由用户控制，需要显式调用。


## MCP Client

### 官方参考架构

```
class MCPClient:
  def __init__(self):
    self.session:Optional[ClientSession] = None
    self.exit_stack = AsyncExitStack()
    self.anthropic = Anthropic()
```

一个MCP Client需要维护与MCP Server连接的session，使用AsyncExitStack资源管理

```
# 以.py为例
async def connect_to_server(self,server_script_path:str):
  command = "python"

  # 建立连接需要写params
  server_params = StdioServerParameters(
    command = command,
    args = [server_script_path],
    env = None
  )

  # 实例化standard I/O 方式的client，并加入上下文
  stdio_transport = await self.exit_stack.enter_async_context(stdio_client(server_params))

  # 
  self.stdio,self.write = stdio_transport
  self.session = await self.exit_stack.enter_async_context(ClientSession(self.stdio, self.write))
```

## Host

## Prompt最佳实践

## 上下文管理
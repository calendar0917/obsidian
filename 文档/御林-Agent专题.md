---
title: "御林招新题：Agent 专题"
subtitle: "御林招新题：Agent 专题"
summary: "学习搭建 Agent"
description: "学习搭建 Agent"
image: ""
date: 2025-10-21
lastmod: 2025-10-21
draft: false
hiddenFromHomePage: True
toc:
 enable: true
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

## **实现工具调用（Tool Calling）**

- **设计工具**：你必须实现至少**两个**工具。当用户提出相关需求时，你的 Agent 必须能够识别并调用相应的工具。

  > qwen 的工具调用整合了 OpenAI，所以可以看 OpenAI 的使用文档：[函数调用 - OpenAI 中文文档](https://www.openaicto.com/capabilities/function-calling)，写得比较简略；[openai-python/api.md at main · openai/openai-python](https://github.com/openai/openai-python/blob/main/api.md) 仓库，详细但是冗杂
  >
  > 基本步骤：
  >
  > 1. 创建助手，以调用外部 API 来回答问题（定义函数）
  > 2. 将自然语言转换为 API 调用
  > 3. 从文本中提取结构化的数据

  - **工具1：天气查询工具**

    > 可以使用爬虫实现，也可以寻找相应的`api`提供商

    - **功能**：接受城市名作为参数。
    - **返回**：指定城市的实时天气信息，包括温度、天气状况（晴、多云、雨等）和湿度。
    - **用户交互示例**：当用户问“北京今天天气怎么样？”时，Agent 必须识别出“北京”并调用此工具，然后返回天气信息。

  > 注册接口：[天气预报查询接口_天聚地合](https://www.juhe.cn/docs/api/id/73)

  ```py
  WEATHER_API_KEY = "..."
  def get_weather(city: str) -> str:
      url = f"http://apis.juhe.cn/simpleWeather/query?city={city}&key={WEATHER_API_KEY}"
      response = requests.get(url)
      data = response.json()
      if data.get("reason") == "查询成功!":
          result = data["result"]
          return (f"{city}当前天气：{result['realtime']['info']}，"
                  f"温度：{result['realtime']['temperature']}℃，"
                  f"湿度：{result['realtime']['humidity']}%")
      return f"查询天气失败：{data.get('reason', '未知错误')}"
  ```

  - **工具2：网络搜索工具**

    > 可以使用Searxng，也可以使用其它专业的`api`

    - **功能**：接受关键词作为参数。
    - **返回**：基于关键词的网络搜索结果摘要。
    - **用户交互示例**：当用户问“2024 年奥运会在哪里举办？”时，Agent 能够使用此工具进行搜索并给出答案。

  > 试了自己的 searxng，可能是格式问题跑不通，换了个 API

  ```python
  def web_search(query: str) -> str:
      # DuckDuckGo 免费搜索 API（无需注册，直接使用）
      url = "https://api.duckduckgo.com/"
      params = {
          "q": query,
          "format": "json",
          "no_html": 1,  # 不返回 HTML 内容
          "no_redirect": 1  # 不返回跳转链接
      }
      try:
          response = requests.get(url, params=params, timeout=10)
          response.raise_for_status()  # 检查 HTTP 状态码（如 404、500 会报错）
      except requests.exceptions.RequestException as e:
          return f"搜索请求失败：{str(e)}"
      
      try:
          data = response.json()
      except requests.exceptions.JSONDecodeError:
          return "搜索结果解析失败：返回内容不是有效的 JSON"
      
      # 提取结果（DuckDuckGo 的 JSON 结构与 SearXNG 不同）
      summary = []
      # 1. 优先取 "Abstract"（摘要）
      if data.get("Abstract"):
          summary.append(f"摘要：{data['Abstract']}")
      # 2. 再取 "RelatedTopics"（相关主题）的前 2 条
      for topic in data.get("RelatedTopics", [])[:2]:
          if "Text" in topic:
              summary.append(f"- {topic['Text']}")
      
      if not summary:
          return "未找到相关搜索结果"
      return "\n".join(summary)
  ```

## **实现记忆功能（Memory）**

- **功能**：你的 Agent 必须能够记住之前的对话内容，并在后续的交互中利用这些信息。

- **用户交互示例**：
  - 用户：“今天上海天气怎么样？”
  - Agent：“上海今天晴，气温 25 度。”
  - 用户：“那明天呢？”
  - Agent 必须能够理解“那明天呢？”是关于“上海天气”的追问，并给出正确的答案。

> 记忆管理：用 `message[]` 存储以前对话的信息

```python
messages = []
While True:
    ...
    messages.append({"role": "user", "content": user_input})
    # 调用的时候再传入 messages
    ...
    # 调用结束，结果再传入 messages
    messages.append({
        "role": "assistant",
        "content": response_message.content,
        "tool_calls": [
            {
                "id": tc.id,
                "type": tc.type,
                "function": {
                    "name": tc.function.name,
                    "arguments": tc.function.arguments
                }
            } for tc in response_message.tool_calls
        ] if response_message.tool_calls else None
    })
```

### 完整示例代码

将工具调用、记忆整合：

```python
import os
import requests
from bs4 import BeautifulSoup
from openai import OpenAI

# 初始化 OpenAI 客户端
client = OpenAI(
    api_key="...",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

# 天气查询工具
WEATHER_API_KEY = "..."  # 需注册聚合数据等平台获取

def get_weather(city: str) -> str:
    url = f"http://apis.juhe.cn/simpleWeather/query?city={city}&key={WEATHER_API_KEY}"
    response = requests.get(url)
    data = response.json()
    if data.get("reason") == "查询成功!":
        result = data["result"]
        return (f"{city}当前天气：{result['realtime']['info']}，"
                f"温度：{result['realtime']['temperature']}℃，"
                f"湿度：{result['realtime']['humidity']}%")
    return f"查询天气失败：{data.get('reason', '未知错误')}"

# 网络搜索工具
# SEARXNG_URL = "http://8.137.38.223:8081/"  # 用不了
def web_search(query: str) -> str:
    # DuckDuckGo 免费搜索 API（无需注册，直接使用）
    url = "https://api.duckduckgo.com/"
    params = {
        "q": query,
        "format": "json",
        "no_html": 1,  # 不返回 HTML 内容
        "no_redirect": 1  # 不返回跳转链接
    }
    try:
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()  # 检查 HTTP 状态码（如 404、500 会报错）
    except requests.exceptions.RequestException as e:
        return f"搜索请求失败：{str(e)}"
    
    try:
        data = response.json()
    except requests.exceptions.JSONDecodeError:
        return "搜索结果解析失败：返回内容不是有效的 JSON"
    
    # 提取结果（DuckDuckGo 的 JSON 结构与 SearXNG 不同）
    summary = []
    # 1. 优先取 "Abstract"（摘要）
    if data.get("Abstract"):
        summary.append(f"摘要：{data['Abstract']}")
    # 2. 再取 "RelatedTopics"（相关主题）的前 2 条
    for topic in data.get("RelatedTopics", [])[:2]:
        if "Text" in topic:
            summary.append(f"- {topic['Text']}")
    
    if not summary:
        return "未找到相关搜索结果"
    return "\n".join(summary)

#核心，调用 AI
# 初始化对话记忆（上下文）
messages = []

while True:
    user_input = input("\n你：")
    if user_input in ["exit", "quit"]:
        print("AI：拜拜~")
        break
    
    # 将用户输入加入记忆
    messages.append({"role": "user", "content": user_input})
    
    # 调用模型生成响应（可能包含工具调用）
    response = client.chat.completions.create(
        model="qwen-plus",
        messages=messages,
        tools=[
            {
                "type": "function",
                "function": {
                    "name": "get_weather",
                    "description": "查询指定城市的实时天气",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "city": {"type": "string", "description": "城市名称，如北京、上海"}
                        },
                        "required": ["city"]
                    }
                }
            },
            {
                "type": "function",
                "function": {
                    "name": "web_search",
                    "description": "网络搜索关键词相关信息",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "query": {"type": "string", "description": "搜索关键词"}
                        },
                        "required": ["query"]
                    }
                }
            }
        ],
        tool_choice="auto", # 让 AI 自己决定是否调用 tools
    )

    response_message = response.choices[0].message

    # 将模型的响应消息（含 tool_calls）加入 messages
    messages.append({
        "role": "assistant",
        "content": response_message.content,
        "tool_calls": [
            {
                "id": tc.id,
                "type": tc.type,
                "function": {
                    "name": tc.function.name,
                    "arguments": tc.function.arguments
                }
            } for tc in response_message.tool_calls
        ] if response_message.tool_calls else None
    })

    # 处理工具调用
    if response_message.tool_calls:
        for tool_call in response_message.tool_calls:
            function_name = tool_call.function.name
            function_args = eval(tool_call.function.arguments)

            # 调用工具
            if function_name == "get_weather":
                tool_result = get_weather(**function_args)
            elif function_name == "web_search":
                tool_result = web_search(** function_args)
            else:
                tool_result = f"未知工具：{function_name}"

            # 工具结果加入 messages（确保 tool_call_id 正确）
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,  # 对应前置 tool_calls 的 id
                "name": function_name,
                "content": tool_result
            })

        # 生成最终回答
        final_response = client.chat.completions.create(
            model="qwen-plus",
            messages=messages,
        )
        print(f"AI：{final_response.choices[0].message.content}")
    else:
        # 无工具调用，直接输出
        print(f"AI：{response_message.content}")
```

效果：

![image-20251021092143661](https://raw.githubusercontent.com/calendar0917/images/master/image-20251021092143661.png)

## **多 Agent 协同工作**

- **场景**：设计至少两个不同的 Agent 角色，让它们在完成一个共同任务时能够互相协作。

> 参考 openai-sdk 文档：[编排多个 Agents - OpenAI Agents SDK - 中文文档](https://openai-agents-sdk.doczh.com/multi_agent/)
>
> 实现要点：
>
> 1. 需要设计一个**消息总线或队列**，让 Agent A 能把任务指令发给 Agent B，Agent B 也能把结果回传给 Agent A。
> 2. 每个 Agent 需要有**角色专属的 Prompt（提示词）**，让其记住自己的分工
> 3. 工具需要有**统一的调用接口**

- **Agent 角色设计**：
  - **Agent A（规划者）**：负责分解任务，将任务分配给其他 Agent，并综合最终结果。
  - **Agent B（执行者）**：专门负责调用工具和执行具体操作。
- **协作流程示例**：
  - 用户：“请帮我查一下，北京明天的天气，然后告诉我这个城市都有哪些著名的 IT 公司。”
  - **Agent A（规划者）**接收请求，将其分解为两个子任务：
    - 子任务1：查询北京明天的天气。
    - 子任务2：查询北京的著名 IT 公司。
  - **Agent A** 将子任务1分配给**Agent B（执行者）**，要求其调用“天气查询工具”。
  - **Agent A** 再次将子任务2分配给**Agent B**，要求其调用“网络搜索工具”。
  - **Agent A** 收到两个子任务的执行结果后，将它们整合，并以流畅的语言回答用户。

> 与平时的代码不同，用 Agent 需要把部分代码功能交给 Ai 去处理

### 完整示例代码

```py
import os
import json
import requests
from openai import OpenAI

# 初始化 OpenAI 客户端
client = OpenAI(
    api_key="...",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

# 工具定义
WEATHER_API_KEY = "..."

def get_weather(city: str) -> str:
    """查询指定城市的实时天气"""
    url = f"http://apis.juhe.cn/simpleWeather/query?city={city}&key={WEATHER_API_KEY}"
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
    except requests.exceptions.RequestException as e:
        return f"天气查询请求失败：{str(e)}"
    except json.JSONDecodeError:
        return "天气查询结果解析失败：返回内容不是有效的JSON"

    if data.get("reason") == "查询成功!":
        result = data["result"]
        return (f"{city}当前天气：{result['realtime']['info']}，"
                f"温度：{result['realtime']['temperature']}℃，"
                f"湿度：{result['realtime']['humidity']}%")
    return f"查询天气失败：{data.get('reason', '未知错误')}"

def web_search(query: str) -> str:
    """使用DuckDuckGo进行网络搜索"""
    url = "https://api.duckduckgo.com/"
    params = {
        "q": query,
        "format": "json",
        "no_html": 1,
        "no_redirect": 1
    }
    try:
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()
    except requests.exceptions.RequestException as e:
        return f"搜索请求失败：{str(e)}"
    
    try:
        data = response.json()
    except json.JSONDecodeError:
        return "搜索结果解析失败：返回内容不是有效的JSON"
    
    summary = []
    if data.get("Abstract"):
        summary.append(f"摘要：{data['Abstract']}")
    for topic in data.get("RelatedTopics", [])[:2]:
        if "Text" in topic:
            summary.append(f"- {topic['Text']}")
    return "\n".join(summary) if summary else "未找到相关搜索结果"

# Agent 角色定义（关键修改：明确Agent B只返回结果）
AGENT_A_PROMPT = """你是 Agent A，任务规划者。职责：
1. 分析用户请求，拆解为需要执行的子任务（每个子任务对应一个工具调用）
2. 为每个子任务指定需要调用的工具（get_weather 或 web_search）和参数
3. 等待所有子任务执行完成后，整合结果并以自然语言回复用户

重要规则：
- 当需要工具调用时，必须生成明确的 tool_call
- 不要自己执行工具调用
- 工具调用结果会通过 tool_response 反馈给你"""

AGENT_B_PROMPT = """你是 Agent B，任务执行者。职责：
1. 接收 Agent A 的子任务指令
2. 严格按指令调用指定工具（get_weather 或 web_search）
3. 仅返回工具调用的原始结果，不要添加任何解释或格式化

重要规则：
- 你不需要生成 tool_call
- 直接返回工具函数的执行结果字符串
- 结果必须是纯文本，不要包含JSON或其他结构"""

# 初始化Agent A的对话记忆（关键：保留完整上下文）
messages_a = [{"role": "system", "content": AGENT_A_PROMPT}]

# 工具列表定义
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询城市实时天气",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "网络搜索关键词",
            "parameters": {
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"]
            }
        }
    }
]

def main():
    # Agent B 的上下文只需初始化一次
    messages_b = [{"role": "system", "content": AGENT_B_PROMPT}]
    
    while True:
        user_input = input("\n你：")
        if user_input.lower() in ["exit", "quit", "退出"]:
            print("AI：拜拜~")
            break
        
        # 将用户输入加入Agent A的上下文
        messages_a.append({"role": "user", "content": user_input})
        
        # Agent A：拆解任务并生成工具调用计划
        while True:  # 循环处理多轮工具调用
            agent_a_response = client.chat.completions.create(
                model="qwen-plus",
                messages=messages_a,
                tools=tools,
                tool_choice="auto"
            )
            agent_a_msg = agent_a_response.choices[0].message
            messages_a.append(agent_a_msg)
            
            # 检查是否需要工具调用
            if not agent_a_msg.tool_calls:
                # 无需工具调用，直接输出结果
                print(f"AI：{agent_a_msg.content}")
                break
            
            # 处理所有工具调用
            tool_results = []
            for tool_call in agent_a_msg.tool_calls:
                func_name = tool_call.function.name
                try:
                    func_args = json.loads(tool_call.function.arguments)
                except json.JSONDecodeError:
                    tool_results.append(f"工具调用参数解析失败：{tool_call.function.arguments}")
                    continue
                
                # Agent B 执行工具调用（关键：直接调用工具函数）
                print(f"正在执行: {func_name}({func_args})...")
                if func_name == "get_weather":
                    result = get_weather(**func_args)
                elif func_name == "web_search":
                    result = web_search(**func_args)
                else:
                    result = f"未知工具：{func_name}"
                
                tool_results.append(result)
                
                # 将结果反馈给 Agent A
                messages_a.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "name": func_name,
                    "content": result
                })
            
            # 清空 Agent B 的上下文（仅保留系统提示）
            messages_b = [messages_b[0]]

        # 保存最终回复到上下文
        if agent_a_msg.content:
            messages_a.append({"role": "assistant", "content": agent_a_msg.content})

if __name__ == "__main__":
    main()

```



![image-20251021102802835](https://raw.githubusercontent.com/calendar0917/images/master/image-20251021102802835.png)

> 不知道为什么搜索又出了问题

## **实现任意形式的知识库**

- **功能**：构建一个外部知识库（可以是简单的文本文件、Markdown 文档，甚至一个小型数据库）。
- **集成**：将知识库集成到你的 Agent 系统中，使其能够通过检索知识库来回答问题。
- **用户交互示例**：
  - 用户：“什么是 DevOps？”
  - Agent 能够通过检索你提供的知识库，而不是纯粹依赖 LLM 的通用知识，来回答这个问题。这展示了 Agent 能够利用私有数据或特定领域数据来提供更精确和定制化的答案。

> 实现：
>
> 1. 需要用到预训练模型（这里用`paraphrase-multilingual-MiniLM-L12-v2`）来识别语义，将语义相近的文本转化为数值相近的向量
>    - 导入 SentenceTransformer 模型
>
> 2. 需要手动处理模型，将文本处理成可理解的嵌入向量
> 3. 还有用向量计算内容相关性等知识……不太懂
>
> ```python
> # 大致的关键代码
> def __init__(self, kb_path="knowledge_base.md"):
>     self.kb_path = kb_path  # 知识库文件路径
>     self.model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')  # 加载预训练模型
>     self.entries = []  # 存储解析后的知识库条目（标题+文本）
>     self.embeddings = []  # 存储文本的嵌入向量（计算机可理解的“数字表示”）
>     self.load_and_process()  # 启动加载与处理流程
> 
> # 还需要手动处理 entry、text 等内容
> def load_and_process(self):
>     # 检查文件是否存在，不存在则创建示例知识库
>     if not os.path.exists(self.kb_path):
>         self.create_sample_kb()
>     
>     # 读取文件内容
>     with open(self.kb_path, 'r', encoding='utf-8') as f:
>         content = f.read()
>     
>     # 按Markdown标题分割条目（核心解析逻辑）
>     entries = re.split(r'\n# ', content)  # 用“换行+# ”分割内容（Markdown一级标题格式）
>     if entries and entries[0].strip() == '':  # 处理开头可能的空字符串
>         entries = entries[1:]
>     
>     # 提取每个条目的标题和文本，存入self.entries
>     for entry in entries:
>         lines = entry.split('\n', 1)  # 按第一个换行分割（标题和正文分离）
>         title = lines[0].strip()  # 标题（如“DevOps”）
>         text = lines[1].strip() if len(lines) > 1 else ""  # 正文内容
>         self.entries.append({"title": title, "text": text})
>         
> # 生成嵌入
> if self.entries:
>     # 将每个条目的“标题+文本”拼接成完整字符串
>     texts = [f"{e['title']}: {e['text']}" for e in self.entries]
>     # 用模型将文本转化为向量（嵌入），存储到 self.embeddings
>     self.embeddings = self.model.encode(texts, convert_to_tensor=True)
> ```

### 完整示例代码

```python
import os
import json
import requests
import re
from openai import OpenAI
from sentence_transformers import SentenceTransformer
import torch

# 初始化 OpenAI 客户端
client = OpenAI(
    api_key="......", 
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

# 工具定义
WEATHER_API_KEY = "......"

def get_weather(city: str) -> str:
    """查询指定城市的实时天气"""
    url = f"http://apis.juhe.cn/simpleWeather/query?city={city}&key={WEATHER_API_KEY}"
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
    except requests.exceptions.RequestException as e:
        return f"天气查询请求失败：{str(e)}"
    except json.JSONDecodeError:
        return "天气查询结果解析失败：返回内容不是有效的JSON"

    if data.get("reason") == "查询成功!":
        result = data["result"]
        return (f"{city}当前天气：{result['realtime']['info']}，"
                f"温度：{result['realtime']['temperature']}℃，"
                f"湿度：{result['realtime']['humidity']}%")
    return f"查询天气失败：{data.get('reason', '未知错误')}"

def web_search(query: str) -> str:
    """使用DuckDuckGo进行网络搜索"""
    url = "https://api.duckduckgo.com/"
    params = {
        "q": query,
        "format": "json",
        "no_html": 1,
        "no_redirect": 1
    }
    try:
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()
    except requests.exceptions.RequestException as e:
        return f"搜索请求失败：{str(e)}"
    
    try:
        data = response.json()
    except json.JSONDecodeError:
        return "搜索结果解析失败：返回内容不是有效的JSON"
    
    summary = []
    if data.get("Abstract"):
        summary.append(f"摘要：{data['Abstract']}")
    for topic in data.get("RelatedTopics", [])[:2]:
        if "Text" in topic:
            summary.append(f"- {topic['Text']}")
    return "\n".join(summary) if summary else "未找到相关搜索结果"

class KnowledgeBase:
    """简单的知识库系统，支持语义搜索"""
    
    def __init__(self, kb_path="knowledge_base.md"):
        self.kb_path = kb_path
        self.model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')
        self.entries = []
        self.embeddings = None  # 初始化为None
        self.load_and_process()
    
    def load_and_process(self):
        """加载知识库文件并生成嵌入向量"""
        if not os.path.exists(self.kb_path):
            # 创建示例知识库
            self.create_sample_kb()
        
        with open(self.kb_path, 'r', encoding='utf-8') as f:
            content = f.read()
        
        # 按Markdown标题分割条目
        entries = re.split(r'\n# ', content)
        if entries and entries[0].strip() == '':
            entries = entries[1:]
        
        self.entries = []
        for entry in entries:
            lines = entry.split('\n', 1)
            title = lines[0].strip()
            text = lines[1].strip() if len(lines) > 1 else ""
            self.entries.append({"title": title, "text": text})
        
        # 生成嵌入
        if self.entries:
            texts = [f"{e['title']}: {e['text']}" for e in self.entries]
            self.embeddings = self.model.encode(texts, convert_to_tensor=True)
        else:
            self.embeddings = None
    
    def create_sample_kb(self):
        """创建示例知识库，不存在时调用"""
        sample_kb = """# DevOps
DevOps 是什么呢？
"""
        with open(self.kb_path, 'w', encoding='utf-8') as f:
            f.write(sample_kb)
        print(f"已创建示例知识库文件: {self.kb_path}")
    
    def search(self, query, top_k=2):
        """搜索知识库，返回最相关的条目"""
        if not self.entries or self.embeddings is None:
            return [{"title": "知识库为空", "text": "没有可用的知识库条目", "score": 0.0}]
            
        query_embedding = self.model.encode([query], convert_to_tensor=True)
        
        # 计算余弦相似度
        cos_scores = torch.nn.functional.cosine_similarity(
            query_embedding, 
            self.embeddings
        )
        
        # 获取top_k结果
        top_results = torch.topk(cos_scores, k=min(top_k, len(cos_scores)))
        
        results = []
        for idx, score in zip(top_results.indices, top_results.values):
            if score > 0.3:  # 设置相似度阈值
                results.append({
                    "title": self.entries[idx]["title"],
                    "text": self.entries[idx]["text"],
                    "score": float(score)
                })
        
        return results

# 初始化知识库
kb = KnowledgeBase()

def query_knowledge_base(query: str) -> str:
    """查询知识库并返回格式化结果"""
    results = kb.search(query)
    
    if not results or (len(results) == 1 and results[0]["title"] == "知识库为空"):
        return "在知识库中未找到相关信息"
    
    response = "【知识库检索结果】\n"
    for i, res in enumerate(results, 1):
        response += f"\n{i}. {res['title']}\n{res['text']}\n相似度: {res['score']:.2f}\n"
    
    return response

# 角色定义
AGENT_A_PROMPT = """你是 Agent A，任务规划者。职责：
1. 分析用户请求，拆解为需要执行的子任务（每个子任务对应一个工具调用）
2. 为每个子任务指定需要调用的工具（get_weather、web_search 或 query_knowledge_base）和参数
3. 等待所有子任务执行完成后，整合结果并以自然语言回复用户

重要规则：
- 当用户询问特定领域知识（如技术概念、公司内部信息等）时，优先使用 query_knowledge_base 工具
- 当需要实时信息（如天气、新闻）时，使用 get_weather 或 web_search
- 不要自己编造知识库中没有的信息
- 工具调用结果会通过 tool_response 反馈给你"""

AGENT_B_PROMPT = """你是 Agent B，任务执行者。职责：
1. 接收 Agent A 的子任务指令
2. 严格按指令调用指定工具（get_weather、web_search 或 query_knowledge_base）
3. 仅返回工具调用的原始结果，不要添加任何解释或格式化

重要规则：
- 你不需要生成 tool_call
- 直接返回工具函数的执行结果字符串
- 结果必须是纯文本，不要包含JSON或其他结构"""

# 初始化Agent A的对话记忆
messages_a = [{"role": "system", "content": AGENT_A_PROMPT}]

# 工具列表定义（新增知识库查询工具）
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询城市实时天气",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "网络搜索关键词",
            "parameters": {
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "query_knowledge_base",
            "description": "查询内部知识库获取特定领域知识",
            "parameters": {
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"]
            }
        }
    }
]

def main():
    while True:
        user_input = input("\n你：")
        if user_input.lower() in ["exit", "quit", "退出"]:
            print("AI：拜拜~")
            break
        
        # 将用户输入加入Agent A的上下文
        messages_a.append({"role": "user", "content": user_input})
        
        # Agent A：拆解任务并生成工具调用计划
        while True:  # 处理多轮工具调用
            try:
                agent_a_response = client.chat.completions.create(
                    model="qwen-plus",
                    messages=messages_a,
                    tools=tools,
                    tool_choice="auto"
                )
                agent_a_msg = agent_a_response.choices[0].message
                messages_a.append(agent_a_msg)
            except Exception as e:
                print(f"API调用错误: {str(e)}")
                break
            
            # 检查是否需要工具调用
            if not hasattr(agent_a_msg, 'tool_calls') or not agent_a_msg.tool_calls:
                # 无需工具调用，直接输出结果
                print(f"AI：{agent_a_msg.content}")
                break
            
            # 处理所有工具调用
            for tool_call in agent_a_msg.tool_calls:
                func_name = tool_call.function.name
                try:
                    func_args = json.loads(tool_call.function.arguments)
                    print(f"正在执行: {func_name}({json.dumps(func_args, ensure_ascii=False)})...")
                except json.JSONDecodeError:
                    # 工具调用参数解析失败，反馈给Agent A
                    messages_a.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "name": func_name,
                        "content": f"工具调用参数解析失败：{tool_call.function.arguments}"
                    })
                    continue
                
                # 执行工具调用
                try:
                    if func_name == "get_weather":
                        result = get_weather(**func_args)
                    elif func_name == "web_search":
                        result = web_search(**func_args)
                    elif func_name == "query_knowledge_base":
                        result = query_knowledge_base(**func_args)
                    else:
                        result = f"未知工具：{func_name}"
                except Exception as e:
                    result = f"执行工具时出错: {str(e)}"
                
                # 将结果反馈给 Agent A
                messages_a.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "name": func_name,
                    "content": result
                })
        
        # 保存最终回复到上下文
        if hasattr(agent_a_msg, 'content') and agent_a_msg.content:
            messages_a.append({"role": "assistant", "content": agent_a_msg.content})

if __name__ == "__main__":
    main()

```

> 勉强跑通，但是具体的实现不太了解，感觉接口有点复杂了
>
> knowledge_base.md:
>
> ```markdown
> # DevOps
> DevOps是一个大笨蛋
> ```

效果：

![image-20251021115319200](https://raw.githubusercontent.com/calendar0917/images/master/image-20251021115319200.png)

> 知识缺漏比较多，先到这里吧
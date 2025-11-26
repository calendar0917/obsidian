---
title: "御林招新题：LLM 专题"
subtitle: "御林招新题：LLM 专题"
summary: "学习 LLM 的使用；python 一个 api 的使用脚本"
description: "学习 LLM 的使用；python 一个 api 的使用脚本"
image: ""
date: 2025-10-19
lastmod: 2025-10-19
draft: false
toc:
 enable: true
weight: false
hiddenFromHomePage: True
categories: ["CTF"]
tags: ["CTF"]
---

> LLM：Large Language Model

## **准备工作：获取 API Key**

- 注册一个 LLM 服务提供商的账号，推荐使用**硅基流动**（）。

> 有 qwen 的 Token了……

- 申请并获取你的 API Key。

## **编程调用 API**

> 看看文档：[大模型服务平台百炼控制台](https://bailian.console.aliyun.com/?tab=doc#/doc/?type=model&url=2840915)，官方提供了基本示例代码

- 选择一门你熟悉的编程语言，推荐使用 **Python**。
- 安装必要的库（推荐使用`openai`库）。
- 编写一个脚本，向 API 发送一个简单的请求。
- 解析 API 的返回结果，并将模型的回复打印到控制台。

```py
import os
from openai import OpenAI

try:
    client = OpenAI(
        # 若没有配置环境变量，请用阿里云百炼API Key将下行替换为：api_key="sk-xxx",
        # 新加坡和北京地域的API Key不同。获取API Key：https://help.aliyun.com/zh/model-studio/get-api-key
        api_key="......",
        # 以下是北京地域base_url，如果使用新加坡地域的模型，需要将base_url替换为：https://dashscope-intl.aliyuncs.com/compatible-mode/v1
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    )

    completion = client.chat.completions.create(
        model="qwen-plus",  # 模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
        messages=[
            {'role': 'system', 'content': '你是一个耐心的助手，熟悉计算机技能'},
            {'role': 'user', 'content': '简述DevOps是什么'}
            ]
    )
    print(completion.choices[0].message.content)
except Exception as e:
    print(f"错误信息：{e}")
    print("请参考文档：https://help.aliyun.com/zh/model-studio/developer-reference/error-code")
```

结果：

```
PS D:\code\test> & C:/python/python3.exe d:/code/test/llm.py
DevOps 是一组结合开发（Development）和运维（Operations）的实践方法，旨在缩短软件开发生命周期，提高软件交付的速度与质量。它通
过自动化、协作和持续改进，实现开发团队与运维团队之间的高效协同。

核心理念包括：

1. **持续集成（CI）**：开发人员频繁地将代码变更合并到主分支，并通过自动化测试验证。
2. **持续交付/部署（CD）**：确保代码可以随时安全地部署到生产环境，部分实现自动发布。
3. **自动化**：涵盖构建、测试、部署和监控等各个环节，减少人为错误，提升效率。
4. **监控与反馈**：实时监控系统运行状态，快速发现问题并反馈给开发团队。
5. **协作与文化**：强调团队之间的沟通、协作和责任共担，打破“开发”与“运维”之间的壁垒。

常用工具包括：Git、Jenkins、Docker、Kubernetes、Ansible、Prometheus、ELK 等。
```



## **模型能力探索**

- 尝试调用至少**三个不同**的模型。
- 向每个模型提问相同的问题，例如“简述DevOps是什么？”或“请编写一个用 Python 计算斐波那契数列的函数”。
- 观察并分析不同模型之间的回答风格、准确性、速度和内容差异。

>- qwen-plus:
>
>```
>DevOps 是一组结合开发（Development）和运维（Operations）的实践方法，旨在缩短软件开发生命周期，提高软件交付的速度与质量。它通
>过自动化、协作和持续改进，实现开发团队与运维团队之间的高效协同。
>
>核心理念包括：
>
>1. **持续集成（CI）**：开发人员频繁地将代码变更合并到主分支，并通过自动化测试验证。
>2. **持续交付/部署（CD）**：确保代码可以随时安全地部署到生产环境，部分实现自动发布。
>3. **自动化**：涵盖构建、测试、部署和监控等各个环节，减少人为错误，提升效率。
>4. **监控与反馈**：实时监控系统运行状态，快速发现问题并反馈给开发团队。
>5. **协作与文化**：强调团队之间的沟通、协作和责任共担，打破“开发”与“运维”之间的壁垒。
>
>常用工具包括：Git、Jenkins、Docker、Kubernetes、Ansible、Prometheus、ELK 等。
>```
>
>- qwen-max
>
>  ```
>  DevOps是一种文化和实践的集合，旨在通过促进开发（Development）、技术运营（Operations）以及其它相关团队之间的沟通、协作与整合，来提高软件开发和交付的速度及质量。它强调在整个软件生命周期中采用自动化工具和技术，以实现持续集成、持续交付/部署（CI/CD），从
>  而更快地响应市场变化，并提高组织的灵活性。
>  
>  DevOps的核心理念包括但不限于：
>  
>  - **文化转型**：鼓励跨职能团队合作，打破传统IT部门间的壁垒。
>  - **自动化**：利用各种工具自动化构建、测试和部署过程，减少人为错误，加快迭代速度。
>  - **持续改进**：基于反馈循环不断优化流程，追求更高效的工作方式。
>  - **测量与监控**：重视性能指标的收集与分析，确保系统稳定性和服务质量。
>  
>  总之，DevOps旨在通过加强不同角色之间的协作，结合敏捷方法论与自动化技术，帮助企业更快地将高质量的产品推向市场
>  ```
>
>- deepseek-v3.2-exp
>
>```
>DevOps 是一种结合软件开发（Development）与 IT 运维（Operations）的文化理念与实践方法，旨在通过自动化与协作，缩短软件开发生命
>周期，提升交付效率、系统稳定性与产品质量。
>
>### 核心目标：
>1. **快速交付**：通过自动化流程（如 CI/CD）加速软件发布。
>2. **高质量输出**：持续测试与监控，及早发现并修复问题。
>3. **跨团队协作**：打破开发与运维之间的壁垒，促进沟通与责任共担。
>
>### 关键实践：
>- **持续集成/持续部署（CI/CD）**：自动化构建、测试和部署流程。
>- **基础设施即代码（IaC）**：用代码管理服务器和配置，确保环境一致性。
>- **自动化测试与监控**：实时跟踪系统性能，快速响应故障。
>
>### 常用工具链示例：
>- **代码管理**：Git、GitLab
>- **CI/CD**：Jenkins、GitHub Actions
>- **容器化**：Docker、Kubernetes
>- **监控**：Prometheus、Grafana
>
>通过融合文化、流程与工具，DevOps 帮助团队更灵活地应对市场需求，实现高效可靠的软件交付。
>```

## **构建一个命令行聊天机器人**

- **功能实现**：编写一个 Python 脚本，创建一个可以持续与用户交互的命令行聊天机器人。
- **交互循环**：使用一个无限循环，不断接收用户的输入，并将其作为提示词（Prompt）发送给 LLM。
- **对话管理**：将 LLM 的回复打印回命令行，然后再次等待用户的输入。
- **退出机制**：当用户输入特定指令（例如 `quit` 或 `exit`）时，程序能够优雅地退出。

```py
# 改成了流式输出
import os
from openai import OpenAI

client = OpenAI(
    # 若没有配置环境变量，请用百炼API Key将下行替换为：api_key="sk-xxx",
    api_key="......",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)
prompt = input()
while(prompt != "exit" and prompt != "quit"):
    completion = client.chat.completions.create(
        model="qwen-plus",  # 此处以qwen-plus为例，可按需更换模型名称。模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
        messages=[{'role': 'system', 'content': 'You are a helpful assistant.'},
                    {'role': 'user', 'content': prompt}],
        stream=True,
        stream_options={"include_usage": True}
        )
    for chunk in completion:
        # 提取当前片段中的文本内容（delta.content）
        if chunk.choices: # 这里得判断一下
            content = chunk.choices[0].delta.content
            if content:  # 只打印非空内容（过滤空片段）
                print(content, end="", flush=True)  # 实时输出，不换行
    print()  # 每个回复结束后换行
    prompt = input()
print("拜拜~")
```



## **提示词工程（Prompt Engineering）**

- **角色扮演**：修改你的聊天机器人脚本，让模型扮演一个特定角色，例如“一位严厉的Linux导师”或“一个幽默的段子手”。
- **指令遵循**：尝试给模型一些复杂的指令，例如“用 Markdown 格式列出所有重要的 Linux 命令，并为每个命令提供一个简短的例子。”
- **限制输出**：要求模型在回答问题时，遵循特定的格式或长度限制，例如“只用100字以内回答。”

```py
import os
from openai import OpenAI

......
system = input("请输入你想让我扮演的角色：") # 更新了这里
prompt = input("你想对我说什么？")
while(prompt != "exit" and prompt != "quit"):
    completion = client.chat.completions.create(
        model="qwen-plus",  # 此处以qwen-plus为例，可按需更换模型名称。模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
        messages=[{'role': 'system', 'content': system},
                    {'role': 'user', 'content': prompt}],
        stream=True,
        stream_options={"include_usage": True}
        )
    ......
    prompt = input("再说点什么呢？")
print("拜拜~")
```


---
title: 一个 CF Workers AI 的简单实例
date: 2024-05-01 23:00:00
tags:
  - 教程
  - LLM
  - CF
categories: 教程
---
## 前言 / Introduction

都说 Cloudflare 是赛博佛祖，是真正推动 serverless 的慈善企业，这句话在Workers AI发布后再次得到了印证，个人搭建 AI 应用、跑 AI 模型从未如此简单过，接下来让我们通过一个简单的实例，亲身**搭建一个基于CF Workers AI的 LLM** 来看看开发流程究竟有多么低，有多么简单

## 前期准备 / Preparations

- 一个 Cloudflare 账号
- 一个 Python 开发环境
	- 需安装requests库
	- 可选装jsonpath

# 教程 / Tutorial
## 获取 Cloudflare Workers AI API Token

> 与其他 API 类似，你需要先获取一个API Token，以便计费和验明身份

进入[Cloudflare 控制台](https://dash.cloudflare.com)，登录你的 Cloudflare 账户，接着点击侧边栏中的**AI**选项卡，进入Workers AI 主界面，接着依次选择 **使用 REST API** -> **获取 API 令牌**，进入一个新的页面，*什么都不用管，直接滑到底*，点击**继续以显示摘要**即可，创建令牌，随后复制生成的令牌并妥善保存

>`请注意，生成的令牌只会显示这一次，且持有令牌可以直接访问你的 Workers AI 资源，请保存在一个安全的地方`

## 初阶代码

### 修改模型和Token

返回 “使用 Workers AI REST API” 页面，在下方代码区选择第三个 **python**，其应该类似下面这样

```python
import requests

API_BASE_URL = "https://api.cloudflare.com/client/v4/accounts/{id}/ai/run/"
headers = {"Authorization": "Bearer {API_TOKEN}"}
# 已隐去我的 account id，其本应填充{id}

def run(model, inputs):
    input = { "messages": inputs }
    response = requests.post(f"{API_BASE_URL}{model}", headers=headers, json=input)
    return response.json()

inputs = [
    { "role": "system", "content": "You are a friendly assistan that helps write stories" },
    { "role": "user", "content": "Write a short story about a llama that goes on a journey to find an orange cloud "}
];
output = run("@cf/meta/llama-2-7b-chat-int8", inputs)
print(output)

```

将代码复制，打开准备好的 Python 环境，复制进去，接下来我们需要将模型改为更适合中国宝宝体质的 **qwen1.5-7b-chat-awq**，在代码中找到
```python
output = run("@cf/meta/llama-2-7b-chat-int8", inputs)
```
将其中的`@cf/meta/llama-2-7b-chat-int8`修改为`@cf/qwen/qwen1.5-7b-chat-awq`，接着找到
```python
headers = {"Authorization": "Bearer {API_TOKEN}"}
```
将刚刚生成的 API Token 替换掉`{API_TOKEN}`即可
让我们试着运行一下程序，此时如果一切正常，程序应该已经可以正常运行并返回相应的结果了

### 修改Prompt

接下来我们需要修改 system prompt 和问题以做到提问的作用，让我们在程序中找到
```python
inputs = [
    { "role": "system", "content": "You are a friendly assistan that helps write stories" },
    { "role": "user", "content": "Write a short story about a llama that goes on a journey to find an orange cloud "}
];
```
其中的`"role": "system", "content":`后面的内容对应的是 LLM 中的 system prompt，将其修改为 **"你是一位优秀的中文助手"** （可以自由发挥，按需求修改）
而`"role": "user", "content":`后面则是你向 AI 提的问题，这里可以修改为你想说的话，例如 **"你好，请介绍一下你自己"**

至此，你其实已经完成了一个基础的实例，就是这么简单，接下来我们将实现**用户输入**和**输出优化**

## 进阶代码

### 实现用户输入

毫无难度的一部分，只需要对 inputs 开刀稍作修改即可，直接展示结果
```python
userinput = input("请输入要提的问题：")  
inputs = [  
    { "role": "system", "content": "你是一位优秀的中文助手" },  
    { "role": "user", "content": userinput}  
];  
output = run("@cf/qwen/qwen1.5-7b-chat-awq", inputs)
```
此处将原本固定的 prompt 修改为了一个读取用户输入的 userinput，便实现了用户输入的功能

### 输出优化

如果你留心注意，不难发现现在的输出真是丑的要命，是一个裸返回 json，类似
```json
{'result': {'response': '你好！很高兴能为你提供帮助。有什么问题或需要帮助的吗？'}, 'success': True, 'errors': [], 'messages': []}
```
我们其实只需要`response`中的内容，要实现剥离 response，我们就需要请出选装的 jsonpath 了，在 output 底下加一行
```python
final = jsonpath(output,"$..response")[0]
```
再将`print(output)`修改为`print(final)`即可

这样，你的输出就变成了纯净的 **"你好！很高兴能为你提供帮助。有什么问题或需要帮助的吗？"**

## 更进一步？

其实到刚才为止，作为一个一轮对话的 AI 已经相当完善了，但是我们也可以更进一步，实现多轮对话的功能，具体实现过程见代码，这里不多做讲解了

### Final Code

```python
import requests  
from jsonpath import jsonpath  
  
info = ["你是一位优秀的中文聊天助手，前文的所有聊天为："]  
API_BASE_URL = "https://api.cloudflare.com/client/v4/accounts/{id}/ai/run/"  
# Replace with your user id
headers = {"Authorization": "Bearer {API_TOKEN}"}  
# Replace with your API_TOKEN
  
  
def run(model, inputs):  
    input = { "messages": inputs }  
    response = requests.post(f"{API_BASE_URL}{model}", headers=headers, json=input)  
    return response.json()  
  
userinput = input("请输入要提的问题：")  
waittoaddU = "用户提问：" + userinput  
info.append(waittoaddU)  
inputs = [  
    { "role": "system", "content": "你是一位优秀的中文助手" },  
    { "role": "user", "content": userinput}  
];  
output = run("@cf/qwen/qwen1.5-7b-chat-awq", inputs)  
final = jsonpath(output,"$..response")[0]  
waittoaddA = "系统回答：" + final  
info.append(waittoaddA)  
print(final) 
  
while True:  
    userinput = input("请输入要提的问题：")  
    if userinput == "EXIT":  
        break  
    inputs = [  
        { "role": "system", "content": "\n".join(info) },  
        { "role": "user", "content": userinput}  
    ];  
    output = run("@cf/qwen/qwen1.5-7b-chat-awq", inputs)  
    waittoaddU = "用户提问：" + userinput  
    info.append(waittoaddU)  
    final = jsonpath(output,"$..response")[0]  
    waittoaddA = "系统回答：" + final  
    info.append(waittoaddA)  
    print(final)
```

---

*感谢您的阅读*
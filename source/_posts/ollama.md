---
title: Mac本地部署大模型
date: 2024-09-14 16:28:43
cover: 'icon.jpeg'
tags: LLM
---


# Mac本地部署大模型

随着大型语言模型（LLM）的发展，人们越来越重视AI的潜力。但是，使用云服务或在线API可能会带来数据隐私、安全性等问题。本文将展示如何在本地机器上部署一个`Ollama`模型，实现快速高效的自主语义分析和生成能力。

## 安装Ollama

### Ollama 简介
> Ollama 是一个基于 Go 语言开发的可以本地运行大模型的开源框架。

官网：[ollama.com/](https://ollama.com/)
GitHub 地址：[github.com/ollama/olla…](https://github.com/ollama/ollama)

### 下载安装Ollama
在`Ollama`官网根据自己的操作系统选择对应的安装包，我用的`Macbook Air M1`，选择`macOS`下载安装。
![下载Ollama](download.png)

### 本地验证
下载安装完成后一般会自动启动`Ollama`服务，也可以自己点击ollama应用或者命令行运行`ollama serve`启动服务。
命令行执行`ollama -v`显示版本号说明安装成功。
```cmd
yuxinxin@yuxinxindeMacBook-Air.local ~ ollama -v
ollama version is 0.3.9
```


### 下载模型
安装完后默认提示安装`llama2`大模型，也可以下载自己需要的模型。

支持的模型官网：[https://ollama.com/library](https://ollama.com/library)

命令行下载模型使用`ollama pull xxx`命令，如下载`llama3.1:8b`：
```cmd
ollama pull llama3.1:8b
```


## 使用Ollama

### 命令行对话

```cmd
yuxinxin@yuxinxindeMacBook-Air.local ~ ollama run llama3.1:8b
>>> 你好呀
你好！很高兴和你互动！你想谈论什么呢？
```

### 使用可视化页面
使用cherry studio调用本地ollama

1. 安装Cherry Studio就不描述了，按照官网教程安装即可
2. 打开Cherry Studio的设置页面
![设置页面](cherry_setting.png)
3. API密钥随便填，本地没有校验
4. API地址填写：http://localhost:11434/v1/
5. 点击底部“管理”按钮，添加需要的模型
![模型选择](models.png)
6. 进入对话页面，选择需要的模型，我这里本地部署里deepseek r1 7b模型
![对话模型选择](choose_model.png)
7. 对话
![对话](chat.png)


## ollama 暴露的接口
ollama 提供了 RESTful API，方便开发者将其集成到其他应用程序中。通过这些接口，可以实现更灵活的交互和自动化任务。

### （一）接口地址
ollama的API地址为`http://localhost:11434/api`，所有 API 请求都基于此地址。

### （二）常见接口示例
1. 对话接口
    - 请求：
        使用POST请求到`http://localhost:11434/api/chat`。请求体为 JSON 格式，包含以下字段：
    ```json
    {
        "model": "deepseek-r1:latest",
        "messages": [{
            "role": "user",
            "content": "请写一首关于春天的诗"
        }],
        "stream": false
    }
    ```
    其中，model指定使用的模型，prompt为输入的文本提示。

    - 响应：
    成功时，响应体为 JSON 格式，包含生成的文本。例如：
    ```json
    {
        "model": "deepseek-r1:latest",
        "created_at": "2025-03-18T09:16:22.980721Z",
        "message": {
            "role": "assistant",
            "content": "<think>\n好，用户让写一首关于春天的诗。首先，我需要确定诗的主题和情感基调。春天通常让人联想到生机、希望和温馨的感觉，所以我想通过自然景物来表达这种感觉。\n\n接下来，考虑诗的结构。中文诗常用四句或八句，这里用五言律诗比较合适，每首五句，押韵。这样结构清晰，容易表达情感。\n\n然后是具体的意象选择。春天有花、柳树、燕子等元素，这些都能很好地表现春天的生机和活力。我打算用“繁花”、“细雨”、“新绿”来描绘春天的景象。\n\n再考虑诗的内容安排。开头点题，中间展开，结尾总结。比如第一句点明主题，第二句描述春天的到来，第三句和第四句用自然现象表现春天的特点，最后两句表达对春天的喜爱和结束。\n\n在用词上，要简洁优美，避免生僻字，让读者容易理解。同时注意押韵，使整首诗读起来流畅自然。\n\n最后，写完后还要写一个赏析部分，解释诗中的意象和情感表达，让用户更好地理解和感受这首诗的魅力。\n</think>\n\n《春》\n春来别样新,繁花动地深。\n细雨迎风落,新绿随云侵。\n喜燕呼莺语,欢蛙奏琴音。\n此景年年见,常驻Memory。\n\n赏析：这首作品描绘了春天的景象，通过“繁花动地深”和“新绿随云侵”的意象展现春色的浓郁。尾联以“燕呼莺语”、“蛙奏琴音”寄托了对春天的热爱与怀念，表达了年复一年春景常驻记忆的情感。"
        },
        "done_reason": "stop",
        "done": true,
        "total_duration": 41346353792,
        "load_duration": 571510084,
        "prompt_eval_count": 10,
        "prompt_eval_duration": 4194000000,
        "eval_count": 371,
        "eval_duration": 36578000000
    }
    ```


2. 列出模型
    - 请求：
    使用GET请求到`http://localhost:11434/api/tags`。
    - 响应：
    响应体为 JSON 格式，列出所有已安装的模型信息，包括模型名称、大小等。例如：
    ```json
    [
        {
            "name": "deepseek-r1:latest",
            "model": "deepseek-r1:latest",
            "modified_at": "2025-02-01T22:20:07.879258573+08:00",
            "size": 4683075271,
            "digest": "0a8c266910232fd3291e71e5ba1e058cc5af9d411192cf88b6d30e92b6e73163",
            "details": {
                "parent_model": "",
                "format": "gguf",
                "family": "qwen2",
                "families": [
                    "qwen2"
                ],
                "parameter_size": "7.6B",
                "quantization_level": "Q4_K_M"
            }
        },
        {
            "name": "llava:latest",
            "model": "llava:latest",
            "modified_at": "2024-12-26T12:59:51.582141696+08:00",
            "size": 4733363377,
            "digest": "8dd30f6b0cb19f555f2c7a7ebda861449ea2cc76bf1f44e262931f45fc81d081",
            "details": {
                "parent_model": "",
                "format": "gguf",
                "family": "llama",
                "families": [
                    "llama",
                    "clip"
                ],
                "parameter_size": "7B",
                "quantization_level": "Q4_0"
            }
        },
        {
            "name": "nomic-embed-text:latest",
            "model": "nomic-embed-text:latest",
            "modified_at": "2024-11-12T20:11:39.707148179+08:00",
            "size": 274302450,
            "digest": "0a109f422b47e3a30ba2b10eca18548e944e8a23073ee3f3e947efcf3c45e59f",
            "details": {
                "parent_model": "",
                "format": "gguf",
                "family": "nomic-bert",
                "families": [
                    "nomic-bert"
                ],
                "parameter_size": "137M",
                "quantization_level": "F16"
            }
        },
        {
            "name": "qwen2.5:latest",
            "model": "qwen2.5:latest",
            "modified_at": "2024-09-21T16:19:27.918562447+08:00",
            "size": 4683087332,
            "digest": "845dbda0ea48ed749caafd9e6037047aa19acfcfd82e704d7ca97d631a0b697e",
            "details": {
                "parent_model": "",
                "format": "gguf",
                "family": "qwen2",
                "families": [
                    "qwen2"
                ],
                "parameter_size": "7.6B",
                "quantization_level": "Q4_K_M"
            }
        }
    ]
    ```

## END
通过在 Mac 上本地部署 ollama 并利用其暴露的接口，无论是进行日常的创意写作、知识问答，还是开发集成 AI 功能的应用程序，都变得触手可及。希望这篇博客能帮助你顺利开启 ollama 的本地部署和使用之旅，探索更多人工智能的奇妙应用。



<div class="aplayer no-destroy" data-id="热门" data-server="tencent" data-type="search" data-fixed="true" data-mini="true" data-listFolded="false" data-order="random" data-preload="none" data-autoplay="true" muted></div>
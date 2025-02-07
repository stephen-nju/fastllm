# LLaMA 类模型转换参考

这个文档提供了了转换LLaMA同结构模型的方法。

LLaMA类模型有着基本相同的结构，但权重和prompt构造有差异。在fastllm中，通过转转模型时修改部分配置，实现对这些变体模型的支持、

## 声明

以下配置方案根据模型的源代码整理，不保证模型推理结果与原版完全一致。

## 修改脚本并转换

这里以支持推理各类Llama结构的基座模型为例，介绍如何应用本文档。

* 方案一：修改转换脚本

以alpaca2flm.py为模板修改。在创建model之后添加：

```python
    model = LlamaForCausalLM.from_pretrained(model_name).float()
    # config.json中定义了自己的model_type的需要添加
    conf = model.config.__dict__
    conf["model_type"] = "llama"
    # 接下来的部分各个Chat模型有差别，Base模型有的需要添加pre_prompt。
    torch2flm.tofile(exportPath, model, tokenizer, pre_prompt = "", 
                     user_role = "", bot_role = "", history_sep = "", 
                     dtype = dtype)
```
其中，`pre_prompt` 、`user_role` 、`bot_role` 、`history_sep`分别为“开始的系统提示词（第一轮对话之前）”，“用户角色标志”，“用户话语结束标志及模型回复开始标志”，“两轮对话之间的分隔符”。

* 方案二：修改config.json
在下载的模型目录下，修改配置文件`config.json`中，修改"model_type"为`llama`，并增加下面的键-值对：

```json
    "pre_prompt": "",
    "user_role": "",
    "bot_role": "",
    "history_sep":  "",
```

如需添加Token ID而非字符串（类似baichuan-chat模型），可以使用“<FLM_FIX_TOKEN_{ID}>”的格式添加。

### 两行加速

```python
    llm.from_hf(model, tokenizer, pre_prompt = "", 
                user_role = "", bot_role = "", history_sep = "", 
                dtype = dtype)
```

## Base Model

见上方“[修改方案](#修改方案)”。

一部分模型需要制定bos_token_id，假设bos_token_id为1则可以配置如下：

```python
    torch2flm.tofile(exportPath, model, tokenizer, pre_prompt = "<FLM_FIX_TOKEN_1>", 
                     user_role = "", bot_role = "", history_sep = "", 
                     dtype = dtype)
```

## Chat Model

对Chat Model，同样是修改转换脚本，或修改模型的config.json，以下是目前常见的chat model的配置：

### InternLM（书生）

* internlm/[internlm-chat-20b](https://huggingface.co/internlm/internlm-chat-20b)

```python
    conf = model.config.__dict__
    conf["model_type"] = "llama"
    torch2flm.tofile(exportPath, model, tokenizer, pre_prompt = "<s><s>", 
                     user_role = "<|User|>:", bot_role = "<eoh>\n<|Bot|>:", 
                     history_sep = "<eoa>\n<s>", dtype = dtype)
```


### XVERSE

* xverse/[XVERSE-13B-Chat](https://huggingface.co/xverse/XVERSE-13B-Chat)
* xverse/[XVERSE-7B-Chat](https://huggingface.co/xverse/XVERSE-7B-Chat)

```python
    conf = model.config.__dict__
    conf["model_type"] = "llama"
    torch2flm.tofile(exportPath, model, tokenizer, pre_prompt = "", 
                     user_role = "Human: ", bot_role = "\n\nAssistant: ", 
                     history_sep = "<FLM_FIX_TOKEN_3>", dtype = dtype)
```

### 其他 llama1 系列

* Vicuna v1.1 v1.3
```python
    torch2flm.tofile(exportPath, model, tokenizer, 
                     pre_prompt="A chat between a curious user and an artificial intelligence assistant. "
                                "The assistant gives helpful, detailed, and polite answers to the user's questions. "
                     user_role="USER: ", bot_role=" ASSISTANT:",  history_sep="<s>", dtype=dtype)
```

* BiLLa 
```python
    torch2flm.tofile(exportPath, model, tokenizer, pre_prompt = "\n", 
                     user_role = "Human: ", bot_role = "\nAssistant: ", 
                     history_sep = "\n", dtype = dtype)
```

### llama2-chat

* meta-llama/Llama-2-chat

|Model|Llama2-chat|Llama2-chat-hf|
|-----|-----|-----|
|  7B | [meta-llama/Llama-2-7b-chat](https://huggingface.co/meta-llama/Llama-2-7b-chat) | [meta-llama/Llama-2-7b-chat-hf](https://huggingface.co/meta-llama/Llama-2-7b-chat-hf) |
| 13B | [meta-llama/Llama-2-13b-chat](https://huggingface.co/meta-llama/Llama-2-13b-chat) | [meta-llama/Llama-2-13b-chat-hf](https://huggingface.co/meta-llama/Llama-2-13b-chat-hf) |

官方示例代码中，可以不用系统提示语：

```python
    torch2flm.tofile(exportPath, model, tokenizer, pre_prompt = "<FLM_FIX_TOKEN_1>", 
                     user_role = "[INST] ", bot_role = " [/INST]", 
                     history_sep = " <FLM_FIX_TOKEN_2><FLM_FIX_TOKEN_1>", dtype = dtype)
```

**Llama-2系列支持系统提示语需要修改代码**，单轮可以使用以下带有系统提示语的版本：

```python
    torch2flm.tofile(exportPath, model, tokenizer, 
                     pre_prompt = "<FLM_FIX_TOKEN_1>[INST] <<SYS>>\nYou are a helpful, respectful and honest assistant. Always answer as helpfully as possible, " \
        "while being safe. Your answers should not include any harmful, unethical, racist, sexist, toxic, dangerous, or illegal content. " \
        "Please ensure that your responses are socially unbiased and positive in nature.\n\nIf a question does not make any sense, " \
        "or is not factually coherent, explain why instead of answering something not correct. If you don't know the answer to a question, " \
        "please don't share false information.\n<</SYS>>\n\n", 
                     user_role = " ", bot_role = " [/INST]", 
                     history_sep = " <FLM_FIX_TOKEN_2><FLM_FIX_TOKEN_1>", dtype = dtype)
```

* ymcui/Chinese-Alpaca-2

|Model|Chinese-Alpaca-2|Chinese-Alpaca-2-16K|
|-----|-----|-----|
|  7B | [ziqingyang/chinese-alpaca-2-7b](https://huggingface.co/ziqingyang/chinese-alpaca-2-7b) | [ziqingyang/chinese-alpaca-2-7b-16k](https://huggingface.co/ziqingyang/chinese-alpaca-2-7b-16k) |
| 13B | [ziqingyang/chinese-alpaca-2-13b](https://huggingface.co/ziqingyang/chinese-alpaca-2-13b) | [ziqingyang/chinese-alpaca-2-13b-16k](https://huggingface.co/ziqingyang/chinese-alpaca-2-13b-16k) |

```python
    torch2flm.tofile(exportPath, model, tokenizer, 
                     pre_prompt = "<FLM_FIX_TOKEN_1>[INST] <<SYS>>\nYou are a helpful assistant. 你是一个乐于助人的助手。\n<</SYS>>\n\n"
                     user_role = " ", bot_role = " [/INST]", 
                     history_sep = " <FLM_FIX_TOKEN_2><FLM_FIX_TOKEN_1>", dtype = dtype)
```

### RUC-GSAI/YuLan-Chat

  * Full
    * [YuLan-Chat-2-13B](https://huggingface.co/yulan-team/YuLan-Chat-2-13b-fp16)
  * Delta (需要原始LLaMA)
    * [YuLan-Chat-1-65B-v2](https://huggingface.co/yulan-team/YuLan-Chat-1-65B-v2-delta) 
    * [YuLan-Chat-1-65B-v1](https://huggingface.co/RUCAIBox/YuLan-Chat-65b-delta) 
    * [YuLan-Chat-1-13B-v1](https://huggingface.co/RUCAIBox/YuLan-Chat-13b-delta) 

```python
    torch2flm.tofile(exportPath, model, tokenizer, 
                     pre_prompt="The following is a conversation between a human and an AI assistant namely YuLan, developed by GSAI, Renmin University of China. " \
                                "The AI assistant gives helpful, detailed, and polite answers to the user's questions.\n"
                     user_role="[|Human|]:", bot_role="\n[|AI|]:", history_sep="\n", dtype=dtype)
```

### WizardCoder

  * [WizardCoder-Python-7B-V1.0](https://huggingface.co/WizardLM/WizardCoder-Python-7B-V1.0)
  * [WizardCoder-Python-13B-V1.0](https://huggingface.co/WizardLM/WizardCoder-Python-13B-V1.0)

```python
    torch2flm.tofile(exportPath, model, tokenizer, 
                     pre_prompt="Below is an instruction that describes a task. "
                                "Write a response that appropriately completes the request.\n\n"
                     user_role="### Instruction:\n", bot_role="\n\n### Response:", history_sep="\n", dtype=dtype)
```

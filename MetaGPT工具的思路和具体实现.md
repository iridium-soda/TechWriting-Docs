# MetaGPT:工具的思路和具体实现

> [!NOTE]
>
> 本文的实现基于[MetaGPT](https://github.com/geekan/metagpt)

## Description

[MetaGPT](https://github.com/geekan/metagpt)是一个多Agent架构的大模型框架，通过多个Agent之间的协作完成复杂的工作任务。本文主要解决在该项目提供的基本架构下整合和外部工具交互的问题。由于该项目这方面的文档缺失且缺乏示例，因此需要从源代码层面进行分析和补充。

## MetaGPT的基本元素和协作结构

> [!tip]
>
> 完整的结构说明可以参考Reference中的官方文档

在MetaGPT中，角色（Role，即Agent）为行动和交互的最基本单位，**每个角色有数个用户编写的动作（Actions）**，在每个行动阶段中，某个角色通过思考选择一个动作执行并得到结果，将结果送（Publish）至记忆池中供其他角色取用和思考。MetaGPT给出的流程图如下：
<img src="https://docs.deepwisdom.ai/main/assets/agent_run_flowchart.6c04f3a2.png" alt="WorkFlow" style="zoom:33%;" />

单个角色内部的动作选择模式有下面几种：

- `react`模式：角色会对每次的执行结果进行阅读和思考，并在下一次执行开始前决定是完成工作流并退出还是重新选择一个动作执行。
- `by_order`模式：按照用户指定的顺序进行行动。
- `plan_and_act`模式：在流程的最初，角色会根据任务指示，制定一个计划将任务拆分为多个子任务，每个子任务由动作执行一次，且子任务之间存在依赖关系，以此决定执行顺序。

在多角色架构下，角色之间通过一个共享的记忆池进行交互和触发。记忆池本质上是一个列表，基本单位为信息（message），具有下面的属性：

- `content`：信息的内容，即信息的主体部分
- `role`：发送信息的角色名（角色的`profile`字段）
- `cause_by`：发送该信息的动作名
- `send_to`：可选项，指定该信息只对某个角色可见（例如用于狼人杀模型中）

当某个角色的某个动作完成后主动将信息推送（publish）至记忆池，对其他角色均可见，可以通过特定方法调用。如果有动作被配置为监听该动作，则会在信息推送后被触发，开始执行任务。

> [!Note]
>
> 目前MetaGPT调取信息的方法只有调取最近的`k`条信息一种。在未来似乎可以接入RAG等方式，但MetaGPT没有实现其他调取信息的方式，似乎也不打算在这个方向进行深化开发。

## 构建一个角色

为了定义一个**完全使用LLM(没有外部工具参与)**的角色，需要在`Role`类上做继承。通用的定义方法如下，以`CodeAnalyzer`为例：

```python
class CodeAnalyzer(Role):
    name: str = "CodeAnalyzer" # 角色的名称
    profile: str = "CodeAnalyzer" # 角色的名称，最好和role一致
    
    def __init__(self, **kwargs): # 重写Role类的__init__方法
        super().__init__(**kwargs) # 原方法的初始化流程
        self.set_actions([SendFiletoFbinfer,WriteReport]) # 通过该方法指定该角色可以执行的动作
        self._set_react_mode(react_mode=RoleReactMode.BY_ORDER.value) # 指定角色的思考模式
    
    async def _act(self) -> Message: # 重写act方法
        logger.info(f"{self._setting}: to do {self.rc.todo}({self.rc.todo.name})")
        todo = self.rc.todo #拿到接下来的动作
        # 提取最近的一条msg作为参数填充到动作中
        msg = self.get_memories(k=1)[0]  # find the most k recent messages 
        
        result = await todo.run(msg.content) # 异步执行动作
        logger.info(f"{result=}")
        msg = Message(content=result, role=self.profile, cause_by=type(todo)) 
        self.rc.memory.add(msg) # 推送信息
        return 
```

通用的动作定义方法和例子如下：

```python
class WriteReport(Action):
    PROMPT_TEMPLATE: str = """
    Write a report based on the analysis results in the following format:
    # Report Summary: Analysis of the provided C code using Facebook Infer (fbinfer)
    ## Code Analysis Summary:
    - Code Language: C
    - Code File: test.c
    - Analysis Tool: Facebook Infer (fbinfer)
    - Analysis Status: Success/Failure
    - Any issues found: Yes/No
    ## Code Analysis Details:
    - [Line Number] Issue Type: Issue Description
    ## Recommendations:
    - Recommendation 1
    - Recommendation 2
    [End of Report]
    
    Here is the result of the analysis:
    {result}
    Your report:
    """ # 静态提示词模板，需要输入到LLM中的，其中需要动态填充的以{}标识
    async def run(self, result: str)->str: # 重写run方法，参数为需要填充进去的内容或其他内容
        prompt = self.PROMPT_TEMPLATE.format(result=result) # 填充动态prompt
        logger.info(f"{prompt=}")
        rsp = await self._aask(prompt) # 向大模型发送prompt并接收结果

        report_text=rsp # To be filtered
        logger.info(f"{report_text=}")
        return report_text # 返回结果
```

## 工具的引入和使用：Action方法和Role方法

### 工具的基本概念

大模型虽然在阅读理解、创造等方面具有强大的能力，但其基于概率的文字补全架构在数值计算、代码执行等客观任务上存在劣势。其次，一些场景下也需要引入已有的程序供大模型自动化调用与执行，并阅读结果，因此工具的概念被提出。在大模型（如GPT-4等）中，“工具”是指可以增强模型能力的外部模块或插件，允许模型在其原有的语言生成能力之外执行更复杂的任务，可以抽象为大模型可以调用执行的“函数”。例如，这些工具可能包括数据库查询、计算器、代码执行、API 调用、浏览器访问、图片生成等。目前主流的底座大模型（GPT，Claude等）均已提供或计划提供原生的工具支持。

一个工具需要具有下面的属性：
1. 工具的功能：大模型需要知道工具的功能，以便进行工具选择，例如计算器、浏览器等。
2. 工具的接口声明：工具暴露给大模型的接口和声明，以便大模型生成调用工具的代码。为了大模型正确生成调用代码/命令，其中动态部分应尽量简单。
3. 工具的实现：为保证工具的运行环境不冲突且方便创造干净的环境和依赖，最好使用容器对工具之间和工具与大模型之间进行隔离，同时容器接口也方便对工具进行简便和快速的调用。

构建一个工具通常有以下步骤：

1. 在容器中创建工具的执行环境，明确指令格式和参数
1. 编写接口代码，对容器做一个包装，作为单独的文件程序保证环境稳定性
1. 编写调用工具的action，在注释中注明功能并且规定传入参数
1. 将action注册到role中

### Role方法与Action方法



## 工具的基本实现步骤

本节以[iridium-soda/meta_toolbox: An extensible toolkit for managing and deploying large model agents through Docker, ensuring uniformity and scalability. (github.com)](https://github.com/iridium-soda/meta_toolbox)中的fbinfer工具为例，说明如何逐级编写和部署工具。

Fbinfer是由META（Facebook）推出的一款轻量级代码静态扫描工具，具有即插即用等特点。例如对于下面的示例代码：

```c
// hello.c
#include <stdlib.h>

void test() {
  int *s = NULL;
  *s = 42;
}
```

运行命令：

```shell
infer run -- gcc -c /code.c
```



有下面的扫描结果：

```plaintext
code.c:7: error: Null Dereference
  `s` could be null (null value originating from line 6) and is dereferenced. 
  5. void test() {
  6.   int *s = NULL;
  7.   *s = 42;
       ^
  8. }


Found 1 issue
             Issue Type(ISSUED_TYPE_ID): #
  Null Dereference(NULLPTR_DEREFERENCE): 1
```

该工具的目标是让角色根据情况调用该工具并进行扫描，阅读扫描结果并作为context进行下一步行动。为简便起见，仅考虑`.c`和`.cpp`代码（使用`gcc`/`g++`编译）

### 基于Docker的工具包装

为了环境的隔离，我们在docker容器中创建并持久化工具的运行环境。 Fbinfer的[环境配置](https://github.com/facebook/infer)比较简单，我们编写了如下的Dockerfile。该容器已发布到dockerhub上（`iridium191/fbinfer:latest`）。为保险起见，推荐将工具的Dockerfile保留到代码目录中以方便本地构建。

```dockerfile
FROM ubuntu:24.04
# 安装必要的软件包并清理缓存
RUN apt update \
    && apt install -y --no-install-recommends --reinstall curl wget xz-utils gcc g++ ca-certificates \
    && rm -rf /var/lib/apt/lists/*
ADD https://github.com/facebook/infer/releases/download/v1.2.0/infer-linux-x86_64-v1.2.0.tar.xz /
RUN  	tar -C /opt -xJ -f infer-linux-x86_64-v1.2.0.tar.xz && \
	ln -s "/opt/infer-linux-x86_64-v1.2.0/bin/infer" /usr/local/bin/infer && \
	rm infer-linux-x86_64-v1.2.0.tar.xz
CMD ["sleep","infinity"]
```



### 基于Action方法的工具实现

基于Action方法的工具实现中，工具作为挂在角色下面的一个动作使用，由大模型选择调用。一个典型的用于调用工具的动作定义如下。

```python
```



## 基于上述方法的Toolbox模块设计

## jyhllm中toolchain的模块整合

## 一些小问题



## Reference

- [MetaGPT | MetaGPT (deepwisdom.ai)](https://docs.deepwisdom.ai/)
- [qianyouliang/MetaGPT-Learn](https://github.com/qianyouliang/MetaGPT-Learn)

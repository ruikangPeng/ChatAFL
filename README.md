# ChatAFL Artifact

ChatAFL是一个由大型语言模型（LLM）引导的协议模糊器。它建立在[ALFNet](https://github.com/aflnet/aflnet)之上而是与三个具体组件集成在一起。首先，模糊器使用LLM为用于结构感知变异的协议提取机器可读语法。其次，模糊器使用LLM来增加记录的消息序列中用作初始种子的消息的多样性。最后，模糊器使用LLM来突破覆盖平台，在那里LLM被提示生成到达新状态的消息。

ChatAFL Artifact在[ProfuzzBench](https://github.com/profuzzbench/profuzzbench)中配置，是一个广泛使用的网络协议状态模糊化基准。这允许与已经建立的格式进行平滑集成。

## Folder structure
```
ChatAFL-Artifact
├── aflnet: 输出状态和状态转换的AFLNet的修改版本 
├── analyse.sh: 分析脚本 
├── benchmark: ProfuzBench的修改版本，仅包含添加了Lighttpd 1.4的基于文本的协议 
├── clean.sh: 清理脚本
├── ChatAFL: ChatAFL的源代码，以及论文中提出的所有策略
├── ChatAFL-CL1: ChatAFL，仅使用结构感知突变（参见消融研究） 
├── ChatAFL-CL2: ChatAFL，仅使用结构感知和初始种子富集（参见消融研究）
├── deps.sh: 安装依赖项的脚本，执行时要求输入密码
├── README: 这个文件
├── run.sh: 在项目上运行模糊器并收集数据的执行脚本
└── setup.sh: 设置docker镜像的准备脚本
```

## 1. 设置和使用

### 1.1. 安装依赖

`Docker`、`Bash`、`Python3`以及`panda`和`matplotlib`库。我们提供了一个助手脚本`deps.sh`，它运行所需的步骤以确保提供所有依赖项：
```
./deps.sh
```

### 1.2. 准备docker镜像 [~40 minutes]

运行以下命令来设置所有docker镜像，包括带有所有模糊器的项目：
```
KEY=<OPENAI_API_KEY> setup.sh
```

这个过程估计大约需要40分钟。我们在工件附录中提供了一个OpenAI API密钥，用于工件评估。

### 1.3. 运行实验

利用`run.sh`脚本运行实验。命令如下：
```
 ./run.sh <container_number> <fuzzed_time> <subjects> <fuzzers>
```

其中：
`container_number`指定创建多少个容器以在特定主题上运行单个模糊器（每个容器在一个主题上运行一个模糊器）。`run_time`表示以分钟为单位的运行时间。 `subjects`是被测对象的列表，`fuzzers`是用来模糊被测对象。例如，命令（`run.sh 1 5 pure-ftpd chatafl`）将为模糊器 ChatAFL创建1个容器，用来模糊对象 pure-ftpd 5分钟。 简而言之，通过使用写`all`代替subject和fuzzer列表，可以执行所有fuzzer和所有subject。

脚本完成后，将在`benchmark`目录中创建一个文件夹`result-<name of subject>`，其中包含每次运行的模糊结果。

### 1.4. 分析结果

`analyze.sh` 脚本用于分析数据，并构建图表，说明每个主题上模糊器随时间的平均代码和状态覆盖率。使用以下命令执行脚本：
```
./analyze.sh <pattern for subjects> <fuzzed_time in minutes> 
```

该脚本包含两个参数-主题的正则表达式和要分析的运行持续时间。请注意，这些参数是可选的，默认情况下脚本将处理所有`result-`文件夹，并假设执行时间为1440分钟，相当于1天。 

执行完成后，脚本将通过构造csv文件来处理归档，csv文件包含随时间变化的分支、状态和状态转换的覆盖数量。此外，这些csv文件将被处理成PNG文件，这些文件是绘图，说明了每个主题上模糊器随时间的平均代码和状态覆盖率（`cov_over_time...`用于代码和分支覆盖率，`state_over_time...`用于状态和状态转换覆盖率）。所有这些信息都被移动到根目录中的`res_`文件夹中，并带有时间戳。 

### 1.5. Cleaning Up

当评估完成时，运行 `clean.sh` 脚本将确保只有剩余的文件在此目录中：
```
./clean.sh
```

## 2. 函数分析

### 2.1. 检查 LLM 生成的 Grammars

语法生成的源代码位于`afl-fuzz.c`中的函数`setup_llm_grammars`中，`chat-llm.c`中有helper函数。

LLM对语法生成的响应可以在运行的最终存档中的`protocol-grammars`目录中找到。

### 2.2. 检查富集种子

种子富集的源代码位于`afl-fuzz.c`中的`get_seeds_with_messsage_types`函数中，`chat-llm.c`中有helper函数。

丰富的种子可以在运行的最终存档中的种子`queue`目录中找到。这些文件的名称为：`id:...,orig:enriched_`。

### 2.3. 正在检查状态暂停响应

状态暂停处理的源代码位于函数`fuzz_one`中，从6846行开始 (`if (uninteresting_times >= UNINTERESTING_THRESHOLD && chat_times < CHATTING_THRESHOLD){`).

状态暂停提示及其相应的响应可以在运行的最终存档中的`stall-interactions`目录中找到。这些文件的格式为`request-<id>`和`response-<id>`，包含我们构建的请求和LLM的响应。


## 3. 复制结果

为了进行论文中概述的实验，我们使用了大量的资源。使用标准台式机在一天内复制所有实验是不切实际的。为了便于对工件进行评估，我们缩小了实验规模，使用了更少的模糊器、受试者和迭代。

### 3.1. 与基线的比较[5人-分钟 + 180计算机-小时]

ChatAFL可以覆盖更多的状态和代码，并比基线工具AFLNet更快地实现相同的状态和编码覆盖。我们运行以下命令来支持这些声明：
```
./run.sh 5 240 kamailio,pure-ftpd,live555 chatafl,aflnet
./analyze.sh ".*" 240
```

命令完成后，将生成一个前缀为`res_`的文件夹。该文件夹包含PNG文件，这些文件说明了随着时间的推移两个模糊器所覆盖的状态和代码，以及所有运行的输出档案。它将被放置在工件的根目录中。

### 3.2. 消融研究[5人-分钟 + 180计算机-小时]

ChatAFL中提出的每一种策略都有助于不同程度的代码覆盖率改进。我们运行以下命令来支持此声明：
```
./run.sh 5 240 proftpd,exim chatafl,chatafl-cl1,chatafl-cl2
./analyze.sh ".*" 240
```
命令完成后，将生成一个前缀为`res_`的文件夹。该文件夹包含PNG文件，这些文件说明了三个模糊器随时间推移所覆盖的代码以及所有运行的输出档案。它将被放置在工件的根目录中。

## 4. 自定义

### 4.1. 增强或试验ChatAFL

如果对任何模糊器进行了修改，则重新执行`setup.sh`将使用修改后的版本重建所有图像。所有提供的ChatAFL版本都包含一个Dockerfile，允许在与主题相同的环境中检查构建失败，并具有清晰的图像，可以在其中设置不同的主题。

### 4.2. 调整模糊器参数

实验中使用的所有参数都位于`config.h`和`chat-llm.h`中。特定于ChatAFL的参数为：

在 `config.h`:
* EPSILON_CHOICE      
* UNINTERESTING_THRESHOLD 
* CHATTING_THRESHOLD  

在 `chat-llm.h`:
* STALL_RETRIES
* GRAMMAR_RETRIES
* MESSAGE_TYPE_RETRIES
* ENRICHMENT_RETRIES
* MAX_ENRICHMENT_MESSAGE_TYPES
* MAX_ENRICHMENT_CORPUS_SIZE

### 4.3. 添加新的sut

要添加额外的sut，我们参考[ProfuzzBench](https://github.com/profuzzbench/profuzzbench#1-how-do-i-extend-profuzzbench)提供的添加新基准sut的说明。例如，我们添加了Lighttpd1.4作为基准测试的新sut。

### 4.4. 故障排除

如果模糊器以错误终止，则会显示一条过早的"I are done"消息。要检查此问题，运行`docker logs<containerID>`将显示失败容器的日志。

## 5. 局限性

目前的工件与OpenAI的大型语言模型（`davinci-003`和`gpt-3.5-turbo`）交互。这对并行化程度提出了第三方限制。此工件中使用的模型的硬限制为每分钟150000个token。

## 6. Special Thanks

We would like to thank the creators of [AFLNet](https://github.com/aflnet/aflnet) and [ProFuzzBench](https://github.com/profuzzbench/profuzzbench) for the tooling and infrastructure they have provided to the community.

## 7. License
This artifact is licensed under the Apache License 2.0 - see the [LICENSE](./LICENSE) file for details.

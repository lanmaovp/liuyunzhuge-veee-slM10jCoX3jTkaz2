
# Semantic Kernel简介


玩过大语言模型（LLM）的都知道OpenAI，然后微软Azure也提供了OpenAI的服务：Azure OpenAI，只需要申请到API Key，就可以使用这些AI服务。使用方式可以是通过在线Web页面直接与AI聊天，也可以调用AI的API服务，将AI的能力集成到自己的应用程序中。不过这些服务都是在线提供的，都会需要根据token计费，所以不仅需要依赖互联网，而且在使用上会有一定成本。于是，就出现了像Ollama这样的本地大语言模型服务，只要你的电脑足够强悍，应用场景允许的情况下，使用本地大语言模型也是一个不错的选择。


既然有这么多AI服务可以选择，那如果在我的应用程序中需要能够很方便地对接不同的AI服务，应该怎么做呢？这就是Semantic Kernel的基本功能，它是一个基于大语言模型开发应用程序的框架，可以让你的应用程序更加方便地集成大语言模型。Semantic Kernel可用于轻松生成 AI 代理并将最新的 AI 模型集成到 C\#、Python 或 Java 代码库中。因此，它虽然在.NET AI生态中扮演着非常重要的角色，但它是支持多编程语言跨平台的应用开发套件。


Semantic Kernel主要包含以下这些核心概念：


1. **连接（Connection）**：与外部 AI 服务和数据源交互，比如在应用程序中实现Open AI和Ollama的无缝整合
2. **插件（Plugins）**：封装应用程序可以使用的功能，比如增强提示词功能，为大语言模型提供更多的上下文信息
3. **规划器（Planner）**：根据用户行为编排执行计划和策略
4. **内存（Memory）**：抽象并简化 AI 应用程序的上下文管理，比如文本向量（Text Embedding）的存储等


有关Semantic Kernel的具体介绍可以参考[微软官方文档](https://github.com)。


# 演练：通过Semantic Kernel使用Microsoft Azure OpenAI Service


话不多说，直接实操。这个演练的目的，就是使用部署在Azure上的gpt\-4o大语言模型来实现一个简单的问答系统。



> 微软于2024年10月21日终止面向个人用户的Azure OpenAI服务，企业用户仍能继续使用。参考：https://finance.sina.com.cn/roll/2024\-10\-18/doc\-incsymyx4982064\.shtml


## 在Azure中部署大语言模型


登录Azure Portal，新建一个Azure AI service，然后点击Go to Azure OpenAI Studio，进入OpenAI Studio：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019094735716-1696478541.png)


进入后，在左侧侧边栏的**共享资源**部分，选择**部署**标签页，然后在**模型部署**页面，点击**部署模型**按钮，在下拉的菜单中，选择**部署基本模型**：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019095118682-1612683672.png)


在**选择模型**对话框中，选择**gpt\-4o**，然后点击**确认**按钮：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019095351166-1971850332.png)


在弹出的对话框**部署模型 gpt\-4o**中，给模型取个名字，然后直接点击**部署**按钮，如果希望对模型版本、安全性等做一些设置，也可以点击**自定义**按钮展开选项。


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019095657141-892139478.png)


部署成功后，就可以在模型部署页面的列表中看到已部署模型的版本以及状态：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019095755494-934373205.png)


点击新部署的模型的名称，进入模型详细信息页面，在页面的**终结点**部分，把**目标URI**和**密钥**复制下来，待会要用。目标URI只需要复制主机名部分即可，比如https://qingy\-m2e0gbl3\-eastus.openai.azure.com这样：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019101325281-1631094361.png)


## 在C\#中使用Semantic Kernel实现问答应用


首先创建一个控制台应用程序，然后添加`Microsoft.SemanticKernel` NuGet包的引用：



```


|  | $ dotnet new console --name ChatApp |
| --- | --- |
|  | $ dotnet add package Microsoft.SemanticKernel |


```

然后编辑Program.cs文件，加入下面的代码：



```


|  | using Microsoft.SemanticKernel; |
| --- | --- |
|  | using Microsoft.SemanticKernel.ChatCompletion; |
|  | using System.Text; |
|  |  |
|  | var apikey = Environment.GetEnvironmentVariable("azureopenaiapikey")!; |
|  |  |
|  | // 初始化Semantic Kernel |
|  | var kernel = Kernel.CreateBuilder() |
|  | .AddAzureOpenAIChatCompletion( |
|  | "gpt-4", |
|  | "https://qingy-m2e0gbl3-eastus.openai.azure.com", |
|  | apikey) |
|  | .Build(); |
|  |  |
|  | // 创建一个对话完成服务以及对话历史对象，用来保存对话历史，以便后续为大模型 |
|  | // 提供对话上下文信息。 |
|  | var chatCompletionService = kernel.GetRequiredService(); |
|  | var chat = new ChatHistory("你是一个AI助手，帮助人们查找信息和回答问题"); |
|  | StringBuilder chatResponse = new(); |
|  |  |
|  | while (true) |
|  | { |
|  | Console.Write("请输入问题>> "); |
|  |  |
|  | // 将用户输入的问题添加到对话中 |
|  | chat.AddUserMessage(Console.ReadLine()!); |
|  |  |
|  | chatResponse.Clear(); |
|  |  |
|  | // 获取大语言模型的反馈，并将结果逐字输出 |
|  | await foreach (var message in |
|  | chatCompletionService.GetStreamingChatMessageContentsAsync(chat)) |
|  | { |
|  | // 输出当前获取的结果字符串 |
|  | Console.Write(message); |
|  |  |
|  | // 将输出内容添加到临时变量中 |
|  | chatResponse.Append(message.Content); |
|  | } |
|  |  |
|  | Console.WriteLine(); |
|  |  |
|  | // 在进入下一次问答之前，将当前回答结果添加到对话历史中，为大语言模型提供问答上下文 |
|  | chat.AddAssistantMessage(chatResponse.ToString()); |
|  |  |
|  | Console.WriteLine(); |
|  | } |


```

在上面的代码中，需要将你的API Key和终结点URI配置进去，为了安全性，这里我使用环境变量保存API Key，然后由程序读入。为了让大语言模型能够了解在一次对话中，我和它之间都讨论了什么内容，在代码中，使用一个`StringBuilder`临时保存了当前对话的应答结果，然后将这个结果又通过Semantic Kernel的`AddAssistantMessage`方法加回到对话中，以便在下一次对话中，大语言模型能够知道我们在聊什么话题。


比如下面的例子中，在第二次提问时我问到“有那几次迁徙？”，AI能知道我是在说人类历史上的大迁徙，然后将我想要的答案列举出来：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019110417401-1408588468.png)


到这里，一个简单的基于gpt\-4o的问答应用就完成了，它的工作流程大致如下：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019125710271-243506316.png)


## AI能回答所有的问题吗？


由于这里使用的gpt\-4o大语言模型是在今年5月份发布的，而大语言模型都是基于现有数据经过训练得到的，所以，它应该不会知道5月份以后的事情，遇到这样的问题，AI只能回答不知道，或者给出一个比较离谱的答案：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019124405516-1871766494.png)


你或许会想，那我将这些知识或者新闻文章下载下来，然后基于上面的代码，将这些信息先添加到对话历史中，让大语言模型能够了解上下文，这样回答问题的时候准确率不是提高了吗？这个思路是对的，可以在进行问答之前，将新闻的文本信息添加到对话历史中：



```


|  | chat.AddUserMessage("这是一些额外的信息：" + await File.ReadAllTextAsync("input.txt")); |
| --- | --- |


```

但是这样做，会造成下面的异常信息：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019130309338-1436712519.png)


这个问题其实就跟大语言模型的Context Window有关。当今所有的大语言模型在一次数据处理上都有一定的限制，这个限制就是Context Window，在这个例子中，我们的模型一次最多处理12万8千个token（token是大语言模型的数据处理单元，它可以是一个词组，一个单词或者是一个字符），而我们却输入了147,845个token，于是就报错了。很明显，我们应该减少传入的数据量，但这样又没办法把完整的新闻文章信息发送给大语言模型。此时就要用到“检索增强生成（RAG）”。


# Semantic Kernel的检索增强生成（RAG）实践


 其实，并不一定非要把整篇新闻文章发给大语言模型，可以换个思路：只需要在新闻文章中摘出跟提问相关的内容发送给大语言模型就可以了，这样就可以大大减小需要发送到大语言模型的token数量。所以，这里就出现了额外的一些步骤：


1. 对大量的文档进行预处理，将文本信息量化并保存下来（Text Embedding）
2. 在提出新问题时，根据问题语义，从保存的文本量化信息（Embeddings）中，找到与问题相关的信息
3. 将这些信息发送给大语言模型，并从大语言模型获得应答
4. 将结果反馈给调用方


流程大致如下：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019155438284-1147269457.png)


虚线灰色框中就是**检索增强生成（RAG）**相关流程，这里就不针对每个标号一一说明了，能够理解上面所述的4个大的步骤，就很好理解这张图中的整体流程。下面我们直接使用Semantic Kernel，通过RAG来增强模型应答。


首先，在Azure OpenAI Studio中，按照上文的步骤，部署一个text\-embedding\-3\-small的模型，同样将终结点URI和API Key记录下来，然后，在项目中添加`Microsoft.SemanticKernel.Plugins.Memory` NuGet包的引用，因为我们打算先使用基于内存的文本向量数据库来运行我们的代码。Semantic Kernel支持多种向量数据库，比如Sqlite，Azure AI Search，Chroma，Milvus，Pinecone，Qdrant，Weaviate等等。在添加引用的时候，需要使用`--prerelease`参数，因为`Microsoft.SemanticKernel.Plugins.Memory`包目前还处于alpha阶段。


将上面的代码改成下面的形式：



```


|  | using Microsoft.SemanticKernel; |
| --- | --- |
|  | using Microsoft.SemanticKernel.ChatCompletion; |
|  | using System.Text; |
|  | using Microsoft.SemanticKernel.Connectors.AzureOpenAI; |
|  | using Microsoft.SemanticKernel.Memory; |
|  | using Microsoft.SemanticKernel.Text; |
|  |  |
|  | #pragma warning disable SKEXP0010, SKEXP0001, SKEXP0050 |
|  |  |
|  | const string CollectionName = "LatestNews"; |
|  |  |
|  | var apikey = Environment.GetEnvironmentVariable("azureopenaiapikey")!; |
|  |  |
|  | // 初始化Semantic Kernel |
|  | var kernel = Kernel.CreateBuilder() |
|  | .AddAzureOpenAIChatCompletion( |
|  | "gpt-4", |
|  | "https://qingy-m2e0gbl3-eastus.openai.azure.com", |
|  | apikey) |
|  | .Build(); |
|  |  |
|  | // 创建文本向量生成服务 |
|  | var textEmbeddingGenerationService = new AzureOpenAITextEmbeddingGenerationService( |
|  | "text-embedding-3-small", |
|  | "https://qingy-m2e0gbl3-eastus.openai.azure.com", |
|  | apikey); |
|  |  |
|  | // 创建用于保存文本向量的内存向量数据库 |
|  | var memory = new MemoryBuilder() |
|  | .WithMemoryStore(new VolatileMemoryStore()) |
|  | .WithTextEmbeddingGeneration(textEmbeddingGenerationService) |
|  | .Build(); |
|  |  |
|  | // 从外部文件以Markdown格式读入内容，然后根据语义产生多个段落 |
|  | var markdownContent = await File.ReadAllTextAsync(@"input.md"); |
|  | var paragraphs = |
|  | TextChunker.SplitMarkdownParagraphs( |
|  | TextChunker.SplitMarkDownLines(markdownContent.Replace("\r\n", " "), 128), |
|  | 64); |
|  |  |
|  | // 将各个段落进行量化并保存到向量数据库 |
|  | for (var i = 0; i < paragraphs.Count; i++) |
|  | { |
|  | await memory.SaveInformationAsync(CollectionName, paragraphs[i], $"paragraph{i}"); |
|  | } |
|  |  |
|  | // 创建一个对话完成服务以及对话历史对象，用来保存对话历史，以便后续为大模型 |
|  | // 提供对话上下文信息。 |
|  | var chatCompletionService = kernel.GetRequiredService(); |
|  | var chat = new ChatHistory("你是一个AI助手，帮助人们查找信息和回答问题"); |
|  | StringBuilder additionalInfo = new(); |
|  | StringBuilder chatResponse = new(); |
|  |  |
|  | while (true) |
|  | { |
|  | Console.Write("请输入问题>> "); |
|  | var question = Console.ReadLine()!; |
|  | additionalInfo.Clear(); |
|  |  |
|  | // 从向量数据库中找到跟提问最为相近的3条信息，将其添加到对话历史中 |
|  | await foreach (var hit in memory.SearchAsync(CollectionName, question, limit: 3)) |
|  | { |
|  | additionalInfo.AppendLine(hit.Metadata.Text); |
|  | } |
|  | var contextLinesToRemove = -1; |
|  | if (additionalInfo.Length != 0) |
|  | { |
|  | additionalInfo.Insert(0, "以下是一些附加信息："); |
|  | contextLinesToRemove = chat.Count; |
|  | chat.AddUserMessage(additionalInfo.ToString()); |
|  | } |
|  |  |
|  | // 将用户输入的问题添加到对话中 |
|  | chat.AddUserMessage(question); |
|  |  |
|  | chatResponse.Clear(); |
|  | // 获取大语言模型的反馈，并将结果逐字输出 |
|  | await foreach (var message in |
|  | chatCompletionService.GetStreamingChatMessageContentsAsync(chat)) |
|  | { |
|  | // 输出当前获取的结果字符串 |
|  | Console.Write(message); |
|  |  |
|  | // 将输出内容添加到临时变量中 |
|  | chatResponse.Append(message.Content); |
|  | } |
|  |  |
|  | Console.WriteLine(); |
|  |  |
|  | // 在进入下一次问答之前，将当前回答结果添加到对话历史中，为大语言模型提供问答上下文 |
|  | chat.AddAssistantMessage(chatResponse.ToString()); |
|  |  |
|  | // 将当次问题相关的内容从对话历史中移除 |
|  | if (contextLinesToRemove >= 0) chat.RemoveAt(contextLinesToRemove); |
|  |  |
|  | Console.WriteLine(); |
|  | } |


```

重新运行程序，然后提出同样的问题，可以看到，现在的答案就正确了：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019162726504-1814128856.png)


 现在看看向量数据库中到底有什么。新添加一个对`Microsoft.SemanticKernel.Connectors.Sqlite` NuGet包的引用，然后，将上面代码的：



```


|  | .WithMemoryStore(new VolatileMemoryStore()) |
| --- | --- |


```

改为：



```


|  | .WithMemoryStore(await SqliteMemoryStore.ConnectAsync("vectors.db")) |
| --- | --- |


```

重新运行程序，执行成功后，在`bin\Debug\net8.0`目录下，可以找到`vectors.db`文件，用Sqlite查看工具（我用的是SQLiteStudio）打开数据库文件，可以看到下面的表和数据：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019163625198-1795703256.png)


Metadata字段保存的就是每个段落的原始数据信息，而Embedding字段则是文本向量，其实它就是一系列的浮点值，代表着文本之间在语义上的**距离**。


# 使用基于Ollama的本地大语言模型


Semantic Kernel现在已经可以[支持Ollama本地大语言模型](https://github.com)了，虽然它目前也还是预览版。可以在项目中通过添加`Microsoft.SemanticKernel.Connectors.Ollama` NuGet包来体验。建议安装最新版本的[Ollama](https://github.com):[slower加速器](https://jisuanqi.org)，然后，下载两个大语言模型，一个是Chat Completion类型的，另一个是Text Embedding类型的。我选择了`llama3.2:3b`和`mxbai-embed-large`这两个模型：


![](https://img2024.cnblogs.com/blog/119825/202410/119825-20241019170516121-899127636.png)


代码上只需要将Azure OpenAI替换为Ollama即可：



```


|  | // 初始化Semantic Kernel |
| --- | --- |
|  | var kernel = Kernel.CreateBuilder() |
|  | .AddOllamaChatCompletion( |
|  | "llama3.2:3b", |
|  | new Uri("http://localhost:11434")) |
|  | .Build(); |
|  |  |
|  | // 创建文本向量生成服务 |
|  | var textEmbeddingGenerationService = new OllamaTextEmbeddingGenerationService( |
|  | "mxbai-embed-large:latest", |
|  | new Uri("http://localhost:11434")); |


```

# 总结


通过本文的介绍，应该可以对Semantic Kernel、RAG以及在C\#中的应用有一定的了解，虽然没有涉及原理性的内容，但基本已经可以在应用层面上提供一定的参考价值。Semantic Kernel虽然有些Plugins还处于预览阶段，但通过本文的介绍，我们已经可以看到它的强大功能，比如，允许我们很方便地接入各种流行的向量数据库，也允许我们很方便地切换到不同的AI大语言模型服务，在AI的应用集成上，Semantic Kernel发挥着重要的作用。


# 参考


本文部分内容参考了微软官方文档《[Demystifying Retrieval Augmented Generation with .NET](https://github.com)》，代码也部分参考了文中内容。文章介绍得更为详细，建议有兴趣的读者移步阅读。



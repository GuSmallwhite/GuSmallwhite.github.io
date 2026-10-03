---
title: "langchain之model层代码阅读"
date: "2023-11-19 12:00:00"
permalink: "2023/11/19/langchain之model层代码阅读/"
categories: ["实验室", "rpc项目"]
tags: ["rpc", "netty"]
---

## 前言

实验室的东西，我来阅读一下这个model层代码，要接模型

## 流程

GPT给的:

根据提供的代码，整个项目是一个命令行聊天机器人，其主要工作是与用户进行对话，并通过语言模型（Star Coder）生成代码。生成的代码会被提交到一个在线评测工具（OjRunTool）中，以进行编译和运行，然后返回评测结果。项目的主要组成部分包括以下几个类和模块：

1. **ChatBot 主体：**

   * ChatBot 的核心逻辑在 `DefaultExecutor` 类中实现。该类通过调用语言模型、提示模板和在线评测工具，与用户进行交互，生成和评测代码。
   * 通过 Flux 发布订阅模式，实现异步处理消息，包括用户消息、系统消息和代码执行结果。
2. **语言模型（Star Coder）：**

   * 语言模型的实现在 `StarCoderLlm` 类中。该类接收用户的描述性问题，调用远程服务获取生成的代码，并将生成的代码和助手消息返回。
3. **提示模板（StarCoderPromptTemplate）：**

   * 提示模板用于生成语言模型的输入提示。在 `StarCoderPromptTemplate` 类中，根据历史消息生成一个包含系统消息、用户消息和助手消息的提示字符串。
4. **在线评测工具（OjRunTool）：**

   * 在线评测工具通过 HTTP 请求将生成的代码提交给一个在线评测服务，并获取评测结果。在 `OjRunTool` 类中，该工具负责处理 HTTP 请求和解析评测结果，同时包含了错误码映射，用于将评测状态映射为 `OjEnum` 枚举值。
5. **HTTP 请求工具（HttpUtil）：**

   * `HttpUtil` 类用于发送 HTTP 请求，其中包含一个用于执行 POST 请求的方法。

整个项目的工作流程可以总结为以下步骤：

1. 用户提供一个问题描述，ChatBot 接收并处理该问题。
2. ChatBot 调用语言模型生成相应的代码。
3. ChatBot 根据历史消息和系统消息，使用提示模板生成语言模型的输入提示。
4. ChatBot 将生成的代码和提示传递给在线评测工具（OjRunTool）。
5. OjRunTool 发送 HTTP 请求将代码提交给在线评测服务，获取评测结果。
6. ChatBot 处理评测结果，生成相应的系统消息或用户消息，与用户进行交互。
7. 通过 Flux 发布订阅模式，异步处理消息的传递和显示。

整体上，项目采用了异步的、模块化的设计，通过不同的组件协同工作，实现了一个基于聊天的命令行代码生成和评测机器人。

## Executor包

这里主要就是执行逻辑，学长发话了，先看看这个executor包

### BaseExecutor

```
public abstract class BaseExecutor {

    public abstract void run(Map<String,Object> args);
}
```

就是一个普通的接口，args里面存放的run的需要的参数，比如指定了什么模型，什么提示之类的

### DefaultExecutor

重点来了，这是默认的执行者

先来看总体的代码：

```
public class DefaultExecutor extends BaseExecutor{
    private static final Map<String, BaseLlm> MODEL_POOL=new HashMap<String,BaseLlm>(){{
        put("StarCoder",new StarCoderLlm());
    }};

    private static final Map<String, BasePromptTemplate> PROMPT_TEMPLATE_MAP=new HashMap<String,BasePromptTemplate>()
    {{
        put("StarCoder",new StarCoderPromptTemplate());
    }};


    private static final Integer MAX_ROUND=3;


    @Override
    public void run(Map<String, Object> args) {
        BaseChatMessageHistory memory= (BaseChatMessageHistory) args.get("memory");
        QuestionBO questionBO= (QuestionBO) args.get("question");
        String modelName=(String)args.get("modelName");
        FluxSink<BaseMessage> emit=(FluxSink) args.get("emit");

        HumanMessage humanMessage=new HumanMessage("Description:"+questionBO.getDescription(),Collections.emptyMap());
        memory.addMessage(humanMessage);
        emit.next(humanMessage);

        BaseLlm model=MODEL_POOL.get(modelName);
        BasePromptTemplate promptTemplate=PROMPT_TEMPLATE_MAP.get(modelName);

        for(int i=0;i<MAX_ROUND;i++){
            String prompt=promptTemplate.parsePrompt(memory.loadMessages());
            Map<String,Object> llmResponse=model.call(prompt, Collections.emptyMap());

            OjRunTool ojRunTool=new OjRunTool();
            Map<String,Object> input=new HashMap<>();
            input.put("question",questionBO);
            input.put("code",llmResponse.get("code"));
            BaseMessage message= (BaseMessage) llmResponse.get("msg");
            emit.next(message);
            memory.addMessage(message);

            Map<String,Object> toolOutput=ojRunTool.run(input);
            OjEnum ojRes= (OjEnum) toolOutput.get("OjRes");
            if(ojRes.equals(OjEnum.PASS)){
                Map<String,Object> extendedParam=new HashMap<>();
                extendedParam.put("statusCode",0);
                SystemMessage systemMessage=new SystemMessage(ojRes.getMessage(), extendedParam);
                emit.next(systemMessage);
                emit.complete();
                break;
            }
            else{
                HumanMessage ojMessage=new HumanMessage(ojRes.getMessage(), Collections.emptyMap());
                emit.next(ojMessage);
                memory.addMessage(ojMessage);
            }
        }
        Map<String,Object> extendedParam=new HashMap<>();
        extendedParam.put("statusCode",1);
        SystemMessage systemMessage=new SystemMessage("The model cannot pass the oj",extendedParam);
        emit.next(systemMessage);
        emit.complete();
    }
}
```

再分下来细说：

### 初始化部分

```
private static final Map<String, BaseLlm> MODEL_POOL = new HashMap<String, BaseLlm>() {{
    put("StarCoder", new StarCoderLlm());
}};
```

MODEL\_POOL，存入可以用来调用的模型，目前只有StarCoder

```
private static final Map<String, BasePromptTemplate> PROMPT_TEMPLATE_MAP = new HashMap<String, BasePromptTemplate>() {{
    put("StarCoder", new StarCoderPromptTemplate());
}};
```

PROMPT\_TEMPLATE\_MAP，存放不同模型对应的提示（prompt

```
private static final Integer MAX_ROUND = 3;
```

最大轮数，就是你这个模型，这个run，run多少轮

### run函数

#### **获取参数信息:**

```
BaseChatMessageHistory memory = (BaseChatMessageHistory) args.get("memory");//获取聊天历史
QuestionBO questionBO = (QuestionBO) args.get("question");//获取问题
String modelName = (String) args.get("modelName");//获取模型名字
FluxSink<BaseMessage> emit = (FluxSink) args.get("emit");//获取FLUXSINK
```

还记得`args`吗，他就是一个map，存入了基本信息

从传递给方法的 `args` 参数中获取了执行所需的信息，包括聊天消息历史 (`memory`)、问题 (`questionBO`)、模型名称 (`modelName`) 和用于发射消息的 `FluxSink` (`emit`)。

`FluxSink` 是什么？

> `FluxSink` 是 Reactor 框架中的一个接口，它提供了一种用于向响应式流中发射（emit）数据的机制。Reactor 是一个用于构建响应式应用程序的库，而 `FluxSink` 则是用于在生产者（Publisher）和订阅者（Subscriber）之间传递数据的一种工具。
>
> 具体而言，`FluxSink` 接口具有用于向数据流中发送数据项的方法，包括：
>
> * `next(T element)`：发射一个数据项给订阅者。
> * `complete()`：发射完成信号给订阅者，表示数据流结束。
> * `error(Throwable error)`：发射错误信号给订阅者，表示在数据流中发生了错误。
>
> 在上下文中，代码中使用 `FluxSink<BaseMessage>` 类型的对象 `emit`，通过 `emit.next(message)` 来发射生成的消息给订阅者。这允许执行器类与外部的响应式流进行交互，将生成的消息推送给订阅者，即外部的响应式处理逻辑。

OK，先了解这么多，然后我再单独写博客也行

#### **创建并发射问题描述消息:**

```
HumanMessage humanMessage = new HumanMessage("Description:" + questionBO.getDescription(), Collections.emptyMap());
memory.addMessage(humanMessage);
emit.next(humanMessage);
```

* 创建了一个人类消息 (`HumanMessage`)，其中包含了问题的描述信息。
* 将该消息添加到聊天消息历史 (`memory`) 中。
* 使用 `emit.next(humanMessage)` 发射该消息给订阅者。~~虽然我还不知道订阅者是谁？~~

#### **获取语言模型和提示模板:**

```
BaseLlm model = MODEL_POOL.get(modelName);
BasePromptTemplate promptTemplate = PROMPT_TEMPLATE_MAP.get(modelName);
```

* 通过模型名称 (`modelName`) 从模型池 (`MODEL_POOL`) 中获取相应的语言模型 (`BaseLlm`)。
* 通过模型名称 (`modelName`) 从提示模板映射 (`PROMPT_TEMPLATE_MAP`) 中获取相应的提示模板 (`BasePromptTemplate`)。

#### **迭代执行:**

```
for (int i = 0; i < MAX_ROUND; i++) {
    // 代码省略...
}
```

#### 循环体内：生成提示和响应

```
String prompt = promptTemplate.parsePrompt(memory.loadMessages());
Map<String, Object> llmResponse = model.call(prompt, Collections.emptyMap());
```

* 使用提示模板 (`promptTemplate`) 生成提示。
* 调用语言模型 (`model`) 生成响应。

#### **使用在线评测器运行代码:**

把模型生成的代码丢给oj，让他判断对不对

```
OjRunTool ojRunTool = new OjRunTool();
Map<String, Object> input = new HashMap<>();//丢给runtool的输入
input.put("question", questionBO);//问题是什么
input.put("code", llmResponse.get("code"));//把LLM返回的代码也放进去
BaseMessage message = (BaseMessage) llmResponse.get("msg");
emit.next(message);//什么勾八
memory.addMessage(message);//把message放到memory里面，其实这里实例化的message对象是BaseChatMessageHistory
Map<String, Object> toolOutput = ojRunTool.run(input);
```

* 创建 `OjRunTool` 实例。
* 准备运行所需的输入信息。
* 发射生成的消息，将消息添加到聊天历史 (`memory`) 中。BaseChatMessageHistory
* 使用在线评测工具 (`OjRunTool`) 运行生成的代码，并获取工具的输出。

#### **处理在线评测结果:**

```
if(ojRes.equals(OjEnum.PASS)){
                Map<String,Object> extendedParam=new HashMap<>();
                extendedParam.put("statusCode",0);
                SystemMessage systemMessage=new SystemMessage(ojRes.getMessage(), extendedParam);
                emit.next(systemMessage);
                emit.complete();
                break;
            }
            else{
                HumanMessage ojMessage=new HumanMessage(ojRes.getMessage(), Collections.emptyMap());
                emit.next(ojMessage);
                memory.addMessage(ojMessage);
            }
```

* 如果评测结果为通过，创建一个包含扩展参数的 `HashMap`，设置 `statusCode` 为 0。
* 创建一个通过的系统消息 (`SystemMessage`)，将消息和扩展参数传递给构造函数。
* 使用 `emit.next(systemMessage)` 发射通过的系统消息给订阅者。
* 使用 `emit.complete()` 发射完成信号，表示数据流结束。
* 使用 `break` 退出循环，因为已经达到了预期的评测结果。
* 如果评测结果未通过，创建一个人类消息 (`HumanMessage`)，将消息和一个空的 `HashMap` 传递给构造函数。**问题：为什么这里返回的是Humanmessage**
* 使用 `emit.next(ojMessage)` 发射未通过的人类消息给订阅者。
* 将未通过的人类消息添加到聊天历史 (`memory`) 中。

#### **结束执行**

```
Map<String, Object> extendedParam = new HashMap<>();
extendedParam.put("statusCode", 1);
SystemMessage systemMessage = new SystemMessage("The model cannot pass the oj", extendedParam);
emit.next(systemMessage);
emit.complete();
```

* 生成未通过的系统消息，发射消息并完成执行。

  问题：这样岂不是所有情况都得发射一次？

## memory包

这里主要存放了memory，包含人类消息，ai消息和系统消息，聊天历史等等

### BaseMessage

一个抽象类，是所有message的爹

```
public abstract class BaseMessage {
    private final String content;//内容

    private final Map<String,Object> extendParams//这个是为后面扩展功能做的接口

    public BaseMessage(String content,Map<String,Object> extendParams){
        this.content=content;
        this.extendParams=extendParams;
    }


    public abstract String type();

    public String getContent(){
        return this.content;
    }

    public Map<String,Object> getExtendParams(){
        return this.extendParams;
    }

    public Object getExtendParam(String key){
        return this.extendParams.get(key);
    }

}
```

### Humanmessage

继承了basemessage，然后没啥变化

```
public class HumanMessage extends BaseMessage {
    public HumanMessage(String content, Map<String,Object> extendParams){
        super(content,extendParams);
    }
    @Override
    public String type() {
        return "human";//告诉别人我是什么类型的
    }
}
```

### systemmessage

其实就是oj的消息

```
public class SystemMessage extends BaseMessage{
    public SystemMessage(String content, Map<String, Object> extendParams) {
        super(content, extendParams);
    }

    @Override
    public String type() {
        return "system";
    }
}
```

### AiMessage

```
public class AiMessage extends BaseMessage{
    public AiMessage(String content, Map<String, Object> extendParams) {
        super(content, extendParams);
    }

    @Override
    public String type() {
        return "ai";
    }
}
```

只是表面他是ai

### BaseChatMessageHistory

聊天历史

```
public abstract class BaseChatMessageHistory {
    void addUserMessage(String content){
        HumanMessage humanMessage=new HumanMessage(content,null);
        this.addMessage(humanMessage);
    }

    void addAiMessage(String content){
        AiMessage aiMessage=new AiMessage(content,null);
        this.addMessage(aiMessage);
    }

    public abstract void clear();

    public abstract List<BaseMessage> loadMessages();

    public abstract void addMessage(BaseMessage message);
}
```

管理聊天记录，包含添加用户和ai信息的方法，以及导入message的方法

## LLM包

包含调用大模型去运行的代码

### BaseLlm

一个抽象类，是所有llm的爹，目测还要接别的llm

```
public abstract class BaseLlm {
    public abstract Map<String,Object> call(String prompt, Map<String,Object> extendedParams);
}
```

### StarCoderLlm

表示是Starcoder模型，然后后面也可以接别的

#### 初始化部分

```
private static final Logger logger = Logger.getLogger(StarCoderLlm.class);
private static final OkHttpClient client = new OkHttpClient().newBuilder()
        .connectTimeout(180, TimeUnit.SECONDS)
        .writeTimeout(180, TimeUnit.SECONDS)
        .readTimeout(180, TimeUnit.SECONDS)
        .build();
```

* `logger` 用于记录日志。
* `client` 是 OkHttpClient 的实例，用于进行 HTTP 请求。在这里，设置了连接、写入和读取的超时时间。

**OKHTTP是什么勾八，然后再说**

> OkHttp 是一个用于在 Java 和 Android 应用程序中进行网络请求的开源 HTTP 客户端库。它由 Square 公司开发，并被广泛用于 Android 应用的网络通信以及 Java 应用的后端通信。以下是 OkHttp 的一些主要特点和用途：

#### call方法

##### 参数

```
public Map<String, Object> call(String prompt, Map<String, Object> extendedParams) {
    // 代码省略...
}
```

* 实现了 `BaseLlm` 类中的抽象方法 `call`。
* 该方法接受一个提示 (`prompt`) 和扩展参数 (`extendedParams`)，并返回一个包含生成的代码和消息的 `Map`。

**prompt是什么，再说**

##### **HTTP 请求和处理响应:**

```
JSONObject json = new JSONObject();
int promptLength = prompt.length();
json.put("prompt", prompt);

String res;
try {
    res = HttpUtil.doPostRequest(client, "http://10.58.0.2:5904/output", json.toJSONString());
    logger.debug("starcoder res:" + res);
} catch (IOException e) {
    logger.error(e.getMessage());
    throw new RuntimeException(e);
}
```

* 创建一个 JSON 对象，将提示放入其中。
* 使用 `HttpUtil.doPostRequest` 方法进行 HTTP POST 请求，将 JSON 对象发送到指定的 URL。
* 捕获可能抛出的 `IOException` 异常，记录错误日志，并将异常重新抛出为 `RuntimeException`。

**HTTPUTIL**再说

##### **解析响应和提取生成的代码**

```
Map<String, Object> responseMap = JSON.parseObject(res, new TypeReference<Map<String, Object>>() {});//将响应解析为map
String llmResponse = (String) responseMap.get("text");
String codeGenerate;
if (llmResponse.contains("```")) {
    codeGenerate = llmResponse.substring(promptLength).split("```")[1].substring(3);
} else {
    codeGenerate = getMessageContent(llmResponse.substring(promptLength));
}
logger.debug("starcoder codeGenerate:" + codeGenerate);
```

* 使用 `JSON.parseObject` 解析 HTTP 响应字符串为 `Map`。
* 从响应中提取模型返回的文本 (`text`)。
* 根据文本中是否包含 “```” 来判断是否有生成的代码，如果有，提取代码；否则，调用 `getMessageContent` 方法获取消息内容。

为什么能这么判断呢？

**学长曰：模型是这样吐出的，我现放着看看**

##### **构造并返回结果 `Map`**

```
Map<String, Object> resMap = new HashMap<>();
resMap.put("code", codeGenerate);
String msgContent = getMessageContent(llmResponse.substring(promptLength));
AiMessage aiMessage = new AiMessage(msgContent, Collections.emptyMap());
resMap.put("msg", aiMessage);

return resMap;
```

* 创建一个包含生成代码和消息的 `Map`。
* 将生成的代码放入 `code` 键中。
* 通过调用 `getMessageContent` 获取消息内容，并将消息放入 `msg` 键中。
* 返回构造好的 `Map`。

**问题：为什么是promptlength来sub**

答：`promptLength` 表示提示的长度，即在模型输入时的提示内容的长度。这是因为生成的文本中，代码块的内容通常从提示内容后开始。

##### **`getMessageContent` 方法**

```
private static String getMessageContent(String text) {
    String tempStr = text.substring(text.indexOf(StarCoderPromptTemplate.ASSISTANT_TOKEN) + 1,
            text.lastIndexOf(StarCoderPromptTemplate.END_TOKEN));
    return tempStr;
}
```

用于从模型返回的文本中提取消息内容

**问题：msgContent和codeGenerate有什么区别**

\*\*答：有的，区别就是codegennerate没有前面的c++\*\*和前后的```

## prompt包

### BasePromptTemplate

就一个方法，就是生成提示

```
public abstract class BasePromptTemplate {
    public abstract String parsePrompt(List<BaseMessage> messages);
}
```

### StarCoderPromptTemplate

Starcoder对应的这个东西

先来看整体代码:

```
public class StarCoderPromptTemplate extends BasePromptTemplate{
    public static final String SYS_TOKEN="<|system|>";
    public static final String USER_TOKEN="<|user|>";
    public static final String ASSISTANT_TOKEN="<|assistant|>";
    public static final String END_TOKEN="<|end|>";

    public static final String SYS_MSG="";


    @Override
    public String parsePrompt(List<BaseMessage> messages) {
        StringBuffer promptBuffer=new StringBuffer();
        if(StringUtils.isNotBlank(SYS_MSG)){
            promptBuffer.append(SYS_TOKEN+"\n" + SYS_MSG + END_TOKEN + "\n");
        }
        if(messages.isEmpty()){
            throw new RuntimeException("Message List Cannot be empty");
        }
        for(BaseMessage message:messages){
            if(message.type().equals("ai")){
                promptBuffer.append(ASSISTANT_TOKEN+"\n" + message.getContent() + END_TOKEN + "\n");
            }
            else if(message.type().equals("human")){
                promptBuffer.append(USER_TOKEN + "\n" + message.getContent() + END_TOKEN + "\n");
            }
        }
        promptBuffer.append(ASSISTANT_TOKEN);


        return promptBuffer.toString();
    }
}
```

### **常量定义**

```
public static final String SYS_TOKEN="<|system|>";
public static final String USER_TOKEN="<|user|>";
public static final String ASSISTANT_TOKEN="<|assistant|>";
public static final String END_TOKEN="<|end|>";
public static final String SYS_MSG="";
```

* 定义了一些常量，包括系统、用户和助手的标记（TOKEN），以及结束标记（END\_TOKEN）和系统消息（SYS\_MSG）。

### **`parsePrompt` 方法**

```
@Override
public String parsePrompt(List<BaseMessage> messages) {
    StringBuffer promptBuffer = new StringBuffer();

    if (StringUtils.isNotBlank(SYS_MSG)) {
        promptBuffer.append(SYS_TOKEN + "\n" + SYS_MSG + END_TOKEN + "\n");
    }

    if (messages.isEmpty()) {
        throw new RuntimeException("Message List Cannot be empty");
    }

    for (BaseMessage message : messages) {
        if (message.type().equals("ai")) {
            promptBuffer.append(ASSISTANT_TOKEN + "\n" + message.getContent() + END_TOKEN + "\n");
        } else if (message.type().equals("human")) {
            promptBuffer.append(USER_TOKEN + "\n" + message.getContent() + END_TOKEN + "\n");
        }
    }

    promptBuffer.append(ASSISTANT_TOKEN);

    return promptBuffer.toString();
}
```

* 实现了 `BasePromptTemplate` 类中的抽象方法 `parsePrompt`。
* 该方法接受一个包含聊天消息的列表 `messages`，并返回一个表示提示的字符串。
* 首先检查是否存在系统消息 (`SYS_MSG`)，如果有，将系统消息添加到提示字符串中。
* 然后遍历消息列表，根据消息的类型（”ai” 或 “human”）将消息内容添加到提示字符串中。
* 最后，添加助手的标记（ASSISTANT\_TOKEN）作为提示的结束标记。

## Util包

### HttpUtil

一个简单的 `HttpUtil` 类，其中包含一个用于发送 HTTP POST 请求的方法。以下是对这个类和方法的解释：

**静态变量定义**

```
private static final MediaType JSON_TYPE = MediaType.parse("application/json; charset=utf-8");
```

定义了一个静态的 `MediaType` 变量 `JSON_TYPE`，表示请求体的媒体类型为 JSON。

**`doPostRequest` 方法**

```
public static String doPostRequest(OkHttpClient client, String url, String jsonBody) throws IOException {
    RequestBody body = RequestBody.create(JSON_TYPE, jsonBody);

    Request request = new Request.Builder()
            .url(url)
            .post(body)
            .build();

    try (Response response = client.newCall(request).execute()) {
        return response.body().string();
    }
}
```

* 该方法用于发送 HTTP POST 请求，并返回响应的字符串形式。
* 接收一个 `OkHttpClient` 对象用于执行请求，一个表示请求地址的字符串 `url`，和一个表示请求体内容的 JSON 字符串 `jsonBody`。
* 创建一个 `RequestBody` 对象，使用 JSON\_TYPE 来指定请求体的媒体类型，并设置请求体的内容为给定的 JSON 字符串。
* 创建一个 `Request` 对象，指定请求的 URL 和请求体，并使用 `post` 方法表示这是一个 POST 请求。
* 使用 `client.newCall(request).execute()` 发起请求，并得到一个 `Response` 对象。
* 在 `try-with-resources` 语句块中，通过 `response.body().string()` 获取响应体的字符串表示，并将其作为方法的返回值。

## tool包

### OjRunTool

先来看总体代码

```
public class OjRunTool extends BaseTool{
    private static final Logger logger= Logger.getLogger(OjRunTool.class);
    private static final OkHttpClient client = new OkHttpClient().newBuilder()
            .connectTimeout(60, TimeUnit.SECONDS)
            .writeTimeout(60, TimeUnit.SECONDS)
            .readTimeout(60, TimeUnit.SECONDS)
            .build();

    //Oj接口返回code_result_status不为0时对应的错误
    private static final Map<Integer,OjEnum> ERROR_CODE_MAP=new HashMap<Integer,OjEnum>(){{
        put(-1,OjEnum.COMPILE_FAILED);
        put(1,OjEnum.CPU_TIME_LIMIT_EXCEEDED);
        put(2,OjEnum.REAL_TIME_LIMIT_EXCEEDED);
        put(3,OjEnum.MEMORY_LIMIT_EXCEEDED);
        put(4,OjEnum.RUNTIME_ERROR);
        put(5,OjEnum.SYSTEM_ERROR);
    }};


    private static OjEnum runAndCheck(String code, String testCase, String answer){
        JSONObject json=new JSONObject();

        json.put("code","#include <bits/stdc++.h>\n"+code);
        json.put("input",testCase);
        String res;
        try {
            res= HttpUtil.doPostRequest(client,"http://172.29.4.19:8082/modelRacetrack/run", json.toJSONString());
            logger.debug("run res:"+res);
        } catch (IOException e) {
            logger.error(e.getMessage());
            throw new RuntimeException(e);
        }
        Map<String,Object> resMap= JSON.parseObject(res,new TypeReference<Map<String,Object>>(){});
        String response= (String) resMap.get("object");
        Map<String,Object> responseMap= JSON.parseObject(response,new TypeReference<Map<String,Object>>(){});

        Integer status= (Integer) responseMap.get("code_result_status");
        if(ERROR_CODE_MAP.containsKey(status)){
            return ERROR_CODE_MAP.get(status);
        }
        else{
            String output= (String) responseMap.get("output");
            if(output.equals(answer)){
                return OjEnum.PASS;
            }
            else{
                return OjEnum.INCORRECT_ANSWER;
            }
        }
    }


    @Override
    public Map<String, Object> run(Map<String, Object> args) {
        TestcaseBO testcaseBO= (TestcaseBO) args.get("testcase");
        String code= (String) args.get("code");
        OjEnum ojRes=runAndCheck(code,testcaseBO.getInput(), testcaseBO.getOutput());

        logger.debug("ojRes:"+ojRes.name());
        Map<String,Object> resMap=new HashMap<>();
        resMap.put("OjRes",ojRes);
        return resMap;
    }
}
```

### **静态变量定义**

```
private static final Logger logger = Logger.getLogger(OjRunTool.class);
private static final OkHttpClient client = new OkHttpClient().newBuilder()
        .connectTimeout(60, TimeUnit.SECONDS)
        .writeTimeout(60, TimeUnit.SECONDS)
        .readTimeout(60, TimeUnit.SECONDS)
        .build();
```

* `logger` 用于记录日志。
* `client` 是 OkHttpClient 的实例，用于进行 HTTP 请求。在这里，设置了连接、写入和读取的超时时间。

### **错误代码映射**

```
private static final Map<Integer, OjEnum> ERROR_CODE_MAP = new HashMap<Integer, OjEnum>() {{
    put(-1, OjEnum.COMPILE_FAILED);
    put(1, OjEnum.CPU_TIME_LIMIT_EXCEEDED);
    put(2, OjEnum.REAL_TIME_LIMIT_EXCEEDED);
    put(3, OjEnum.MEMORY_LIMIT_EXCEEDED);
    put(4, OjEnum.RUNTIME_ERROR);
    put(5, OjEnum.SYSTEM_ERROR);
}};
```

* 定义了一个映射关系，将 Oj 接口返回的错误代码映射为对应的 `OjEnum` 枚举值。

### **`runAndCheck` 方法**

该方法用于运行代码并检查结果。以下是对该方法的主要解释：

#### **构建请求**

```
JSONObject json = new JSONObject();
json.put("code", "#include <bits/stdc++.h>\n" + code);
json.put("input", testCase);
```

* 创建一个 `JSONObject` 对象，用于构建 HTTP 请求的 JSON 数据。
* 将代码和测试用例放入 JSON 中。

#### **发送 HTTP 请求**

```
try {
    res = HttpUtil.doPostRequest(client, "http://172.29.4.19:8082/modelRacetrack/run", json.toJSONString());
    logger.debug("run res:" + res);
} catch (IOException e) {
    logger.error(e.getMessage());
    throw new RuntimeException(e);
}
```

* 使用 `HttpUtil.doPostRequest` 方法发送 HTTP POST 请求到指定的评测接口。
* 请求内容包括代码和测试用例。
* 捕获可能的 `IOException` 异常，记录错误日志，并将异常重新抛出为运行时异常。

**解析评测结果**

```
Map<String, Object> resMap = JSON.parseObject(res, new TypeReference<Map<String, Object>>() {
});
String response = (String) resMap.get("object");
Map<String, Object> responseMap = JSON.parseObject(response, new TypeReference<Map<String, Object>>() {
});
```

#### **判断状态码**

```
Integer status = (Integer) responseMap.get("code_result_status");
if (ERROR_CODE_MAP.containsKey(status)) {
    return ERROR_CODE_MAP.get(status);
} else {
    // 代码省略...
}
```

#### **判断输出结果**

```
String output = (String) responseMap.get("output");
if (output.equals(answer)) {
    return OjEnum.PASS;
} else {
    return OjEnum.INCORRECT_ANSWER;
}
```

* 如果状态码没有对应的错误，继续判断评测结果的输出与预期答案是否一致。
* 如果一致，返回 `OjEnum.PASS` 表示通过；否则，返回 `OjEnum.INCORRECT_ANSWER` 表示答案错误。

整个方法的作用是向在线评测接口发送 HTTP 请求，运行代码，并根据返回的评测结果判断代码运行状态。这包括了编译失败、超时、内存超限、运行错误等不同的状态。如果评测通过，还会判断输出结果是否与预期答案一致。
# openai-sdk-java —— 统一大模型对接 SDK 设计实现

星球开源共建项目 **《OpenAI SDK》** 统一大模型标准化对接的技术组件项目，此项目以解决实际市面上的场景为诉求，将 OpenAI、Claude、PalM、文心一言、通义千问、讯飞星火、智谱 ChatGLM、腾讯混元等这些大模型做一个统一的 SDK 对接组件。

## 一、发布记录

- 1.2 支持 ChatGPT、ChatGLM、讯飞星火

## 二、安装使用

```pom
<dependency>
    <groupId>cn.bugstack</groupId>
    <artifactId>openai-sdk-java</artifactId>
    <version>1.2</version>
</dependency>
```

- 在系统中引入 POM 并入 ApiTest 创建使用。

```java
@Slf4j
public class ApiTest {

    private OpenAiSession openAiSession;

    @Before
    public void test_OpenAiSessionFactory() throws IOException {
        // 0. 注意；将 resources 包下的 .env 修改为 .env.local 并添加各项配置
        Properties prop = new Properties();
        prop.load(ApiTest.class.getClassLoader().getResourceAsStream(".env.local"));

        // 1. ChatGLM、ChatGPT 配置
        ChatGLMConfig chatGLMConfig = new ChatGLMConfig();
        chatGLMConfig.setApiHost(prop.getProperty("chatglm.config.apihost"));
        chatGLMConfig.setApiSecretKey(prop.getProperty("chatglm.config.apisecretkey"));

        ChatGPTConfig chatGPTConfig = new ChatGPTConfig();
        chatGPTConfig.setApiHost(prop.getProperty("chatgpt.config.apihost"));
        chatGPTConfig.setApiKey(prop.getProperty("chatgpt.config.apikey"));

        XunFeiConfig xunFeiConfig = new XunFeiConfig();
        xunFeiConfig.setApiHost(prop.getProperty("xunfei.config.apihost"));
        xunFeiConfig.setAppid(prop.getProperty("xunfei.config.apiappid"));
        xunFeiConfig.setApiKey(prop.getProperty("xunfei.config.apiapikey"));
        xunFeiConfig.setApiSecret(prop.getProperty("xunfei.config.apiapisecret"));

        // 2. 配置文件
        Configuration configuration = new Configuration();
        configuration.setLevel(HttpLoggingInterceptor.Level.HEADERS);
        configuration.setChatGLMConfig(chatGLMConfig);
        configuration.setChatGPTConfig(chatGPTConfig);
        configuration.setXunFeiConfig(xunFeiConfig);

        // 3. 会话工厂
        OpenAiSessionFactory factory = new DefaultOpenAiSessionFactory(configuration);
        // 4. 开启会话
        this.openAiSession = factory.openSession();
    }

    /**
     * 文本 & 流式对话；选择不同的模型测试 GPT_3_5_TURBO、GPT_3_5_TURBO_1106、GPT_3_5_TURBO_16K、GPT_4、CHATGLM_TURBO
     */
    @Test
    public void test_completions() throws Exception {
        // 1. 创建参数
        CompletionRequest request = CompletionRequest.builder()
                .stream(true)
                .messages(Collections.singletonList(Message.builder().role(CompletionRequest.Role.USER).content("1+1").build()))
//                .model(CompletionRequest.Model.GPT_3_5_TURBO_1106.getCode())
                .model(CompletionRequest.Model.CHATGLM_TURBO.getCode())
//                .model(CompletionRequest.Model.XUNFEI.getCode())
                .build();

        // 2. 请求等待
        CountDownLatch countDownLatch = new CountDownLatch(1);

        // 3. 应答请求
        EventSource eventSource = openAiSession.completions(request, new EventSourceListener() {
            @Override
            public void onEvent(EventSource eventSource, @Nullable String id, @Nullable String type, String data) {
                if ("[DONE]".equalsIgnoreCase(data)) {
                    log.info("OpenAI 应答完成");
                    return;
                }

                CompletionResponse chatCompletionResponse = JSON.parseObject(data, CompletionResponse.class);
                List<ChatChoice> choices = chatCompletionResponse.getChoices();
                for (ChatChoice chatChoice : choices) {
                    Message delta = chatChoice.getDelta();
                    if (CompletionRequest.Role.ASSISTANT.getCode().equals(delta.getRole())) continue;

                    // 应答完成
                    String finishReason = chatChoice.getFinishReason();
                    if (StringUtils.isNoneBlank(finishReason) && "stop".equalsIgnoreCase(finishReason)) {
                        return;
                    }

                    log.info("测试结果：{}", delta.getContent());
                }
            }

            @Override
            public void onClosed(EventSource eventSource) {
                log.info("对话完成");
                countDownLatch.countDown();
            }

            @Override
            public void onFailure(EventSource eventSource, @Nullable Throwable t, @Nullable Response response) {
                log.info("对话异常");
                countDownLatch.countDown();
            }
        });

        countDownLatch.await();
    }

    /**
     * 同步请求响应 GPT_3_5_TURBO、GPT_3_5_TURBO_1106、GPT_3_5_TURBO_16K、GPT_4、CHATGLM_TURBO
     * @throws Exception
     */
    @Test
    public void test_completions_future() throws Exception {
        // 1. 创建参数
        CompletionRequest request = CompletionRequest.builder()
                .stream(true)
                .messages(Collections.singletonList(Message.builder().role(CompletionRequest.Role.USER).content("1+1").build()))
                .model(CompletionRequest.Model.CHATGLM_TURBO.getCode())
                .build();

        // 2. 同步响应
        CompletableFuture<String> future = openAiSession.completions(request);
        String response = future.get(500, TimeUnit.SECONDS);
        log.info("测试结果：{}", response);
    }

}
```

## 三、对接计划

- [x] [OpenAI ChatGPT 系列模型](https://platform.openai.com/)
- [ ] [Anthropic Claude 系列模型](https://www.anthropic.com/)
- [ ] [Google PaLM2 系列模型](https://developers.generativeai.google/)
- [ ] [百度文心一言系列模型](https://cloud.baidu.com/doc/WENXINWORKSHOP/index.html)
- [ ] [阿里通义千问系列模型](https://help.aliyun.com/document_detail/2400395.html)
- [x] [讯飞星火认知大模型](https://www.xfyun.cn/doc/spark/Web.html)
- [x] [智谱 ChatGLM 系列模型](https://bigmodel.cn/)
- [ ] [360 智脑](https://ai.360.cn/)
- [ ] [腾讯混元大模型](https://cloud.tencent.com/document/product/1729)

## 四、开发规范

1. 打开地址 [openai-sdk-java](https://gitcode.net/KnowledgePlanet/open-hub/openai-sdk-java) Fork 代码到自己的仓库
2. 熟悉工程模型和代码，并调试运行理解整个框架的设计实现。之后开始承接需求并提交代码到自己的仓库。对于自己已经完成运行的调试的代码，可以提交 PR 代码。小傅哥在评审后，会合并你的提交。这样你就成为一个贡献者了，并记录在文档。
3. 修改 .env 为 .env.local 并配置相关的信息，就可以在 ApiTest 中调试了。

```java
# 主要type
feat:     增加新功能
fix:      修复bug

# 特殊type
docs:     只改动了文档相关的内容
style:    不影响代码含义的改动，例如去掉空格、改变缩进、增删分号
build:    构造工具的或者外部依赖的改动，例如webpack，npm
refactor: 代码重构时使用
revert:   执行git revert打印的message

# 暂不使用type
test:     添加测试或者修改现有测试
perf:     提高性能的改动
ci:       与CI（持续集成服务）有关的改动
chore:    不修改src或者test的其余修改，例如构建过程或辅助工具的变动
```

## 五、需求开发

1. 【简单】工程中有标记 `TODO` 标签待开发点，此类的功能比如在A模型中实现了，B、C 模型未实现，可以参考代码开发。
2. 【中等】阅读模型API文档，补全功能。这部分会从会话的调用，一直到执行器包下对应的实现，开发具体实现。
3. 【复杂】对未实现对接的模型，阅读API文档，添加对接。

---

以上开发内容，小傅哥会陆续的提交代码，你可以赶在我的前面实现，这样可以很好和我的开发进行对比，学习设计思想和落地实现。

## 六、其他资料

- https://gpgtools.org/ - brew install gpg
- https://issues.sonatype.org/browse/OSSRH-97151
- https://central.sonatype.org/publish/publish-maven/#javadoc-and-sources-attachments

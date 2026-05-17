# Chapter 3. 프롬프트 엔지니어링

## 3.1 프롬프트 템플릿

프롬프트란 AI 모델, 특히 대규모 언어 모델(LLM)에게 사용자가 원하는 작업을 구체적으로 지시하거나 질문의 형태로 요사항을 전달하는 일종의 명령문입니다. LLM은 입력된 프롬프트를 바탕으로 학습된 패턴과 지식을 활용하여 그에 상응하는 출력을 생성하게 됩니다. 프롬프트는 LLM에게 "어떤 상황에서", "어떤 방식으로", "어떤 형식으로" 응답해야 하는지 알려주는 중요한 역할을 합니다.

Spring AI는 프롬프트를 `Prompt` 클래스로 표현합니다. 이 클래스에는 메시지(`Message`)와 대화 옵션(`ChatOption`)을 담고 있으며, 각 메시지는 프롬프트 내에서 고유한 역할을 가집니다.

| 메시지 타입 | 역할 설명 |
|---|---|
| SystemMessage | LLM의 행동과 응답 스타일을 지시하는 메시지. 주로 LLM이 입력을 해석하는 방법과 답변하는 방식을 지시합니다. |
| UserMessage | 사용자의 질문, 명령을 담고 있는 메시지. LLM의 응답을 형성하는 기초가 됩니다. |
| AssistantMessage | LLM의 메시지. 단순한 답변을 넘어, 대화 기억 유지에도 사용되어 이전 대화 내용에 대해 도움을 줍니다. |

프롬프트는 정적 텍스트일 수도 있지만, 데이터 바인딩 처리 표시자를 가지고 있는 동적 텍스트일 수도 있습니다. 이러한 동적 텍스트의 프롬프트를 **템플릿(template)**이라고 합니다. 프롬프트 템플릿은 데이터를 바인딩하여 완성된 프롬프트를 생성합니다.

Spring AI는 프롬프트 템플릿을 위해 `PromptTemplate`을 제공합니다. `PromptTemplate`은 LLM에 전달할 완성된 프롬프트를 처리 표시자가 있는 템플릿 형태로 정의하고, 데이터 바인딩을 통해 동적으로 프롬프트를 완성합니다. 다음은 `{topic}`과 `{num}` 자리 표시자를 가지고 있는 `PromptTemplate`을 생성합니다.

```java
PromptTemplate promptTemplate = PromptTemplate.builder()
    .template("{topic}에 대해 동물 {num}개를 출력해 줘.")
    .build();
```

`PromptTemplate`는 자리 표시자에 데이터를 바인딩하기 위해 `create()` 메소드를 사용합니다. `create()` 메소드의 매개값은 각 자리 표시자에 바인딩할 값을 가진 `Map` 객체입니다. `topic`에 'AI'를 바인딩하고, `num`에 3을 바인딩할 경우 다음과 같이 호출합니다.

```java
Prompt prompt = promptTemplate.create(Map.of("topic", "AI", "num", 3));
```

`PromptTemplate`, `SystemPromptTemplate`, `AssistantPromptTemplate`는 `Prompt`만 반환하는 것이 아니라, 필요에 따라 완성된 텍스트와 메시지 객체를 반환할 수도 있습니다. 이때 `render()`와 `createMessage()`를 사용합니다.

```java
String userText = promptTemplate.render(Map.of("topic", "AI", "num", 3));
UserMessage userMessage = promptTemplate.createMessage(Map.of("topic", "AI", "num", 3));
```

### 예제 코드

VS Code에서 `book-spring-ai/projects/ch03-prompt` 프로젝트 폴더를 엽니다. `service/AiServicePromptTemplate.java` 파일을 열면 사용자로부터 한국어 문장과 번역할 언어를 받아 LLM에게 번역 요청하는 서비스 클래스를 볼 수 있습니다.

```java
@Service
@Slf4j
public class AiServicePromptTemplate {
    // #### 필드 ####
    private ChatClient chatClient;

    // ① SystemPromptTemplate 선언
    private PromptTemplate systemTemplate = SystemPromptTemplate.builder()
        .template("""
            HTML과 CSS를 사용해서 파란 글자로 출력하세요.
            <Span> 태그 안에 들어갈 내용만 출력하세요.
            """)
        .build();

    // ② PromptTemplate 선언
    private PromptTemplate userTemplate = PromptTemplate.builder()
        .template("다음 한국어 문장을 {language}로 번역해 주세요.\n 문장: {statement}")
        .build();

    // #### 생성자 ####
    public AiServicePromptTemplate(ChatClient.Builder chatClientBuilder) {
        chatClient = chatClientBuilder.build();
    }
}
```

① `SystemPromptTemplate`을 생성합니다. 템플릿 내용에는 자리 표시자가 없지만, 다른 메시지에서 동일한 시스템 메시지가 필요할 때 재사용할 수 있도록 필드로 선언합니다.

② `PromptTemplate`을 생성합니다. 템플릿 내용에는 `{language}`와 `{statement}` 자리 표시자가 있습니다. 마찬가지로 재사용할 수 있도록 필드로 선언합니다.

#### promptTemplate1() — PromptTemplate으로 Prompt 직접 생성

```java
public Flux<String> promptTemplate1(String statement, String language) {
    Prompt prompt = userTemplate.create(
        Map.of("statement", statement, "language", language));
    Flux<String> response = chatClient.prompt(prompt)
        .stream()
        .content();
    return response;
}
```

① `PromptTemplate`(`userTemplate`)으로 `Prompt`를 생성합니다. 바인딩 데이터로 `statement`와 `language` 매개값을 전달했습니다.

② `ChatClient`의 `prompt()` 메소드를 호출합니다. 실제로 내부적으로 전달되는 것은 `UserMessage`입니다.

`promptTemplate1()`은 `UserMessage`가 포함된 프롬프트를 LLM에 전달할 경우에 사용하면 좋습니다. 만약 `SystemMessage`와 함께 `UserMessage`가 포함된 프롬프트를 LLM에 전달하고 싶다면 `promptTemplate2()` 또는 `promptTemplate3()`과 같이 작성합니다.

#### promptTemplate2() — messages()로 복수 메시지 전달

```java
public Flux<String> promptTemplate2(String statement, String language) {
    Flux<String> response = chatClient.prompt()
        .messages(
            systemTemplate.createMessage(),
            userTemplate.createMessage(Map.of("statement", statement, "language", language)))
        .stream()
        .content();
    return response;
}
```

각각의 프롬프트 템플릿에서 `createMessage()`를 호출해서 `Message` 객체를 만들고 `.messages()`에 제공합니다.

#### promptTemplate3() — render()로 텍스트 추출 후 전달

```java
public Flux<String> promptTemplate3(String statement, String language) {
    Flux<String> response = chatClient.prompt()
        .user(userTemplate.render(Map.of("statement", statement, "language", language)))
        .system(systemTemplate.render())
        .stream()
        .content();
    return response;
}
```

각각의 프롬프트 템플릿에서 `render()`를 호출해서 메시지 텍스트를 얻고, `system()`과 `user()`에 각각 제공합니다.

#### promptTemplate4() — Java formatted()로 바인딩

```java
public Flux<String> promptTemplate4(String statement, String language) {
    String systemText = """
        다음 문자열을 이용해서 파란 글자로 출력하세요.
        <Span> 태그 안에 들어갈 내용만 출력하세요.
        """;

    String userText = """
        다음 한국어 문장을 %s로 번역해 주세요.
        문장: %s
        """.formatted(language, statement);

    Flux<String> response = chatClient.prompt()
        .system(systemText)
        .user(userText)
        .stream()
        .content();
    return response;
}
```

Java의 `String` 타입은 개행변수화된 문자열을 만들 수 있습니다. `formatted()` 메소드로 `%s`(텍스트), `%d`(정수), `%(실수)` 자리 표시자에 데이터를 바인딩한 문자열을 만듭니다. `String`의 `formatted()`를 사용하는 것은 표준 자바 라이브러리를 사용하는 것이므로, 메소드 내에서 바인딩해서 사용할 경우에 추천합니다.

#### 컨트롤러 코드

`controller/AiControllerPromptTemplate.java` 파일을 열면 `/ai/prompt-template` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerPromptTemplate {
    @Autowired
    private AiServicePromptTemplate aiService;

    @PostMapping(
        value = "/prompt-template",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_NDJSON_VALUE
    )
    public Flux<String> promptTemplate(
        @RequestParam("language") String language,
        @RequestParam("statement") String statement) {

        Flux<String> response = aiService.promptTemplate1(statement, language);
        //Flux<String> response = aiService.promptTemplate2(statement, language);
        //Flux<String> response = aiService.promptTemplate3(statement, language);
        //Flux<String> response = aiService.promptTemplate4(statement, language);
        return response;
    }
}
```

프로젝트를 실행하고 브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[prompt-template]` 버튼을 클릭합니다. 사용자가 보낸 한국어 문장을 타킷 언어로 번역하기 위해 `promptTemplate1()`~`promptTemplate4()`까지 주석을 하나씩 풀고 결과를 비교할 수 있습니다.

## 3.2 복수 메시지 추가

LLM에 요청할 때 하나의 `UserMessage`와 하나의 `SystemMessage`만 포함하는 것은 일반적인 방법입니다. 대부분의 경우는 그렇겠지만, 경우에 따라서는 한 개의 `SystemMessage`와 여러 개의 `UserMessage`, 여러 개의 `AssistantMessage`도 같이 포함할 수 있습니다.

대표적인 예로 대화 기억을 유지하기 위해, 이전 대화 내용(`UserMessage` + `AssistantMessage`) 전체를 프롬프트에 포함시킬 수 있습니다.

> **아이서 참고 — 대화 기억(Chat Memory)**
> Spring AI에서 대화 기억 기능은 09장에서 자세히 다룹니다. 대화 기억을 위한 기본 기능은 동일합니다.

### 예제 코드

`service/AiServiceMultiMessage.java` 파일을 엽니다. 이 서비스 클래스에는 이전 대화 내용 전체를 프롬프트에 포함할 수 있도록 `multi Message()` 메소드가 다음과 같이 선언되어 있습니다.

```java
public String multiMessage(String question, List<Message> chatMemory) {
    // ① 시스템 메시지 생성
    SystemMessage systemMessage = SystemMessage.builder()
        .text("""
            당신은 AI 비서입니다.
            제공되는 지난 대화 내용을 보고 우선적으로 답변해 주세요.
            """)
        .build();

    // ③ 처음 사용 시 시스템 메시지 저장
    if(chatMemory.size() == 0) {
        chatMemory.add(systemMessage);
    }

    log.info(chatMemory.toString());

    // ④ LLM에게 요청하고 응답받기
    ChatResponse chatResponse = chatClient.prompt()
        .messages(chatMemory)   // ⑤ 이전 대화 내용 추가
        .user(question)         // ⑥ 사용자 질문 추가
        .call()                 // ⑦
        .chatResponse();        // ⑧ ChatResponse로 반환

    // ⑨ 대화 메시지 저장
    UserMessage userMessage = UserMessage.builder().text(question).build();
    chatMemory.add(userMessage);

    // ⑩ 대화 메시지 저장
    AssistantMessage assistantMessage = chatResponse.getResult().getOutput();
    chatMemory.add(assistantMessage);

    String text = assistantMessage.getText();
    return text;
}
```

① `multiMessage()` 메소드의 매개값은 사용자의 질문과 이전 대화 내용이 저장된 `List<Message>`입니다. 두 개는 컨트롤러가 제공합니다.

② `SystemMessage`를 생성합니다. LLM에 지시하는 메시지를 만들었습니다.

③ 첫 대화일 경우에 `SystemMessage`를 `List<Message>`의 첫 메시지로 추가합니다.

④ `ChatClient`의 `prompt()`를 호출하고, 메소드 체이닝의 결과로 `ChatResponse`를 반환합니다.

⑤ `messages()` 메소드로 이전 대화 내용의 `List<Message>`를 추가합니다.

⑥ `user()` 메소드로 현재 사용자의 질문을 프롬프트에 추가합니다.

⑦ `call()` 메소드를 이어서 호출합니다. 안전한 응답이 오면 `ChatResponse`를 반환합니다.

⑧ `chatResponse()` 메소드로부터 `ChatResponse`를 얻습니다.

⑨ `ChatResponse`에서 `UserMessage`를 얻고 `List<Message>`에 추가합니다.

⑩ `AssistantMessage`에 저장되어 있는 텍스트 답변을 얻고 반환합니다.

`controller/AiControllerMultiMessages.java` 파일을 열면 `/ai/multi-messages` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerMultiMessages {
    @Autowired                                          // ①
    private AiServiceMultiMessages aiService;

    @PostMapping(
        value = "/multi-messages",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.TEXT_PLAIN_VALUE
    )
    public String multiMessages(
        @RequestParam("question") String question,
        HttpSession session) {                          // ②

        List<Message> chatMemory =                      // ③
            (List<Message>) session.getAttribute("chatMemory");
        if(chatMemory == null) {
            chatMemory = new ArrayList<>();
            session.setAttribute("chatMemory", chatMemory);
        }

        String answer = aiService.multiMessages(question, chatMemory); // ④
        return answer;
    }
}
```

① `AiServiceMultiMessages`를 주입합니다.

② 클라이언트가 보낸 사용자 질문과 함께 `HttpSession`을 주입합니다.

③ `HttpSession`에서 이전 대화 내용(`List<Message>`)을 찾되, 없으면 새로 생성해서 `HttpSession`에 저장합니다.

④ 사용자 질문과 대화 기억을 `AiServiceMultiMessages`의 `multiMessages()` 메서드에 전달해 완전한 텍스트 답변을 얻고 반환합니다.

프로젝트를 실행하고 브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[multi-messages]` 버튼을 클릭합니다. 질문 입력란에는 친구에게 말을 하듯 입력하고 `[제출]` 버튼을 클릭하면 이전 대화 내용을 기억하며 답변하는 것을 확인할 수 있습니다.

## 3.3 디폴트 메시지와 옵션

애플리케이션이 LLM에게 요청할 때 공통적으로 사용되는 메시지와 옵션이 있을 수 있습니다. 이러한 메시지와 옵션은 `ChatClient`를 생성할 때 기본으로 설정하면, LLM을 요청할 때 생략할 수 있습니다. `ChatClient.Builder`에는 다음과 같이 기본 설정을 위한 `default` 메소드가 있습니다.

| 메소드 | 설명 |
|---|---|
| `defaultSystem()` | 기본 SystemMessage를 추가 |
| `defaultUser()` | 기본 UserMessage를 추가 |
| `defaultOptions()` | 기본 대화 옵션을 설정 |

### 예제 코드

`service/AiServiceDefaultMethod.java` 파일을 엽니다. 이 서비스 클래스는 개별 LLM 요청에서 공통으로 사용하는 시스템 메시지와 대화 옵션(`ChatOptions`)을 `ChatClient`를 생성할 때 기본으로 설정합니다.

```java
@Service
@Slf4j
public class AiServiceDefaultMethod {
    // #### 필드 ####
    private ChatClient chatClient;

    // #### 생성자 ####
    public AiServiceDefaultMethod(ChatClient.Builder chatClientBuilder) {
        chatClient = chatClientBuilder
            .defaultSystem("적절한 검사서, 웃음 등을 넣어서 친절하게 주세요.")
            .defaultOptions(ChatOptions.builder()    // ①
                .temperature(1.0)
                .maxTokens(300)
                .build())
            .build();
    }

    // #### 메소드 ####
    public Flux<String> defaultMethod(String question) {
        Flux<String> response = chatClient.prompt()
            .user(question)
            .stream()
            .content();
        return response;
    }
}
```

① `ChatClient`를 생성합니다. 대화 옵션은 다양한 응답을 생성할 수 있도록 temperature를 최대 1.0으로 설정하고, 너무 긴 답변을 하지 않도록 최대 토큰 수를 300으로 설정했습니다.

`controller/AiControllerDefaultMethod.java` 파일을 열면 `/ai/default-method` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerDefaultMethod {
    @Autowired
    private AiServiceDefaultMethod aiService;   // ①

    @PostMapping(
        value = "/default-method",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_NDJSON_VALUE
    )
    public Flux<String> defaultMethod(
            @RequestParam("question") String question) {    // ②
        Flux<String> response = aiService.defaultMethod(question);
        return response;
    }
}
```

① `AiServiceDefaultMethod`를 주입받습니다.

② `defaultMethod()`는 사용자 질문을 받아 스트림 응답을 반환합니다.

## 3.4 프롬프트 엔지니어링

프롬프트 엔지니어링(Prompt Engineering)은 대규모 언어 모델을 효과적으로 활용하기 위해 입력 프롬프트를 설계하고 최적화하는 과학입니다. 이는 LLM이 주어진 입력을 정확히 이해하고, 목표에 부합하는 출력을 생성할 수 있도록 돕는 중요한 작업입니다.

다음은 프롬프트 엔지니어링 기본 가이드라인 표입니다.

| 기본 원칙 | 설명 |
|---|---|
| 명확하고 구체적인 요청 | 프롬프트는 모호하지 않고 구체적이어야 합니다. LLM이 원하는 답변의 방향을 명확히 정의해야 합니다. |
| 모델의 이해를 돕는 배경 정보 제공 | LLM이 답변을 더 정확히 이해할 수 있도록 메시지에 배경 정보나 문맥을 사용합니다. |
| 간결하고 직관적인 문장 사용 | 모호하고 수식어가 없는 복잡한 문장보다는 간결하고 직관적인 문장을 사용합니다. |
| 적절한 예시 사용 | LLM이 사용자가 원하는 스타일이나 출력 형식을 정확히 이해할 수 있도록 예시를 포함하는 것이 좋습니다. |
| 단답계 질문 피하기 | 어떤 질문을 한 프롬프트에 단지 말고, 하나의 질문에 집중하여 LLM이 정확한 답변을 할 수 있도록 합니다. |
| LLM의 한계 이해 | LLM이 처리할 수 있는 범위의 한계를 이해하고, 그에 맞는 프롬프트를 설계해야 합니다. |
| LLM의 역할 부여하기 | 모델에게 정확한 역할을 부여하여 특정 작업에 적합하게 동작하도록 합니다. (예: "너는 역사 전문가입니다. 19세기 역사에 대해 설명해 주세요.") |

## 3.5 제로-샷 프롬프트

제로-샷(Zero-Shot) 프롬프트는 AI에게 예시 없이 작업을 수행하도록 요청하는 방법입니다. 이 방식은 모델이 처음부터 지시된 텍스트를 이해하고 실행할 수 있는 능력이 있을 경우에 사용 가능합니다.

LLM은 방대한 텍스트 데이터를 학습하여 "번역", "요약", "분류"와 같은 작업이 무엇인지 알고 있습니다. 그렇기 때문에 명시적인 예시 없이도 이러한 작업을 잘 처리할 수 있습니다.

아래 내용은 프롬프트 엔지니어링에서 많이 사용하는 기법들입니다.

- 제로-샷 프롬프트
- 퓨-샷 프롬프트
- 역할 부여 프롬프트
- 스텝-백 프롬프트
- 생각의 사슬 프롬프트
- 자기 일관성

### 예제 코드

`service/AiServiceZeroShotPrompt.java` 파일을 엽니다. 이 서비스 클래스는 제로-샷 프롬프트를 이용해서 리뷰 감정을 POSITIVE(긍정적), NEUTRAL(중립적), NEGATIVE(부정적)으로 분류합니다.

```java
@Service
@Slf4j
public class AiServiceZeroShotPrompt {
    // #### 필드 ####
    private ChatClient chatClient;

    private PromptTemplate promptTemplate = PromptTemplate.builder()  // ①
        .template("""
            영화 리뷰를 [긍정적, 부정적, 중립적] 중에서 하나로 분류하세요.
            레이블만 반환하세요.
            리뷰: {review}
            """)
        .build();

    // #### 생성자 ####
    public AiServiceZeroShotPrompt(ChatClient.Builder chatClientBuilder) {
        chatClient = chatClientBuilder
            .defaultOptions(ChatOptions.builder()      // ②
                .temperature(0.0)
                .maxTokens(4)
                .build())
            .build();
    }

    // #### 메소드 ####
    public String zeroShotPrompt(String review) {
        String sentiment = chatClient.prompt()         // ③
            .user(promptTemplate.render(Map.of("review", review)))
            .call()
            .content();
        return sentiment;
    }
}
```

① 재사용 가능한 `PromptTemplate`을 생성합니다. `{review}` 자리 표시자에 사용자의 리뷰를 바인딩합니다. LLM은 대규모 텍스트로 훈련되어 있기 때문에 프롬프트에 예시 없이도 분류 작업을 처리할 수 있습니다.

② `ChatClient`를 생성할 때 기본 대화 옵션으로 temperature를 가장 낮은 0.0으로 설정했습니다. 세 가지 레이블 중 하나만 답변하면 되므로 다양성이 필요 없습니다. 최대 토큰 수도 레이블 길이에 맞게 4개로 제한했습니다.

③ `ChatClient`의 `prompt()`를 시작합니다. `PromptTemplate`에 바인딩할 리뷰를 제공하고, `call()`로 완전한 응답(레이블)을 받아 `content()`로 출력합니다.

`controller/AiControllerZeroShotPrompt.java` 파일을 열면 `/ai/zero-shot-prompt` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerZeroShotPrompt {
    @Autowired
    private AiServiceZeroShotPrompt aiService;   // ①

    @PostMapping(
        value = "/zero-shot-prompt",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.TEXT_PLAIN_VALUE
    )
    public String zeroShotPrompt(
            @RequestParam("review") String review) {  // ②
        String reviewSentiment = aiService.zeroShotPrompt(review);
        return reviewSentiment;
    }
}
```

① ② `AiServiceZeroShotPrompt`를 필드 주입받고, `zeroShotPrompt()` 메서드를 호출해서 리뷰 감정 레이블을 반환합니다.

프로젝트를 실행하고 브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[zero-shot-prompt]` 버튼을 클릭합니다. 리뷰 입력란에 내용을 다추고 [제출]을 클릭하면 긍정적·부정적·중립적 레이블이 반환됩니다.

## 3.6 퓨-샷 프롬프트

퓨-샷(Few-Shot) 프롬프트는 LLM에게 몇 개의 예시를 제공하여 사용자가 원하는 방식으로 출력하도록 유도하는 기법입니다. 한 개의 예시를 제공하는 것을 원-샷(One-Shot)이라고도 합니다.

LLM은 기본적으로 많은 데이터로 학습된 상태지만, 어떤 방식으로 답변해야 하는지에 대한 명확한 기준이 없습니다. 퓨-샷 프롬프트를 사용하면 LLM이 사용자가 원하는 형식을 학습하고, 이를 기반으로 새로운 질문에도 동일한 형식으로 답변할 수 있습니다.

퓨-샷 프롬프트는 원하는 출력이 없는 경우, 예시를 몇 개 제시함으로써 결과물의 품질을 크게 향상시킬 수 있습니다.

### 예제 코드

`service/AiServiceFewShotPrompt.java` 파일을 엽니다. 이 서비스 클래스는 고객이 피자를 주문할 때 사이즈, 크기, 재료를 받아서, LLM이 정리해서 구조화된 JSON으로 반환합니다.

```java
public String fewShotPrompt(String order) {
    // ① 프롬프트 정의 (예시 포함)
    String strPrompt = """
        예시1:
        작은 피자 하나, 치즈향 토마토 소스, 페퍼로니 올려서 주세요.
        고객 주문을 유효한 JSON 형식으로 바꿔주세요.
        추가 설명을 포함하지 마세요.

        JSON 응답:
        {
            "size": "small",
            "type": "normal",
            "ingredients": ["cheese", "tomato sauce", "pepperoni"]
        }

        예시2:
        큰 피자 하나, 토마토 소스와 바질, 모짜렐라 올려서 주세요.
        JSON 응답:
        {
            "size": "large",
            "type": "normal",
            "ingredients": ["tomato sauce", "basil", "mozzarella"]
        }

        고객 주문: %s""".formatted(order);

    Prompt prompt = Prompt.builder()              // ②
        .content(strPrompt)
        .build();

    // ③ LLM으로 요청하고 응답 받음
    String pizzaOrderJson = chatClient.prompt(prompt)
        .options(ChatOptions.builder()
            .temperature(0.0)
            .maxTokens(300)
            .build())
        .call()
        .content();

    return pizzaOrderJson;
}
```

① `PromptTemplate`을 사용하지 않은 이유는 예시에서 `{ }`가 자리 표시자가 아니라 JSON을 표현하고 있기 때문입니다. 대신 Java `formatted()`로 `%s`에 주문 내용을 바인딩합니다.

② 문자열을 가지고 직접 `Prompt`를 생성합니다.

③ `ChatClient.prompt()`를 매개값으로 주었습니다. 대화 옵션은 다양성 응답이 필요 없으므로 temperature를 가장 낮은 0.0으로 설정하고, 최대 토큰 수는 300으로 설정했습니다. 동기 요청인 `call()` 메소드를 호출하고, 오면 문자열 JSON을 반환하도록 `content()`를 호출했습니다.

`controller/AiControllerFewShotPrompt.java` 파일을 열면 `/ai/few-shot-prompt` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerFewShotPrompt {
    @Autowired
    private AiServiceFewShotPrompt aiService;   // ①

    @PostMapping(
        value = "/few-shot-prompt",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public String fewShotPrompt(
            @RequestParam("order") String order) {   // ②
        String json = aiService.fewShotPrompt(order);
        return json;
    }
}
```

① ② `AiServiceFewShotPrompt`를 필드 주입받고, `fewShotPrompt()` 메서드를 호출해서 서술식 주문을 JSON으로 반환합니다.

프로젝트를 실행하고 브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[few-shot-prompt]` 버튼을 클릭합니다. 주문을 입력하면 피자 주문이 JSON 형식으로 반환됩니다.

## 3.7 역할 부여 프롬프트

LLM에게 특정 역할이나 인물로 반응하도록 지시하면 출력 결과가 해당 분야 전문성, 특정 정체성, 전문성 또는 관점을 반영하게 됩니다. 이를 통해 출력 내용의 스타일, 톤(진지한, 유머스러운), 깊이를 조정할 수 있습니다.

역할을 부여함으로써 LLM은 해당 분야의 대화 스타일로 출력합니다. 이러한 역할에는 전문가("당신은 경험이 풍부한 데이터 과학자입니다"), 안내자("여행 가이드 역할을 하세요"), 또는 스타일리시한 인물("제이스미어처럼 답변해 주세요")이 포함됩니다.

### 예제 코드

`service/AiServiceRoleAssignmentPrompt.java` 파일을 엽니다. 이 서비스 클래스는 LLM이 여행 가이드 역할을 하도록 합니다. 사용자가 원하는 위치에서 방문할 수 있는 3곳을 제안하고, 사용자가 방문하고 싶은 장소의 유형에 따라 안내하도록 지시합니다.

```java
public Flux<String> roleAssignment(String requirements) {
    Flux<String> travelSuggestions = chatClient.prompt()
        // ① 시스템 메시지 추가
        .system("""
            당신이 여행 가이드 역할을 해 주겠으면 좋겠습니다.
            이제 요청사항을 알려주세요. 사용자에게 위치를 알려주시고,
            근처에 있는 3곳을 제안해 주고, 이들을 방문하고 싶은
            곳도를 할 수 있습니다.
            """)
        // ② 사용자 메시지 추가
        .user("요청사항: %s".formatted(requirements))
        // ③ 대화 옵션 설정
        .options(ChatOptions.builder()
            .temperature(1.0)
            .maxTokens(1000)
            .build())
        // ④ LLM으로 요청하고 응답 받기
        .stream()
        .content();
    return travelSuggestions;
}
```

① 시스템 메시지로 여행 가이드 역할을 부여합니다.

② 사용자 메시지로 요청 사항을 전달합니다.

③ 대화 옵션을 설정합니다. temperature를 1.0으로 높여 다양한 답변을 유도하고, 최대 토큰 수는 1000으로 설정합니다.

④ 스트림으로 LLM에 요청하고 응답을 받습니다.

`controller/AiControllerRoleAssignmentPrompt.java` 파일을 열면 `/ai/role-assignment` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerRoleAssignmentPrompt {
    @Autowired
    private AiServiceRoleAssignmentPrompt aiService;   // ①

    @PostMapping(
        value = "/role-assignment",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_NDJSON_VALUE
    )
    public Flux<String> roleAssignment(
            @RequestParam("requirements") String requirements) {   // ②
        Flux<String> travelSuggestions = aiService.roleAssignment(requirements);
        return travelSuggestions;
    }
}
```

① ② `AiServiceRoleAssignmentPrompt`를 필드 주입받고, `roleAssignment()` 메서드를 호출해서 스트림 응답을 반환합니다.

브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[role-assignment]` 버튼을 클릭하고 요청 사항을 입력해서 테스트합니다.

## 3.8 스텝-백 프롬프트

스텝-백(Step-Back) 프롬프트는 복잡한 질문을 여러 단계로 분해하는 기법입니다. 이 기법은 LLM이 즉각적인 답변을 생성하기 전에 "한 걸음 물러나" 문제와 관련된 폭넓은 배경 지식을 탐색하도록 유도합니다. 단계별 답변에 대한 다음 단계의 배경 지식으로 이어지기 때문에, LLM은 단계적으로 배경 지식을 쌓아가며 더 정확한 답변을 제공할 수 있습니다.

사용자가 다음과 같은 질문을 했다고 가정해 보겠습니다.

> 서울에서 올롬도로 갈 때 비용이 가장 적게 드는 방법은?

이 질문은 다음과 같이 여러 단계의 질문으로 분해할 수 있습니다.

- **단계1**: 서울에서 올롬도로 가는 교통 수단은 무엇인가요?
- **단계2**: 각 교통 수단의 비용은 얼마인가요?
- **단계3 (최종)**: 비용이 가장 적은 교통 수단은 무엇인가요?

각 단계의 답변은 다음 단계 사용자 텍스트의 문맥으로 포함되어 LLM으로 전달됩니다.

### 예제 코드

`service/AiServiceStepBackPrompt.java` 파일을 엽니다. 이 서비스 클래스는 사용자의 질문을 여러 단계의 질문으로 분해한 후, 단계별 답변을 다음 단계의 문맥으로 추가합니다.

#### stepBackPrompt() — 질문 분해 및 단계별 처리

```java
public String stepBackPrompt(String question) throws Exception {
    // ① 질문을 단계별로 분해 요청
    String questions = chatClient.prompt()
        .user("""
            사용자 질문을 정리하기 Step-Back 프롬프트 기법을 사용하려고 합니다.
            사용자 질문을 단계별 질문으로 재구성해 주세요.
            맨 마지막 질문은 사용자 질문과 일치해야 합니다.
            단계별 질문은 JSON 배열로 출력해 주세요.
            예시: ["...", "...", ...]
            사용자 질문: %s
            """.formatted(question))
        .call()
        .content();

    // ② JSON 배열 추출
    String json = questions.substring(
        questions.indexOf("["), questions.indexOf("]") + 1);
    log.info(json);

    // ③ JSON → List<String> 파싱
    ObjectMapper objectMapper = new ObjectMapper();
    List<String> listQuestion = objectMapper.readValue(
        json, new TypeReference<List<String>>() {});

    String[] answerArray = new String[listQuestion.size()];
    for (int i = 0; i < listQuestion.size(); i++) {
        String stepQuestion = listQuestion.get(i);
        // ④ 단계별 답변 수집
        String stepAnswer = getStepAnswer(stepQuestion, answerArray);
        answerArray[i] = stepAnswer;
        log.info("단계: {}, 질문: {}, 답변: {}", i + 1, stepQuestion, stepAnswer);
    }

    return answerArray[answerArray.length - 1]; // ⑤ 최종 답변 반환
}
```

① LLM에게 단계별로 분해해 달라고 요청합니다. 마지막 질문은 원래 질문과 동일해야 합니다.

② 응답 문자열에서 `[...]` 부분을 추출합니다. JSON 배열이 아닌 텍스트가 앞뒤에 붙을 수 있기 때문입니다.

③ JSON을 `List<String>`으로 파싱합니다.

④ `getStepAnswer()`로 단계별 답변을 얻고 배열에 저장합니다.

⑤ 마지막 답변(최종 단계)을 반환합니다.

#### getStepAnswer() — 이전 단계 문맥을 포함한 단계별 답변

```java
public String getStepAnswer(String question, String... prevStepAnswers) {
    // ① 이전 단계 답변을 문맥으로 누적
    String context = "";
    for (String prevStepAnswer : prevStepAnswers) {
        context += Objects.requireNonNullElse(prevStepAnswer, "");
    }

    // ② 이전 답변을 문맥으로 포함해서 LLM에 요청
    String answer = chatClient.prompt()
        .user("""
            문맥: %s
            질문: %s
            """.formatted(context, question))
        .call()
        .content();
    return answer;
}
```

① 이전 단계 답변들을 `context`에 누적합니다. `null`인 경우에는 빈 문자열로 처리합니다.

② `ChatClient`로 LLM에 요청할 때, 이전 단계 답변을 문맥으로 프롬프트에 포함합니다.

`controller/AiControllerStepBackPrompt.java` 파일을 열면 `/ai/step-back-prompt` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerStepBackPrompt {
    @Autowired
    private AiServiceStepBackPrompt aiService;   // ①

    @PostMapping(
        value = "/step-back-prompt",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.TEXT_PLAIN_VALUE
    )
    public String stepBackPrompt(
            @RequestParam("question") String question)
            throws Exception {   // ②
        String answer = aiService.stepBackPrompt(question);
        return answer;
    }
}
```

① ② `AiServiceStepBackPrompt`를 주입받고, `stepBackPrompt()` 메서드를 호출해서 최종 단계 답변을 반환합니다.

브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[step-back-prompt]` 버튼을 클릭합니다. 단계별 질문에 대한 답변을 얻기 때문에 시간이 조금 걸릴 수 있습니다.

## 3.9 생각의 사슬 프롬프트

CoT(Chain of Thought, 생각의 사슬) 프롬프트는 LLM에게 문제를 해결하는 과정을 명시적으로 요청하거나 논리적인 단계로 생각하도록 요구함으로써, 다단계 추론이 필요한 작업에서 성능을 향상시킬 수 있습니다.

CoT는 모델이 최종 답을 도출하기 전에 중간 추론 단계를 생성하도록 유도합니다. 이는 인간이 복잡한 문제를 해결하는 방식과 유사하며, 모델의 사고 과정을 명확하게 만들고 더 정확한 결론에 도달할 수 있도록 돕습니다.

프롬프트에 **"한 걸음씩 생각해 봅시다.(Let's think step by step.)"** 라는 핵심 문구를 넣어 모델이 자신의 사고 과정을 보여주도록 유도합니다. 추가적으로 Few-Shot 예시를 제시해 주면 사고 과정이 명확해지고 정답을 도출할 확률이 높아집니다. CoT는 특히 복잡한 수학 문제에 유용합니다.

### 예제 코드

`service/AiServiceChainOfThoughtPrompt.java` 파일을 엽니다. 이 서비스 클래스는 수학 문제를 풀기 위해 CoT 프롬프트를 사용합니다. "한 걸음씩 생각해 봅시다" 문구로 사고 과정을 명확히 유도합니다.

```java
@Service
@Slf4j
public class AiServiceChainOfThoughtPrompt {
    private ChatClient chatClient;

    public AiServiceChainOfThoughtPrompt(ChatClient.Builder chatClientBuilder) {
        chatClient = chatClientBuilder.build();
    }

    public Flux<String> chainOfThought(String question) {
        Flux<String> answer = chatClient.prompt()
            .user("""
                한 걸음씩 생각해 봅시다.   // ①
                %s
                """.formatted(question))
            .stream()
            .content();
        return answer;
    }
}
```

① 사용자 텍스트에 질문과 함께 "한 걸음씩 생각해 봅시다" 문구를 포함합니다. 예시는 필요 없지만 논리적 단계를 보여줘야 할 때 이 문구가 핵심 역할을 합니다.

`controller/AiControllerChainOfThoughtPrompt.java` 파일을 열면 `/ai/chain-of-thought` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerChainOfThoughtPrompt {
    @Autowired
    private AiServiceChainOfThoughtPrompt aiService;   // ①

    @PostMapping(
        value = "/chain-of-thought",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_NDJSON_VALUE
    )
    public Flux<String> chainOfThought(
            @RequestParam("question") String question) {   // ②
        Flux<String> answer = aiService.chainOfThought(question);
        return answer;
    }
}
```

① ② `AiServiceChainOfThoughtPrompt`를 필드 주입받고, `chainOfThought()` 메서드를 호출해서 비동기 스트림 응답을 반환합니다.

브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[chain-of-thought]` 버튼을 클릭하고 수학 문제를 입력해서 테스트합니다.

## 3.10 자기 일관성

자기 일관성(Self-Consistency)은 LLM에게 여러 번 요청해서 얻은 응답을 집계하여 다수결로 최종 응답을 정하는 기법입니다. 즉, LLM이 일관성 있게 응답하는 것을 체크하는 것입니다. 이 기법은 LLM 출력의 분산성을 해결합니다.

지식 내용이 중요한지 아닌지, LLM이 판단하도록 해서 다수결로 최종 응답을 결정합니다.

### 예제 코드

`service/AiServiceSelfConsistency.java` 파일을 엽니다. 이 서비스 클래스는 메일 또는 메시지 내용을 보고 5번에 걸쳐 판단하고 다수결로 최종 응답을 결정합니다.

```java
@Service
@Slf4j
public class AiServiceSelfConsistency {
    private ChatClient chatClient;

    // ① PromptTemplate 필드 선언
    private PromptTemplate promptTemplate = PromptTemplate.builder()
        .template("""
            다음 내용을 IMPORTANT, NOT_IMPORTANT 둘 중 하나로 분류해 주세요.
            내용: {content}
            """)
        .build();

    public AiServiceSelfConsistency(ChatClient.Builder chatClientBuilder) {
        chatClient = chatClientBuilder.build();
    }

    public String selfConsistency(String content) {
        int importantCount = 0;
        int notImportantCount = 0;

        String userText = promptTemplate.render(Map.of("content", content));

        // ① 5번에 걸쳐 응답 받기
        for (int i = 0; i < 5; i++) {
            // ② LLM 요청 및 응답 받기
            String output = chatClient.prompt()
                .user(userText)
                .options(ChatOptions.builder()
                    .temperature(1.0)    // 다양한 응답 유도
                    .build())
                .call()
                .content();

            log.info("{}: {}", i, output.toString());

            // ③ 결과 집계
            if (output.equals("IMPORTANT")) {
                importantCount++;
            } else {
                notImportantCount++;
            }
        }

        // ④ 다수결로 최종 분류 결정
        String finalClassification = importantCount > notImportantCount ?
            "중요함" : "중요하지 않음";
        return finalClassification;
    }
}
```

① 재사용 가능한 `PromptTemplate`을 필드로 선언합니다. `{content}` 자리 표시자에 바인딩되는 내용이 중요한지 여부를 두 레이블로 분류하도록 템플릿을 작성했습니다.

② 5번에 걸쳐 LLM에 요청하고 응답을 받습니다. 다양한 판단을 하기 위해 `temperature`를 최대 1.0으로 설정했습니다.

③ ④ 응답 결과에 따라 집계하고, 다수결 결과로 최종 응답(분류)을 결정합니다.

`controller/AiControllerSelfConsistency.java` 파일을 열면 `/ai/self-consistency` 요청 매핑 메소드를 볼 수 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerSelfConsistency {
    @Autowired
    private AiServiceSelfConsistency aiService;   // ①

    @PostMapping(
        value = "/self-consistency",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.TEXT_PLAIN_VALUE
    )
    public String selfConsistency(
            @RequestParam("content") String content) {   // ②
        String answer = aiService.selfConsistency(content);
        return answer;
    }
}
```

① ② `AiServiceSelfConsistency`를 필드 주입받고, `selfConsistency()` 메서드를 호출해서 다수결 판단 결과를 반환합니다.

브라우저에서 `http://localhost:8080`으로 요청한 뒤, `[self-consistency]` 버튼을 클릭합니다. 메시지를 입력하면 LLM이 5번 판단한 결과를 집계하여 "중요함" 또는 "중요하지 않음"으로 출력합니다. 일상적인 인사말과 같이 정보가 없는 내용은 "중요하지 않음"으로 판단합니다.
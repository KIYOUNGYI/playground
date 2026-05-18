# Chapter 4. 구조화된 출력 변환

## 4.1 구조화된 출력 변환기

**구조화된 출력(Structured Output)**이란 데이터의 의미와 관계를 고려해서 JSON과 같은 형식으로 출력하는 것을 말합니다. 데이터를 전달하거나 애플리케이션에서는 매우 중요합니다.

일반적으로 LLM의 출력은 텍스트 문장이나, LLM이 구조화된 출력을 하려면 프롬프트에 출력 형식 지침을 포함해야 합니다. 또한 출력을 구조화하기 위해 애플리케이션에서 LLM의 출력을 적절히 처리해야 합니다.

Spring AI는 이러한 작업을 할 수 있도록 구조화된 출력 변환기를 제공합니다.

### StructuredOutputConverter 인터페이스

```java
public interface StructuredOutputConverter<T>
    extends FormatProvider, Converter<String, T> {
}
```

| 인터페이스 | 메서드 | 설명 |
|---|---|---|
| `FormatProvider` | `getFormat(): String` | 출력 형식 지침 문자열 반환. PromptTemplate에 추가되어 LLM에게 형식 지침을 전달 |
| `Converter<String, T>` | `convert(String output): T` | LLM 출력 텍스트를 T 객체로 변환 |

### 작동 흐름

```
Raw Input
    ↓
PromptTemplate + FormatProvider(형식 지침)
    ↓
ChatModel.call(Prompt): String
    ↓
Raw Output (text)
    ↓
convert(String output): T
    ↓
Structured Output (T)
```

### 구조화된 출력 변환기 목록

| 구조화된 출력 변환기 | 설명 |
|---|---|
| `ListOutputConverter` | FormatProvider: 쉼표로 구분된 목록 출력을 위한 형식 지침 제공 / Converter: LLM 출력을 `List<String>`으로 변환 |
| `BeanOutputConverter<T>` | Converter: LLM 출력을 T 객체로 변환 |
| `MapOutputConverter` | FormatProvider: JSON 형식 출력을 위한 형식 지침 제공 / Converter: LLM 출력을 `Map<String, Object>`으로 변환 |

애플리케이션에서 이들 구현체를 사용하는 방법은 다음 두 가지가 있습니다.

| 구분 | 저수준 | 고수준 |
|---|---|---|
| 설명 | 변환기를 직접 생성하여 형식 지침을 생성하고, 변환하는 방법 | ChatClient의 메서드 체이닝에 `entity()` 메서드로 출력하는 방법 |

---

## 4.2 List(String)으로 변환 (ListOutputConverter)

LLM의 출력을 `List<String>`으로 변환하고 싶다면, `ListOutputConverter`를 사용할 수 있습니다. 이 변환기는 LLM이 쉼표로 구분된 텍스트를 출력하고, LLM 출력을 `List<String>`으로 변환합니다.

이번에는 고수준 방식의 `listOutputConverterHighLevel()` 메소드를 보면서 설명하겠습니다.

프로젝트는 `book-spring-ai/projects/ch04-structured-output` 폴더를 엽니다.

- **01** VS Code로 `book-spring-ai/projects/ch04-structured-output/service/AiServiceListOutputConverter.java` 파일을 엽니다.
- **02** `service/AiServiceListOutputConverter.java` 파일을 엽니다. 이 서비스 클래스는 주어진 도시의 유명한 호텔 5개를 LLM에게 물어보고, LLM 출력을 `List<String>`으로 변환합니다.
- **03** LLM이 쉼표로 구분된 호텔 이름들을 출력하면 `List<String>`으로 변환합니다.

### 저수준(Low Level) 방식

```java
public List<String> listOutputConverterLowLevel(String city) {
    // 구조화된 출력 변환기 생성
    ListOutputConverter converter = new ListOutputConverter();

    // 프롬프트 템플릿 생성
    PromptTemplate promptTemplate = PromptTemplate.builder()
        .template("{city}에서 유명한 호텔 5개를 출력해주세요. {format}")
        .build();

    // {format} 자리에 converter.getFormat() 형식 지침을 삽입
    Prompt prompt = promptTemplate.create(
        Map.of("city", city, "format", converter.getFormat()));

    // LLM의 출력을 텍스트로 받음
    String commaSeparatedString = chatClient.prompt(prompt).call().content();

    // ListOutputConverter로 List<String>으로 변환
    List<String> hotelList = converter.convert(commaSeparatedString);
    return hotelList;
}
```

- **1** LLM의 출력을 `List<String>`으로 변환하는 `ListOutputConverter`를 생성합니다.
- **2** 프롬프트 템플릿에서 프롬프트를 생성할 때, 도시 이름과 출력 형식 지침을 자리 표시자에 바인딩해서 사용합니다. LLM에게 형식 지침을 제공해야 합니다.
- **3** LLM이 쉼표로 구분된 호텔 이름들을 출력하면 `List<String>`으로 변환합니다.

### 고수준(High Level) 방식

- **04** 이번에는 고수준 방식의 `listOutputConverterHighLevel()` 메소드를 보겠습니다.

```java
public List<String> listOutputConverterHighLevel(String city) {
    List<String> hotelList = chatClient.prompt()
        .user("%s에서 유명한 호텔 5개를 출력하세요.".formatted(city))
        .call()
        .entity(new ListOutputConverter());
    return hotelList;
}
```

- **1** `entity()` 메소드를 사용할 때, `ListOutputConverter` 객체를 사용해서 LLM의 출력을 `List<String>`으로 변환합니다. 저수준 방식과 마찬가지로, 형식 지침을 메시지에 자동으로 삽입하며, 쉼표로 구분된 호텔 이름들을 출력하도록 합니다.

### 컨트롤러

- **05** controller/AiControllerListOutputConverter.java 파일을 엽니다. 이 컨트롤러 클래스에는 `/ai/list-output-converter` 요청 매핑 메서드가 다음과 같이 선언되어 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerListOutputConverter {
    // ##### 필드 #####
    @Autowired
    private AiServiceListOutputConverter aiService;

    // ##### 메소드 #####
    @PostMapping(
        value = "/list-output-converter",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public List<String> listOutputConverter(@RequestParam("city") String city) {
        List<String> hotelList = aiService.listOutputConverterLowLevel(city);
        // List<String> hotelList = aiService.listOutputConverterHighLevel(city);
        return hotelList;
    }
}
```

- **1** 저수준과 고수준 방식의 메소드를 주석처리하여 번갈아 사용합니다.

- **06** 프로젝트를 실행합니다. 브라우저로 `http://localhost:8080`으로 요청하고, 도시 이름을 입력한 후 [list-output-converter] 버튼을 클릭하면서 저수준과 고수준 방식의 메소드를 출력하고 있습니다. 2에서 주석을 번갈아가면서 실행해 보세요.

---

## 4.3 T로 변환 (BeanOutputConverter)

LLM의 출력을 T 객체로 변환하고 싶다면, `BeanOutputConverter<T>`를 사용할 수 있습니다. T는 변환할 자바 타입입니다. 이 변환기는 LLM이 JSON 출력을 생성하고, LLM의 출력을 T 객체로 변환합니다.

프로젝트 소스 코드를 보면서 설명하겠습니다.

- **01** `service/AiServiceBeanOutputConverter.java` 파일을 엽니다. 이 서비스 클래스는 주어진 도시를 LLM에게 물어보고, LLM 출력을 `Hotel` 객체로 변환합니다.
- **02** 먼저 저수준 방식의 `beanOutputConverterLowLevel()` 메소드를 보겠습니다.

### Hotel DTO 클래스

```java
@Data
public class Hotel {
    // 도시 이름
    private String city;
    // 호텔 목록
    private List<String> names;
}
```

### 저수준(Low Level) 방식

```java
public Hotel beanOutputConverterLowLevel(String city) {
    // 구조화된 출력 변환기 생성
    BeanOutputConverter<Hotel> beanOutputConverter =
        new BeanOutputConverter<>(Hotel.class);

    // 프롬프트 템플릿 생성
    PromptTemplate promptTemplate = PromptTemplate.builder()
        .template("{city}에서 유명한 호텔 5개를 출력해주세요.\n{format}")
        .build();

    // 프롬프트 생성
    Prompt prompt = promptTemplate.create(Map.of(
        "city", city,
        "format", beanOutputConverter.getFormat()));

    // LLM의 JSON 출력 얻기
    String json = chatClient.prompt(prompt).call().content();

    // JSON을 Hotel로 변환
    Hotel hotel = beanOutputConverter.convert(json);
    return hotel;
}
```

### 고수준(High Level) 방식

```java
public Hotel beanOutputConverterHighLevel(String city) {
    Hotel hotel = chatClient.prompt()
        .user("%s에서 유명한 호텔 5개를 출력하세요.".formatted(city))
        .call()
        .entity(Hotel.class);
    return hotel;
}
```

- **1** `entity(Hotel.class)` 메서드: `BeanOutputConverter`를 사용하지 않고 `Hotel.class`를 직접 전달합니다. 저수준 방식과 마찬가지로, 형식 지침을 메시지에 자동으로 삽입하며, LLM의 JSON 출력을 `Hotel`로 변환합니다.

### 컨트롤러

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerBeanOutputConverter {
    // ##### 필드 #####
    @Autowired
    private AiServiceBeanOutputConverter aiService;

    @PostMapping(
        value = "/bean-output-converter",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public Hotel beanOutputConverter(@RequestParam("city") String city) {
        Hotel hotel = aiService.beanOutputConverterLowLevel(city);
        // Hotel hotel = aiService.beanOutputConverterHighLevel(city);
        return hotel;
    }
}
```

- 프로젝트를 실행합니다. 브라우저로 `http://localhost:8080`에서 [bean-output-converter] 버튼을 클릭하여 테스트합니다.

---

## 4.4 List(T)로 변환 (BeanOutputConverter)

LLM의 출력을 `List<T>` 객체로 변환하고 싶다면, `BeanOutputConverter<List<T>>`를 사용할 수 있습니다. 이 변환할 자바 타입은 `List<T>` 타입입니다. `ParameterizedTypeReference<List<Hotel>>`를 사용해서 LLM의 JSON 출력을 `List<Hotel>` 객체로 변환합니다.

프로젝트 소스 코드를 보면서 설명하겠습니다.

- **01** `service/AiServiceParameterizedTypeReference.java` 파일을 엽니다. 이 서비스 클래스는 주어진 도시들의 유명한 호텔 3개를 LLM에게 물어보고, `List<Hotel>`로 변환합니다.
- **02** 먼저 저수준 방식의 `genericBeanOutputConverterLowLevel()` 메소드를 보겠습니다.

### 저수준(Low Level) 방식

```java
public List<Hotel> genericBeanOutputConverterLowLevel(String cities) {
    // 구조화된 출력 변환기 생성
    BeanOutputConverter<List<Hotel>> beanOutputConverter =
        new BeanOutputConverter<>(new ParameterizedTypeReference<List<Hotel>>() {});

    // 프롬프트 템플릿 생성
    PromptTemplate promptTemplate = new PromptTemplate("""
        다음 도시들에서 유명한 호텔 3개를 출력하세요.
        {cities}
        {format}
        """);

    // 프롬프트 생성
    Prompt prompt = promptTemplate.create(Map.of(
        "cities", cities,
        "format", beanOutputConverter.getFormat()));

    // LLM의 JSON 출력 얻기
    String json = chatClient.prompt(prompt).call().content();

    // JSON을 List<Hotel>로 변환
    List<Hotel> hotelList = beanOutputConverter.convert(json);
    return hotelList;
}
```

- **1** `BeanOutputConverter<List<Hotel>>` 생성 시 `ParameterizedTypeReference<List<Hotel>>(){}`를 사용합니다. `ParameterizedTypeReference`는 자바 제네릭 타입 정보를 런타임에 보존하기 위해 사용합니다. 같은 도시 이름과 출력 형식 지침을 자리 표시자에 바인딩해서 사용합니다.
- **2** 프롬프트 템플릿에서 프롬프트를 생성할 때, 도시 이름과 출력 형식 지침을 자리 표시자에 바인딩해서 사용합니다. 출력 형식 지침을 제공해야 합니다.
- **3** LLM의 JSON 출력을 `List<Hotel>`로 변환합니다.

### 고수준(High Level) 방식

```java
public List<Hotel> genericBeanOutputConverterHighLevel(String cities) {
    List<Hotel> hotelList = chatClient.prompt()
        .user("""
            %s
            다음 도시들에서 유명한 호텔 3개를 출력하세요.
            """.formatted(cities))
        .call()
        .entity(new ParameterizedTypeReference<List<Hotel>>() {});
    return hotelList;
}
```

- **1** `entity(new ParameterizedTypeReference<List<Hotel>>(){})` 메서드로 변환합니다. 코드에서는 `BeanOutputConverter`를 사용하는 것은 없지만, 저수준 방식과 마찬가지로 형식 지침을 메시지에 추가하고, LLM의 JSON 출력을 변환합니다.

### 컨트롤러

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerParameterizedTypeReference {
    // ##### 필드 #####
    @Autowired
    private AiServiceParameterizedTypeReference aiService;

    @PostMapping(
        value = "/generic-bean-output-converter",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public List<Hotel> genericBeanOutputConverter(@RequestParam("cities") String cities) {
        List<Hotel> hotelList = aiService.genericBeanOutputConverterLowLevel(cities);
        // List<Hotel> hotelList = aiService.genericBeanOutputConverterHighLevel(cities);
        return hotelList;
    }
}
```

- 프로젝트를 실행합니다. 브라우저로 `http://localhost:8080`에서 [generic-bean-output-converter] 버튼을 클릭하여 쉼표로 구분된 도시 이름을 입력하면서 저수준과 고수준 방식의 메소드를 번갈아가면서 테스트합니다.

---

## 4.5 Map으로 변환 (MapOutputConverter)

LLM의 출력을 `Map<String, Object>` 객체로 변환하고 싶다면, `MapOutputConverter`를 사용할 수 있습니다. 이 변환기는 LLM이 JSON 출력을 생성하고, LLM의 출력을 `Map<String, Object>`로 변환합니다.

- **01** `service/AiServiceMapOutputConverter.java` 파일을 엽니다. 이 서비스 클래스는 주어진 호텔의 정보를 LLM에게 물어보고, `Map<String, Object>`로 변환합니다.
- **02** 먼저 저수준 방식의 `mapOutputConverterLowLevel()` 메소드를 보겠습니다.

### 저수준(Low Level) 방식

```java
public Map<String, Object> mapOutputConverterLowLevel(String hotel) {
    // 구조화된 출력 변환기 생성
    MapOutputConverter mapOutputConverter = new MapOutputConverter();

    // 프롬프트 템플릿 생성
    PromptTemplate promptTemplate = new PromptTemplate(
        "호텔 {hotel}에 대해 알려주세요. {format}");

    // 프롬프트 생성
    Prompt prompt = promptTemplate.create(Map.of(
        "hotel", hotel,
        "format", mapOutputConverter.getFormat()));

    // LLM의 JSON 출력 얻기
    String json = chatClient.prompt(prompt).call().content();

    // List<String>으로 변환
    Map<String, Object> hotelInfo = mapOutputConverter.convert(json);
    return hotelInfo;
}
```

- **1** LLM의 출력을 `Map<String, Object>`로 변환하는 `MapOutputConverter`를 생성합니다.
- **2** 프롬프트 템플릿에서 호텔 이름과 출력 형식 지침을 자리 표시자에 바인딩해서 사용합니다. LLM에게 형식 지침을 제공해야 합니다.
- **3** LLM의 JSON 출력을 `Map<String, Object>`로 변환합니다.

### 고수준(High Level) 방식

```java
public Map<String, Object> mapOutputConverterHighLevel(String hotel) {
    Map<String, Object> hotelInfo = chatClient.prompt()
        .user("%s호텔 정보를 알려주세요.".formatted(hotel))
        .call()
        .entity(new MapOutputConverter());
    return hotelInfo;
}
```

- **1** `entity(new MapOutputConverter())` 메서드: `MapOutputConverter` 객체를 사용하면 형식 지침을 메시지에 자동으로 포함시킵니다. 저수준 방식과 마찬가지로, 출력 형식 지침을 메시지에 추가합니다. 그리고 LLM의 출력을 `Map<String, Object>`로 변환합니다.

### 컨트롤러

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerMapOutputConverter {
    // ##### 필드 #####
    @Autowired
    private AiServiceMapOutputConverter aiService;

    @PostMapping(
        value = "/map-output-converter",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public Map<String, Object> mapOutputConverter(@RequestParam("hotel") String hotel) {
        Map<String, Object> hotelInfo = aiService.mapOutputConverterLowLevel(hotel);
        // Map<String, Object> hotelInfo = aiService.mapOutputConverterHighLevel(hotel);
        return hotelInfo;
    }
}
```

- **2** 저수준과 고수준 방식의 메소드를 주석처리하여 번갈아 사용합니다.

- **05** 프로젝트를 실행합니다. 브라우저로 `http://localhost:8080`에서 [map-output-converter] 버튼을 클릭하고, 호텔 이름을 입력한 후 저수준과 고수준 방식의 메소드를 번갈아가면서 테스트합니다.

---

## 4.6 시스템 메시지와 함께 사용

LLM에게 지시한 메시지에 시스템 메시지를 일반적으로 포함시키지만, `entity()`는 사용자 메시지에 출력 형식 지침을 추가합니다. LLM의 출력 형식을 두 메시지에 모두 포함시켜, 서술식 설명과 예시는 시스템 메시지에, LLM의 출력 형식을 구체적으로 지정하는 2차 지침을 사용자 메시지에 추가하도록 구조화된 출력을 강경하겠습니다.

프로젝트 소스 코드를 보면서 설명하겠습니다.

- **01** `dto/ReviewClassification.java` 파일을 엽니다. 이 클래스는 영화 리뷰를 분류하는 타입입니다. `entity()`로 구체적인 타입 정보를 제공해 주면 구조화된 출력 형식 지침이 2차 사용자 메시지에 추가됩니다.

```java
@Data
public class ReviewClassification {
    // ##### 필드 #####
    private String review;
    private Sentiment classification;
}
```

```java
public enum Sentiment {
    POSITIVE, NEUTRAL, NEGATIVE
}
```

- **02** `service/AiServiceSystemMessage.java` 파일을 엽니다. 이 서비스 클래스에는 출력 형식 지침을 포함시키고, `BeanOutputConverter`를 이용해서 시스템 메시지를 사용자 메시지에 형식 지침을 포함시켜 `ReviewClassification`을 분류합니다. 그리고 최종적으로 LLM 출력을 `ReviewClassification`으로 변환합니다.

- **03** `classifyReview()` 메소드를 보겠습니다.

```java
public ReviewClassification classifyReview(String review) {
    ReviewClassification reviewClassification = chatClient.prompt()
        .system("""
            영화 리뷰를 [POSITIVE, NEUTRAL, NEGATIVE] 중에서 하나로 분류하세요.
            주어진 '%s' 형식으로 출력해주세요.
            """)
        .user("%s".formatted(review))
        .options(ChatOptions.builder().temperature(0.0).build())
        .call()
        .entity(ReviewClassification.class);
    return reviewClassification;
}
```

- **1** 서술식 설명으로 시스템 메시지를 생성하고 프롬프트에 추가합니다.
- **2** `entity()`로 `ReviewClassification` 타입을 생성해 2차 출력 형식 지침을 생성해서 사용자 메시지 프롬프트에 추가합니다.
- **3** `entity(ReviewClassification.class)`로 LLM 출력을 `ReviewClassification`으로 변환하고 프롬프트에 추가합니다.

- **04** `controller/AiControllerSystemMessage.java` 파일을 엽니다. 이 컨트롤러 클래스에는 `/ai/system-message` 요청 매핑 메서드가 다음과 같이 선언되어 있습니다.

```java
@RestController
@RequestMapping("/ai")
@Slf4j
public class AiControllerSystemMessage {
    // ##### 필드 #####
    @Autowired
    private AiServiceSystemMessage aiService;

    @PostMapping(
        value = "/system-message",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public ReviewClassification reviewClassification(@RequestParam("review") String review) {
        ReviewClassification reviewClassification = aiService.classifyReview(review);
        return reviewClassification;
    }
}
```

- **05** 프로젝트를 실행합니다. 브라우저로 `http://localhost:8080`에서 [system-message] 버튼을 클릭하고 테스트합니다. 리뷰를 입력하면 다음과 같이 컨트롤러가 반환한 `ReviewClassification`이 반환된 것을 볼 수 있습니다.

실행 결과:
```json
{
    "review": "...",
    "classification": "POSITIVE"
}
```
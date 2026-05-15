# Hello RAG — Spring Boot 실습

> Java 21 / Spring Boot 3.3.5 / Gradle 기반으로 RAG 파이프라인을 처음부터 구현한다.
> 각 Task를 순서대로 따라 치면 동작하는 RAG 앱이 완성된다.

---

## 개념 요약

```
사용자 질문
    ↓
① Embedding → 벡터 변환
    ↓
② Vector Store에서 유사 문서 검색
    ↓
③ [문서 + 질문] 조합 → 프롬프트 구성
    ↓
④ Claude API 호출
    ↓
⑤ 답변 반환
```

---

## T1. 프로젝트 생성

[start.spring.io](https://start.spring.io) 에서 아래 설정으로 생성한다.

| 항목          | 값               |
|-------------|-----------------|
| Project     | Gradle - Groovy |
| Language    | Java            |
| Spring Boot | 3.3.5           |
| Java        | 21              |
| Group       | com.example     |
| Artifact    | hello-rag       |
| Packaging   | Jar             |

Dependencies는 아무것도 추가하지 않고 **GENERATE** 클릭 → 압축 해제 후 IDE에서 열기.

---

## T2. build.gradle — Spring AI 의존성 추가

생성된 `build.gradle` 을 열고 아래 내용으로 교체한다.

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.5'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
}

ext {
    set('springAiVersion', '1.0.0')
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.ai:spring-ai-bom:${springAiVersion}"
    }
}

dependencies {
    // Spring Boot 기본
    implementation 'org.springframework.boot:spring-boot-starter'

    // Claude(Anthropic) 연동
    implementation 'org.springframework.ai:spring-ai-anthropic-spring-boot-starter'

    // 로컬 Embedding 모델 (ONNX, API 키 불필요)
    implementation 'org.springframework.ai:spring-ai-transformers-spring-boot-starter'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

저장 후 Gradle 새로고침(sync).

---

## T3. application.yml 작성

`src/main/resources/application.properties` 를 삭제하고, 같은 위치에 `application.yml` 을 새로 만든다.

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-6
          max-tokens: 500
```

> `ANTHROPIC_API_KEY` 는 T9에서 환경변수로 주입한다. 지금은 그대로 둔다.

---

## T4. 메인 클래스 — CommandLineRunner 구현

`src/main/java/com/example/hellorag/HelloRagApplication.java` 를 열고 아래처럼 수정한다.

```java
package com.example.hellorag;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import java.util.List;
import java.util.stream.Collectors;

@SpringBootApplication
public class HelloRagApplication implements CommandLineRunner {

    @Autowired
    private ChatClient.Builder chatClientBuilder;

    @Autowired
    private EmbeddingModel embeddingModel;

    public static void main(String[] args) {
        SpringApplication.run(HelloRagApplication.class, args);
    }

    @Override
    public void run(String... args) {
        // T5에서 채운다
    }
}
```

---

## T5. 문서 준비 및 Vector Store 적재

`run()` 메서드 안에 아래 코드를 추가한다.

```java
// 1. 문서 준비 (실제 서비스에서는 DB나 파일에서 읽어온다)
List<Document> documents = List.of(
                new Document("BootNotification은 충전기가 부팅될 때 중앙 시스템에 전송하는 첫 번째 메시지다."),
                new Document("Heartbeat 메시지는 충전기가 살아있음을 알리기 위해 주기적으로 전송된다."),
                new Document("RemoteStartTransaction은 중앙 시스템이 충전을 원격으로 시작할 때 사용한다."),
                new Document("StatusNotification은 충전기 상태 변화를 중앙 시스템에 알린다."),
                new Document("Authorize는 RFID 카드 인증을 처리하는 메시지다.")
        );

// 2. 인메모리 Vector Store 구성 및 적재
SimpleVectorStore vectorStore = SimpleVectorStore.builder(embeddingModel).build();
vectorStore.add(documents);

System.out.println("✅ 문서 " + documents.size() + "개 적재 완료");
```

---

## T6. 유사 문서 검색

T5 코드 아래에 이어서 추가한다.

```java
// 3. 사용자 질문
String userQuery = "충전기가 처음 켜졌을 때 무슨 메시지를 보내?";

// 4. 유사 문서 검색 (topK: 상위 2개)
List<Document> similar = vectorStore.similaritySearch(
        SearchRequest.builder()
                .query(userQuery)
                .topK(2)
                .build()
);

// 검색 결과 출력
System.out.println("\n=== 검색된 문서 ===");
similar.forEach(d -> System.out.println("- " + d.getFormattedContent()));

// 5. 검색된 문서를 하나의 텍스트로 합치기
String context = similar.stream()
        .map(Document::getFormattedContent)
        .collect(Collectors.joining("\n"));
```

---

## T7. Claude 호출

T6 코드 아래에 이어서 추가한다.

```java
// 6. [문서 + 질문] 조합 → 프롬프트 구성
String prompt = """
                아래 참고 문서를 바탕으로 질문에 답하세요.
                
                [참고 문서]
                %s
                
                [질문]
                %s
                """.formatted(context, userQuery);

// 7. Claude 호출
ChatClient chatClient = chatClientBuilder.build();
String response = chatClient.prompt(prompt).call().content();

System.out.println("\n=== Claude 답변 ===");
System.out.println(response);
```

---

## T8. 완성된 run() 전체 코드 확인

T5 ~ T7 을 합친 최종 `run()` 메서드 전체 모습이다. 빠진 부분이 있으면 여기서 맞춰본다.

```java

@Override
public void run(String... args) {
    // 1. 문서 준비
    List<Document> documents = List.of(
            new Document("BootNotification은 충전기가 부팅될 때 중앙 시스템에 전송하는 첫 번째 메시지다."),
            new Document("Heartbeat 메시지는 충전기가 살아있음을 알리기 위해 주기적으로 전송된다."),
            new Document("RemoteStartTransaction은 중앙 시스템이 충전을 원격으로 시작할 때 사용한다."),
            new Document("StatusNotification은 충전기 상태 변화를 중앙 시스템에 알린다."),
            new Document("Authorize는 RFID 카드 인증을 처리하는 메시지다.")
    );

    // 2. Vector Store 적재
    SimpleVectorStore vectorStore = SimpleVectorStore.builder(embeddingModel).build();
    vectorStore.add(documents);
    System.out.println("✅ 문서 " + documents.size() + "개 적재 완료");

    // 3. 질문 및 검색
    String userQuery = "충전기가 처음 켜졌을 때 무슨 메시지를 보내?";
    List<Document> similar = vectorStore.similaritySearch(
            SearchRequest.builder().query(userQuery).topK(2).build()
    );

    System.out.println("\n=== 검색된 문서 ===");
    similar.forEach(d -> System.out.println("- " + d.getFormattedContent()));

    String context = similar.stream()
            .map(Document::getFormattedContent)
            .collect(Collectors.joining("\n"));

    // 4. Claude 호출
    String prompt = """
            아래 참고 문서를 바탕으로 질문에 답하세요.
            
            [참고 문서]
            %s
            
            [질문]
            %s
            """.formatted(context, userQuery);

    String response = chatClientBuilder.build().prompt(prompt).call().content();
    System.out.println("\n=== Claude 답변 ===");
    System.out.println(response);
}
```

---

## T9. 환경변수 설정 및 실행

터미널에서 프로젝트 루트로 이동 후 실행한다.

```bash
# Anthropic API 키 설정
export ANTHROPIC_API_KEY=sk-ant-xxxxxxxx

# 실행
./gradlew bootRun
```

---

## T10. 기대 출력 확인

정상 실행되면 아래와 같은 출력이 나온다.

```
✅ 문서 5개 적재 완료

=== 검색된 문서 ===
- BootNotification은 충전기가 부팅될 때 중앙 시스템에 전송하는 첫 번째 메시지다.
- StatusNotification은 충전기 상태 변화를 중앙 시스템에 알린다.

=== Claude 답변 ===
충전기가 처음 켜졌을 때는 BootNotification 메시지를 전송합니다.
이 메시지는 충전기가 부팅 완료 후 중앙 시스템에 자신의 존재를 알리는 첫 번째 신호입니다.
```

---

## 핵심 정리

| 역할         | 담당 컴포넌트                           |
|------------|-----------------------------------|
| 문서 → 벡터 변환 | `EmbeddingModel` (로컬 ONNX, 자동 주입) |
| 벡터 저장 / 검색 | `SimpleVectorStore`               |
| 프롬프트 조합    | 직접 문자열 포맷팅                        |
| LLM 호출     | `ChatClient` (Anthropic Claude)   |

다음 단계: 문서를 하드코딩 대신 파일/DB에서 읽어오고, `SimpleVectorStore` 를 실제 Vector DB(pgvector, Pinecone 등)로 교체하면 프로덕션 RAG가 된다.
# 2024년 12월 9일 (월)
> Protobuf

## 개요
Protobuf는 구글에서 만든 데이터를 직렬화 혹은 역직렬화할 수 있는 데이터 구조이다.

이렇게만 보면 이미 대중화되어있는 JSON과 차이가 있는지 모르겠지만, 이 구조를 적용하였을 때 JSON보다 약 30% 이상 좋은 성능을 보여준다고 한다.

즉, 그냥 성능이 좋은 JSON이라 생각하자.

## 스키마
Protobuf를 사용하기 위해서는 전용 스키마 문법을 따라야 한다.

```protobuf
syntax = "proto3";

message Person {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

> 이 스키마를 자바 레코드 코드로 바꾸면 아래와 같다.
> ```java
> record Person(int id, String name, String email)
> ```

위 Protobuf 스키마를 실질적으로 사용할 수 있는 코드로 바꿔야하는데, 이 때 프로그래밍 가능한 대부분의 언어로 전환할 수 있다.

```bash
brew install protoc

# 자바
protoc --java_out=src/main/java person.proto

# 코틀린
protoc --kotlin_out=src/main/kotlin person.proto

# 자바스크립트
protoc --js_out=import_style=commonjs,binary:. person.proto
```

## 통신
아래 예제에서 Spring Boot와 React가 통신하는 코드이다.

### 백엔드
일단 protobuf를 사용하기 위해서 아래 라이브러리를 추가해야 한다.

```gradle
dependencies {
    implementation("com.google.protobuf:protobuf-java:3.22.2")
    implementation("com.google.protobuf:protobuf-java-util:3.22.2")
}
```

또한, buffer로 오는 데이터를 처리하기 위해서 스프링 부트에서 데이터를 변환해주는 `HttpMessageConveter`를 Bean으로 등록해야 한다.

```java
@Bean
public ProtobufHttpMessageConverter protobufHttpMessageConverter() {

    return new ProtobufHttpMessageConverter();
}
```

이제 앞으로 `produce` 혹은 `comsume` 즉, 데이터가 서버로 들어오거나 나갈 때 Protobuf를 통해 통신할 수 있다.

```java
@GetMaping(consumes = [MediaType.APPLICATION_PROTOBUF_VALUE], produces = [MediaType.APPLICATION_PROTOBUF_VALUE])
public void get(@RequestBody data: DataDTO) {

    // TODO: Implementation
}
```

### 프론트엔드
백엔드에서 라이브러리를 추가했듯이 프론트엔드(NodeJS)에서도 추가해야한다.

```bash
npm install google-protobuf
```

그럼 이제 아래처럼 스키마를 사용할 수 있다.

```js
import { Person } from "./protobuf/person";

const response = await axios.get('http://localhost:8080/api', {
    headers:{
        'Accept': 'application/x-protobuf',
        'MediaType': 'application/x-protobuf',
    },
    responseType: "arraybuffer",
});

const bytes = new Uint8Array(response.data);
const person = Person.deserializeBinary(bytes)
```
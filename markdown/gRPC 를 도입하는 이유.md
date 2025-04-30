# gRPC 를 도입하는 이유에 대해서 알아보자 (feat. HTTP 1.0 vs HTTP 1.1 vs HTTP 2.0)

HTTP 1.0, HTTP 1.1 의 문제와 gRPC 가 MSA 구조에 도입되면 어떤 이점이 있는지 알아보자.

---

## Monolithic → MSA 전환시 Network Latency

Monolithic 아키텍처에서는 하나의 Machine 에서 동일한 프로세스 내에서 실행되고, 각 서비스 간의 통신은 **메서드 호출** 로 이루어지므로 별도의 **네트워크 통신이 필요 없다.**  
또한, 하나의 어플리케이션의 모든 서비스와 모듈이 동일한 메모리 공간에서 실행된다.

하지만, MSA 구조에서는 동일 장비가 아닐 수도 있는 여러 장비에 각각의 프로세스로 분리되다보니,  
보통 **REST 통신**을 통해 메시지를 주고 받는 구조가 되는데, 이 때 **잠재적인 latency** 가 존재한다.  
응답속도가 저하된다는 단점이 존재하게 된다.  

## Q. 어떤 요인으로 인해 응답 속도 저하가 발생할까?

A. HTTP 는 기본적으로 TCP 위에서 동작하기 때문에,  
데이터 송수신에 앞서서 `3-way handshake`, 종료 시 `4-way handshake` 과정을 거친다.  
MSA 구조에서 서버간 통신이 빈번하면 매번 연결을 맺고 끊는 비효율이 발생하여 latency 가 증가한다.

---

### Tip) TCP 연결 / 종료 과정

#### TCP Connection Process (3-way handshake)

![image](https://github.com/user-attachments/assets/a9182551-5bce-45f7-9250-06f60aa41634)


1. **SYN**  
   - Client → Server : SYN (ISN=1000)

2. **SYN + ACK**  
   - Server → Client : SYN-ACK (ISN=2000, ACK=1001)

3. **ACK**  
   - Client → Server : ACK (ACK=2001)

#### TCP Termination Process (4-way handshake)

1. **FIN**  
   - Client → Server : FIN

2. **ACK**  
   - Server → Client : ACK

3. **FIN**  
   - Server → Client : FIN

4. **ACK**  
   - Client → Server : ACK

HTTP 1.0 은 요청과 응답마다 Connection 을 맺고 끊어야 해서 latency 가 발생한다.

---

## HTTP 1.1

HTTP 1.0 은 각 요청마다 새로운 TCP 연결을 맺고 끊는다.  
이로 인해 `3-way handshake` 와 `4-way handshake` 에 의한 오버헤드가 발생하고, 요청/응답이 순차적으로 처리되어 latency 가 크다.

HTTP 1.1 에서는 `Persistent Connection` 과 `Pipelining` 을 통해 개선되었다.

### 1. Persistent Connection (지속 연결)

![image](https://github.com/user-attachments/assets/f20c1bea-5ee4-461d-82dd-734d91666a3e)


- 연결을 한 번 맺고 여러 요청을 처리할 수 있다.
- `Connection: keep-alive` 헤더를 통해 연결이 유지된다.

### 2. Pipelining

![image](https://github.com/user-attachments/assets/a923020d-ec01-4db4-8c7b-6f9cac8ffee1)


- 클라이언트가 여러 요청을 연속으로 보내고, 서버는 순서대로 처리한다.
- 하나의 연결에서 다수의 요청과 응답이 가능하여 latency 를 줄인다.
- 단, 응답 순서가 맞지 않으면 지연이 발생할 수 있다.

> 완전한 멀티플렉싱이 아니기 때문에 순차적 처리로 인한 지연이 여전히 존재한다.

---

## HTTP 1.1 의 문제점

### 1. Head-of-Line Blocking (HOLB)

![image](https://github.com/user-attachments/assets/3adbff2e-0bcf-41ba-b267-be721a44d28c)


- 요청을 순차적으로 처리해야 하므로, 첫 요청이 지연되면 나머지 요청도 지연된다.

### 2. Connection 관리의 비효율성

![image](https://github.com/user-attachments/assets/41ad54fb-1f17-4746-b9e3-8a103a7a9cf0)


- 여러 클라이언트가 Persistent Connection 을 유지하면 서버가 많은 연결을 동시에 관리해야 해 부하 발생

### 3. Header Overhead

![image](https://github.com/user-attachments/assets/7989fc47-7d37-4d7a-94b3-354f713d8bca)


- 각 요청마다 동일한 헤더를 반복 전송해 네트워크 대역폭 낭비

---

## HTTP 2.0

### 1. Multiplexing

![image](https://github.com/user-attachments/assets/d03bd802-9034-4faf-9cfb-8603d6240414)


- 하나의 연결에서 여러 요청을 병렬 처리할 수 있어 HOLB 문제를 해결
- 스트림마다 우선 순위 지정 가능

> 단, TCP 기반이라 Transport Layer 에서는 HOLB 가 여전히 존재함  
> 완전한 HOLB 해결은 HTTP/3(QUIC) 에서 가능

### 2. Header 압축 (HPACK)

![image](https://github.com/user-attachments/assets/ff0d50bf-35b7-4a74-a59d-98bb32dbb52c)


- 반복되는 헤더를 압축하여 데이터 전송량 및 대역폭 절약

### 3. Server Push

![image](https://github.com/user-attachments/assets/dfd7142c-d021-4fd1-9fb3-02aed966b5f3)


- 클라이언트 요청 없이 필요한 리소스를 서버가 미리 전송 가능  
- 페이지 렌더링 속도 향상  
- 단, 과도하면 오히려 비효율 발생

---

## HTTP 1.1 과 HTTP 2.0 차이점

![image](https://github.com/user-attachments/assets/4c4bf3ff-d262-4805-9fb0-a9d5f0fc5794)


| 특징 | HTTP 1.1 | HTTP 2 |
|------|----------|--------|
| Connection | Persistent | Persistent + Multiplexing |
| 처리 방식 | 순차 처리 | 병렬 처리 |
| Header | 압축 없음 | HPACK |
| Protocol | Text | Binary |
| 우선순위 | 없음 | 가능 |
| Server Push | 없음 | 있음 |

---

## REST API 의 단점

### 1. Serialization / Deserialization

![image](https://github.com/user-attachments/assets/2cafe299-7cfb-4ba9-956a-466ea1043e10)


- JSON 은 텍스트 기반이라 직렬화/역직렬화 시 CPU 사용량이 높다  
- 빈번한 통신 시 성능 저하 유발

### 2. 타입 제약

- 날짜, 시간, 바이너리 처리에 추가 파싱 필요  
- 타입 불일치 발생 가능

### 3. 데이터 중복

- key-value 쌍 반복으로 데이터 크기 증가

---

## Q. gRPC 에서는 REST 의 단점을 어떻게 해결 했을까?

A. gRPC 는 `protobuf(Protocol Buffers)` 를 사용하여 해결

### 1. Serialization / Deserialization

![image](https://github.com/user-attachments/assets/888eaed3-3867-405c-aac6-8eccf86a110a)


- 바이너리 형식으로 인코딩되어 JSON 보다 크기 작고 빠르다

### 2. 타입 제약

![image](https://github.com/user-attachments/assets/d1b3c1da-d514-4d8a-9d2e-66d887d01fea)


- 각 필드의 타입을 명확히 정의하여 파싱 오류 방지

### 3. 데이터 중복

![image](https://github.com/user-attachments/assets/13ffe964-6a5d-4d26-90fa-4e15e3f7c9d7)


- 필드 이름이 스키마에만 정의되므로 데이터 중복이 없다

### JSON vs Protobuf 벤치마크

| Benchmark | Mode | Score (ops/s) |
|-----------|------|----------------|
| JSON | thrpt | 3550.345 |
| Protobuf | thrpt | 4203.831 |

---

## Tips) gRPC 서버와 클라이언트 동작 원리

![image](https://github.com/user-attachments/assets/f07f3970-7832-4532-961e-794de14c3be1)


### 서버

- proto 파일로 서비스 인터페이스 정의
- 서버는 해당 인터페이스를 구현한 메서드 제공

### 클라이언트

- 서버와 통신할 `stub` 생성
- 메서드 호출 → 네트워크 요청 변환 → 서버 응답

### 예시 proto 파일

```proto
syntax = "proto3";

option java_package = "lego.example";
option java_outer_classname = "PersonProto";

service PersonService {
  rpc GetPerson(PersonRequest) returns (PersonResponse);
}

message PersonRequest {
  string name = 1;
}

message PersonResponse {
  Person person = 1;
}

message Person {
  string name = 1;
  int32 age = 2;
  string phoneNumber = 3;
  Address address = 4;
}

message Address {
  string city = 1;
  string zipCode = 2;
}
```

### gRPC 서버 구현 예시 (Java)

```java
public final class PersonServer {
  public static void main(String[] args) throws Exception {
    Server server = ServerBuilder.forPort(50051)
        .addService(new PersonServiceImpl())
        .build()
        .start();
    server.awaitTermination();
  }

  static class PersonServiceImpl extends PersonServiceGrpc.PersonServiceImplBase {
    @Override
    public void getPerson(PersonRequest request, StreamObserver<PersonResponse> responseObserver) {
      Person person = Person.newBuilder()
          .setName(request.getName())
          .setAge(35)
          .setPhoneNumber("010-1234-1234")
          .setAddress(Address.newBuilder().setCity("Seoul").setZipCode("1234").build())
          .build();

      responseObserver.onNext(PersonResponse.newBuilder().setPerson(person).build());
      responseObserver.onCompleted();
    }
  }
}
```

### gRPC 클라이언트 구현 예시 (Java)

```java
public final class PersonClient {
  public static void main(String[] args) {
    ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 50051)
        .usePlaintext()
        .build();

    PersonServiceBlockingStub stub = PersonServiceGrpc.newBlockingStub(channel);

    PersonRequest request = PersonRequest.newBuilder().setName("eottabom").build();
    PersonResponse response = stub.getPerson(request);

    System.out.println(response);

    channel.shutdown();
  }
}
```

---

## 결론

- MSA 에서는 네트워크 통신 증가로 인해 Latency 이슈 발생
- HTTP/2 는 Multiplexing 과 Header Compression 으로 이를 일부 개선
- gRPC 는 HTTP/2 기반으로 위 장점 유지
- gRPC + Protobuf 는 JSON 의 성능 문제를 해결하고, 직렬화 비용과 데이터 전송 크기를 줄임
- 명확한 타입 정의와 언어/플랫폼 중립성으로 유지보수가 쉬움

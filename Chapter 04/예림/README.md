# Chapter 4. 카프카 상세 개념 설명

## 4.1 토픽과 파티션
- 토픽은 카프카의 시작과 끝이다. 카프카를 사용하는 것은 토픽을 만들면서 시작된다.
토픽에 대해 잘 이해하고 설정을 잘하는 것이 카프카를 통한 데이터 활용도를 높이는 길이다.


### 4.1.1 적정 파티션 개수
- 토픽 생성 시 파티션 개수 고려사항 3가지
  - 데이터 처리량
  - 메시지키 사용 여부
  - 브로커, 컨슈머 영향도
- 데이터 처리 속도를 올리는 방법은 2가지다.
- 첫 번쨰는, 컨슈머의 처리량을 늘리는 것
  - 서버의 사양을 올리는 스케일 업
  - gc 튜닝
- 두 번째는, 컨슈머를 추가해서 병렬 처리량을 늘리는 것
  - 데이터 처리량을 늘리는 가장 확실한 방법이다.
  - 프로듀서가 보내는 데이터가 초당 1,000레코드이고 컨슈머가 처리할 수 있는 데이터가 초당 100레코드라면 최소한 필요한 파티션 개수는 10개이다.
  - 만약 컨슈머 데이터 처리량이 프로듀서가 보내는 데이터보다 적다면 컨슈머 랙이 생기고, 데이터 처리 지연이 발생한다.
  - 카프카 컨슈머를 개발할 때 내부 로직을 고민하여 시간 복잡도를 줄이기 위해 다양한 노력을 하는 것도 좋다.
- 파티션 개수를 무조건 늘리는 것만이 능사가 아니다. 파티션 개수를 늘리게 됨에 따라 컨슈머, 브로커의 부담이 있기 때문이다.
- 메시지 키를 사용함과 동시에 데이터 처리 순서를 지켜야 하는 경우에 대해 고려해야 한다.
  - 메시지 키 사용 여부는 데이터 처리 순서와 밀접한 연관이 있다.
<img width="273" height="88" alt="스크린샷 2025-08-03 오후 11 49 13" src="https://github.com/user-attachments/assets/a2174810-2de3-4ad1-9fd8-888714107655" />

### 4.1.2 토픽 정리 정책 (cleanup.policy)
- 토픽의 데이터는 시간 또는 용량에 따라 삭제 규칙을 적용할 수 있다.
- 데이터를 오래동안 삭제하지 않으면 저장소 사용량이 지속적으로 늘어나게 된다.
- cleanup.policy 옵션을 사용하여 데이터를 삭제할 수 있는 2가지 삭제 정책을 제공한다.

  - delete(삭제): 데이터의 완전 삭제
  - compact(압축): 동일 메시지 키의 가장 오래된 데이터를 삭제하는 것

#### 토픽 삭제 정책(delete policy)
- 토픽을 운영하면 일반적으로 대부분의 토픽의 cleanup.policy를 delete로 설정한다.
- 토픽의 데이터를 삭제할 때는 세그먼트 단위로 삭제를 진행하다.
- 세그먼트는 토픽의 데이터를 저장하는 명시적인 파일 시스템 단위이다. 

#### 토픽 압축 정책(compact policy)
- 압축이란 메시지 키별로 해당 메시지 키의 레코드 중 오래된 데이터를 삭제하는 정책을 뜻한다.
- 메시지 키를 기준으로 오래된 데이터를 삭제하기 때문에 삭제 정책과 다르게 1개 파티션에서 오프셋의 증가가 일정하지 않을 수 있다.
- 즉, 1부터 10까지 오프셋이 있고, 4,5,6이 동일한 메시지 키를 가질 경우, 오프셋과 관계없이 중간에 있는 4번, 5번 오프셋의 레코드가 삭제될 수 있다는 뜻이다.
- 4,5,6이 동일한 메시지 키를 가지고 있는데, 6번에 비해 4번, 5번 오프셋의 레코드는 오래된 데이터이기 때문이다.

### 4.1.3 ISR
- ISR은 리더 파티션과 팔로워 파티션이 모두 싱크가 된 상태를 뜻한다.
- 복제 개수가 2인 토픽을 가정해보자. 이 토픽에는 리더 파티션 1개와 팔로워 파티션이 1개가 존재할 것이다. 리더 파티션에 0부터 3의 오프셋이 있다고 가정할 때, 팔로워 파티션에 동기화가 완료되려면 0부터 3까지 오프셋이 존재해야 한다.
- 리더 파티션과 팔로워 파티션이 동기화된 상태에서는 리더 또는 팔로워 파티션이 위치하는 브로커에 장애가 발생하더라도 데이터를 안전하게 사용할 수 있다.

## 4.2 카프카 프로듀서
### 4.2.1 acks 옵션
- 카프카 프로듀서의 acks 옵션은 0, 1, all(또는 -1) 값을 가질 수 있다.

#### acks=0
- acks를 0으로 설정하는 것은 프로듀서가 리더 파티션으로 데이터를 전송했을 때 리더 파티션으로 데이터가 저장되었는지 확인하지 않는다는 뜻이다.
- 리더 파티션은 데이터가 저장된 이후에 데이터가 몇 번째 오프셋에 저장되었는지 리턴하는데, acks가 0으로 설정되어 있다면 프로듀서는 리더 파티션에 데이터가 저장되었는지 여부에 대한 응답 값을 받지 않는다. 그렇기 때문에 프로듀서가 데이터를 보낸 이후에 이 데이터가 몇 번째 오프셋에 저장되었는지 확인할 수 없다.

#### acks=1
- acks를 1로 설정할 경우 프로듀서는 보낸 데이터가 리더 파티션에만 정상적으로 적재되었는지 확인한다.
- 만약 리더 파티션에 정상적으로 적재되지 않았다면 리더 파티션에 적재될 때까지 재시도할 수 있다. 그러나 리더 파티션에 적재되었음을 보장하더라도 데이터는 유실될 수 있다.
- 왜냐하면 복제 개수를 2이상으로 운영할 경우 리더 파티션에 적재가 완료되어 있어도 팔로워 파티션에는 아직 데이터가 동기화되지 않을 수 있는데, 팔로워 파티션이 데이터를 복제하기 직전에 리더 파티션이 있는 브로커에 장애가 발생하면 동기화되지 못한 일부 데이터가 유실될 수 있기 때문이다.

#### acks=all 또는 acks=-1
- acks를 all 또는 -1로 설정할 경우 프로듀서는 보낸 데이터가 리더 파티션과 팔로워 파티션에 모두 정상적으로 적재되었는지 확인한다.
- 리더 파티션뿐만 아니라 팔로워 파티션까지 데이터가 적재되었는지 확인하기 때문에 0또는 1 옵션보다도 속도가 느리다. 그럼에도 불구하고 팔로워 파티션에 데이터가 정상 적재되었는지 기다리기 때문에 일부 브로커에 장애가 발생하더라도 프로듀서는 안전하게 데이터를 전송하고 저장할 수 있음을 보장할 수 있다.

## 4.3 카프카 컨슈머
### 4.3.1 멀티 스레드 컨슈머
- 카프카는 처리량을 늘리기 위해 파티션과 컨슈머 개수를 늘려서 운영할 수 있다.
- 파티션을 여러 개로 운영하는 경우 데이터를 병렬처리하기 위해서 파티션 개수와 컨슈머 개수를 동일하게 맞추는 것이 가장 좋은 방법이다.
- 토픽의 파티션은 1개 이상으로 이루어져 있으며 1개의 파티션은 1개 컨슈머가 할당되어 데이터를 처리할 수 있다.
- 파티션 개수가 n개라면 동일 컨슈머 그룹으로 묶인 컨슈머 스레드를 최대 n개 운영할 수 있다. 그러므로 n개의 스레드를 가진 1개의 프로세스를 운영하거나 1개의 스레드를 가진 프로세서를 n개 운영하는 방법도 있다.

#### 카프카 컨슈머 멀티 스레드 전략
- 하나의 파티션은 동일 컨슈머 중 최대 1대까지 할당된다. 그리고 하나의 컨슈머는 여러 파티션에 할당될 수 있다.
- 이런 특징을 가장 잘 살리는 방법은 1개의 애플리케이션에 구독하고자 하는 토픽의 파티션 개수만큼 컨슈머 스레드 개수를 늘려서 운영하는 것이다.
- 컨슈머 스레드를 늘려서 운영하면 각 스레드에 각 파티션이 할당되며, 파티션의 레코드들을 병렬처리할 수 있다.

### 4.3.2 컨슈머 랙
- 컨슈머 랙은 토픽의 최신 오프셋과 컨슈머 오프셋 간의 차이다.
- 컨슈머 랙은 컨슈머 그룹과 토픽, 파티션 별로 생성된다.
  - 파티션 3개로 구성된 토픽에 컨슈머가 할당되면 컨슈머 랙은 총 3개가 된다.
- 프로듀서가 보내는 데이터양이 컨슈머의 데이터 처리량보다 크다면 컨슈머 랙은 늘어난다.
- 최솟값은 0으로 지연이 없음을 뜻한다.
- 컨슈머 랙을 확인하는 방법 3가지
  - 카프카 명령어 사용
    - kafka-consumer-groups.sh 명령어 사용
    - 일회성에 그치고 지표를 지속적으로 기록하고 모니터링하기에는 부종
    - 테스트용 카프카에서 주로 사용
  - 컨슈머 애플리케이션에서 `metrics()` 사용
    - KafkaCosumer 인스턴스의 metrics() 메서드 사용
    - 컨슈머 랙 관련 모니터링 지표는 3가지
      - records-lag-max
      - recrds-lag,
      - records-lag-avg
  - `metrics()`로 컨슈머 랙 확인하는 방법은 3가지 문제점이 있다.
    - 컨슈머가 정상 동작할 때만 확인 가능
    - 컨슈머 애플리케이션에 컨슈머 랙 모니터링 코드를 중복해서 작성해야 함
    - 카프카 서드 파티 애플리케이션의 컨슈머 랙 모니터링이 불가능
  - 외부 모니터링 툴 사용
    - 카프카 클러스터 종합 모니터링 툴을 사용하면 카프카 운영에 필요한 다양한 지표를 모니터링할 수 있다.
    - 외부 모니터링 툴을 사용하면 카프카 클러스터에 연결된 모든 컨슈머, 토픽들의 랙 정보를 한 번에 모니터링할 수 있다.
    - 컨슈머의 데이터 처리와는 별개로 지표를 수집하기 프로듀서나 컨슈머의 동작에 영향을 미치지 않는다.

#### 카프카 버로우
- 링크드인에서 공개한 오픈소스 컨슈머 랙 체크 툴
- 카프카 클러스터와 연동하면 REST API를 통해 컨슈머 그룹별 컨슈머 랙 조회 가능
- 다수의 카프카 클러스터를 동시에 연결하여 컨슈머 랙을 확인하기 때문에 한 번의 설정으로 다수의 카프카 클러스터 컨슈머 랙을 확인할 수 있다.
- 컨슈머 랙 지표를 수집, 적재, 알림 설정을 하고 싶다면 별도의 저장소와 대시보드를 구축해야 한다.
- 컨슈머 랙이 임게치에 도달할 때마다 알림을 받는 것은 무의미한 일이기 때문에 버로우에서는 임계치가 아닌 **슬라이딩 윈도우 계산**을 통해 문제가 생긴 파티션과 컨슈머의 상태를 표현한다.
- `컨슈머 랙 평가` : 버로우에서 컨슈머 랙의 상태를 표현하는 것
  - 컨슈머 랙과 파티션의 오프셋을 슬라이딩 윈도우로 계산하면 상태가 정해진다.
<img width="459" height="170" alt="스크린샷 2025-08-12 오후 11 39 18" src="https://github.com/user-attachments/assets/df79c811-2017-4a8d-b7e5-224d4389c1cc" />

- 파티션의 상태를 `OK`, `STALLED`, `STOPPED`로 표현
- 컨슈머의 상태를 `OK`, `WARNING`, `ERROR`로 표현

<img width="421" height="176" alt="스크린샷 2025-08-13 오후 11 58 57" src="https://github.com/user-attachments/assets/5ee2a666-7a15-479b-8cb8-03533fbb0df1" />
- 프로듀서가 지속적으로 데이터를 추가하기 때문에 최신 오프셋은 계속해서 증가한다. 컨슈머는 처리를 하면서 컨슈머 랙이
 증가하지만 다시 컨슈머 랙이 0으로 줄어드는 추세를 볼 수 있다. -> 정상 동작 -> 파티션 OK, 컨슈머 OK

<img width="339" height="149" alt="스크린샷 2025-08-15 오후 11 54 35" src="https://github.com/user-attachments/assets/22df2e49-9637-469e-a986-7c100db0704a" />

- 최신 오프셋과 컨슈머 오프셋의 거리가 계속 벌어지면서 컨슈머 랙은 지속적으로 증가 -> 파티션은 OK, 컨슈머는 WARNING
  - 컨슈머의 데이터 처리량이 데이터양에 비해 적기 때문

<img width="383" height="154" alt="스크린샷 2025-08-25 오후 10 54 35" src="https://github.com/user-attachments/assets/31914522-14f1-433f-b97d-a8077f9011d4" />

- 최신 오프셋이 지속적으로 증가하고 있지만 컨슈머 오프셋이 멈춘 것을 확인할 수 있다. 이로 인해 컨슈머 랙이 급격하게 증가하는 모습을 볼 수 있다. -> 파티션은 STALLED, 컨슈머는 ERROR
  - 확실히 비정상 -> 즉각 조치 필요

#### 컨슈머 랙 모니터링 아키텍처
- 컨슈머 랙을 개별적으로 모니터링하기 위해서 별개의 저장소와 대시보드를 사용하는 것이 효과적
- 무료로 설치할 수 있는 아키텍처
  <img width="323" height="254" alt="스크린샷 2025-08-25 오후 10 56 51" src="https://github.com/user-attachments/assets/6742c301-9147-482a-861f-8c597050ba2d" />
  - 버로우
  - 텔레그래프 : 데이터 수집 및 전달에 특화된 툴
  - 엘라스틱서치 : 컨슈머 랙 정보를 담는 저장소
  - 그라파나
₩. 버로우 설정
  - borrow.toml에 버로우와 연동할 클러스터 정보 입력
2. 텔레그레프 설정
  - input.borrow 플러그인을 사용하면 일정 주기마다 컨슈머 랙 조회 및 정보 적재 가능
  ```
  [[inputs.borrow]]
  ...
  [[output.elasticsearch]]
  ...
3. 인덱스 패턴 등록 - 키바나 사용
4. 그라파나 설정 - 그라파나의 데이터 소스에 엘라스틱서치 정보와 설정한 인덱스 패턴을 넣어서 등록한다.

- 컨슈머 랙 정보는 텔레그래프를 통해 엘라스틱서치에 저장된다.
- 엘라스틱서치에 저장된 데이터는 그라파나를 통해 그래프로 시각화할 수 있다.

### 4.3.3 컨슈머 배포 프로세스
- 중단 배포
  - 컨슈머 애플리케이션을 완전히 종료한 이후에 개선된 코드를 가진 애플리케이션을 배포하는 방식
    - 한정된 서버 자원을 운영하는 기업에 적합
  <img width="423" height="355" alt="image" src="https://github.com/user-attachments/assets/c9b3d238-4515-491b-938b-7c78ca3753cd" />
- 무중단 배포
  - 블루 / 그린, 롤링, 카나리 배포
<img width="381" height="400" alt="스크린샷 2025-08-31 오후 5 14 32" src="https://github.com/user-attachments/assets/e804fc7a-88fa-4586-9eed-55e17f4379e1" />

## 4.4 스프링 카프카

- 카프카를 스프링에서 효과적으로 사용할 수 있도록 만들어진 라이브러리

### 4.4.1 스프링 카프카 프로듀서
- 카프카 템플릿은 프로듀서 팩토리 클래스를 통해 생성할 수 있다.
- 카프카 템플릿 사용 방법
  - 스프링 기본 제공 ㅌㅁ프릿 사용
  - 직접 프로듀서 팩토리로 생성
 
#### 기본 카프카 템플릿
- build.gradle
```gradle
  dependencies {
      ...
      implementation 'org.springframework.kafka:spring-kafka'
      testImplementation 'org.springframework.kafka:spring-kafka-test'
  }
  ```
- application.yml
  ```
  spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      acks: all
  ```
- 스프링 카프카에서 프로듀서를 사용할 경우에는 필수 옵션이 없다. 그렇기 때문에 옵션을 설정하지 않으면 bootstrap-servers는 localhost:9092, key-serializer, value-serializer는 StringSerializer로 자동 설정되어 실행된다.
```java
@SpringBootApplication
@RequiredArgsConstructor
public class SpringProducerApplication implements CommandLineRunner {
    private static String TOPIC_NAME = "test";

    private final KafkaTemplate<Integer, String> template;

    public static void main(String[] args) {
        SpringApplication.run(SpringProducerApplication.class, args);
    }

    @Override
    public void run(String... args) {
        for (int i = 0; i < 10; i++) {
            template.send(TOPIC_NAME, "test" + i);
        }

        System.exit(0);
    }
}
```

#### 커스텀 카프카 템플릿
- 커스텀 카프카 템플릿은 프로듀서 팩토리를 통해 만든 카프카 템플릿 객체를 빈으로 등록하여 사용하는 것이다. 프로듀서에 필요한 각종 옵션을 선언하여 사용할 수 있다.
- 한 스프링 카프카 애플리케이션 내부에 다양한 종류의 카프카 프로듀서 인스턴스를 생성하고 싶다면 이 방식을 사용하면 된다.
- A클러스터로 전송하는 카프카 프로듀서와 B클러스터로 전송하는 카프카 프로듀서를 동시에 사용할 수 있다.


- 커스텀 카프카 템플릿을 사용하여 2개의 카프카 템플릿을 빈으로 등록하여 사용할 수 있다.
```java
@Configuration
public class KafkaTemplateConfiguration {
    /**
     * KafkaTemplate 빈 객체 등록
     * Bean name : customKafkaTemplate
     * @return
     */
    @Bean
    public KafkaTemplate<String, String> customKafkaTemplate() {

        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.ACKS_CONFIG, "all");

        ProducerFactory<String, String> pf = new DefaultKafkaProducerFactory<>(props);

        /* 빈 객체로 사용할 KafkaTemplate 인스턴스를 초기화하고 리턴한다. */
        return new KafkaTemplate<>(pf);
    }
}

```

```java
@SpringBootApplication
@RequiredArgsConstructor
public class SpringProducerApplication implements CommandLineRunner {
    private static String TOPIC_NAME = "test";

    private final KafkaTemplate<Integer, String> customKafkaTemplate;

    public static void main(String[] args) {
        SpringApplication.run(SpringProducerApplication.class, args);
    }

    @Override
    public void run(String... args) {
        ListenableFuture<SendResult<Integer, String>> future = customKafkaTemplate.send(TOPIC_NAME, "test");
        future.addCallback(new KafkaSendCallback<Integer, String>() {
            /* 프로듀서가 보낸 데이터의 브로커 적재 여부를 비동기로 확인 */
            @Override // 정상일 경우
            public void onSuccess(SendResult<Integer, String> result) {

            }

            @Override // 이슈가 발생했을 경우
            public void onFailure(KafkaProducerException ex) {

            }
        });

        System.exit(0);
    }
}
```
- ListenableFuture 인스턴스에 addCallback 함수를 붙여 프로듀서가 보낸 데이터의 브로커의 적재 여부를 비동기로 확인할 수 있다.

### 4.4.2 스프링 카프카 컨슈머
- 스프링 카프카의 컨슈머는 기존 컨슈머를 2개의 타입으로 나누고 커밋을 7가지로 나누어 세분화했다.

- 리스너의 종류에 따라 한번 호출하는 메서드에서 처리하는 레코드의 개수가 달라진다.

- 레코드 리스너
  - 단 1개의 레코드를 처리한다.

  - 스프링 카프카 컨슈머의 기본 리스너
- 배치 리스너 (BatchMessageListener)
  -  기존 카프카 클라이언트 라이브러리의 poll() ㅔㅁ서드로 리턴받은 CnsumerRecords처럼 한번에 여러개 레코드들을 처리한다.
-  그 외
  - 매뉴얼 커밋을 사용할 경우 Acknowledging 붙은 리스너를 사용하고, KafkaConsumer 인스턴스에 직접 접근하여 컨트롤하고 싶다면 ConsumerAware가 붙은 리스너를 사용하면 된다.

#### 7가지 커밋
- 카프카 클라이언트 라이브러리에서 컨슈머를 구현할때 가장 어려운 부분이 커밋을 구현하는 것이다. 카프카 컨슈머에서 커밋을 직접 구현할때 실제 운영 환경에서 다양한 종류의 커밋을 구현해서 사용한다. 스프링 카프카에서는 사용자가 사용할만한 커밋의 종류를 7가지로 세분화하고 미리 로직을 만들어놓았다.

 

- 스프링 카프카에서는 커밋이라 부르지 않고, 'AckMode'라고 부른다. 프로듀서에서 사용하는 acks 옵션과 동일한 어원인 Acknowledgment를 사용하므로 AckMode와 acks를 혼동하지 않도록 주의해야한다.

- 스프링 카프카 컨슈머의 AckMode 기본값은 BATCH이고, 컨슈머의 enable.auto.commit 옵션은 false 로 지정된다.

#### 기본 리스너 컨테이너
- 기본 리스너 컨테이너는 기본 리스너 컨테이너 팩토리를 통해 생성된 리스너 컨테이너를 사용한다.

#### 배치 리스너
배치 리스너는 레코드 리스너와 다르게 KafkaListener로 사용되는 메서드의 파라미터를 List 또는 ConsumerRecords로 받는다. 배치 리스너를 사용하는 경우에는 파라미터를 List 또는 ConsumerRecords로 선언해야한다.

#### 배치 컨슈머 리스너
- 배치 컨슈머 리스너는 컨슈머를 직접 사용하기 위해 컨슈머 인스턴스를 파라미터로 받는다.  동기, 비동기 커밋이나 컨슈머 인스턴스에서 제공하는 메서드들을 활용하고 싶다면 배치 컨슈머 리스너를 사용한다.

#### 배치 커밋 리스너
배치 커밋 리스너는 컨테이너에서 관리하는 AckMode를 사용하기 위해 Acknowledgement 인스턴스를 파라미터로 받는다. Acknowledgement 인스턴스는 커밋을 수행하기 위한 한정적인 메서드만 제공한다. 컨슈머 컨테이너에서 관리하는 AckMode를 사용하여 커밋하고 싶다면 배치 커밋 리스너를 사용하면 된다.

#### 커스텀 리스너 컨테이너
- 서로 다른 설정을 가진 2개 이상의 리스너를 구현하거나 리밸런스 리스너를 구현하기 위해서는 커스텀 리스너 컨테이너를 사용해야한다.

- KafkaListenerContainerFactory
- 커스텀 리스너 컨테이너를 만들기 위해서 스프링 카프카에서 카프카 리스너 컨테이너 팩토리 인스턴스를 생성해야한다.
- 카프카 리스너 컨테이너 팩토리를 빈으로 등록하고 KafkaListener 어노테이션에서 커스텀 리스너 컨테이너 팩토리를 등록하면 커스텀 리스너 컨테이너를 사용할 수 있다.

```java
@Configuration
public class ListenerContainerConfiguration {

  @Bean
  public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String> customContainerFactory() {
  ...
  Map<String, Object> props = new HashMap<>();
  props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
  props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
  props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

  // DefaultKafkaConsumerFactory 는 리스너 컨테이너 팩토리를 생성할때 컨슈머 기본 옵션을 설정하는 용도로 사용된다.
  DefaultKafkaConsumerFactory cf = new DefaultKafkaConsumerFactory<>(props);

  // ConcurrentKafkaListenerContainerFactory 는 리스너 컨테이너를 만들기위해 사용한다. 
  ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();

  // 리밸런스 리스너를 선언한다.
  factory.getContainerProperties().setConsumerRebalanceListener(new ConsumerAwareRebalanceListener() {

    // 커밋이 되기 전에 리밸런스가 발생했을때
    @Override
    public void onPartitionsRevokedBeforeCommit(Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {

    }

    // 커밋이 된 후에 리밸런스가 발생했을때
    @Override
    public void onPartitionsRevokedAfterCommit(Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {

    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {

    }

    @Override
    public void onPartitionsLost(Collection<TopicPartition> partitions) {

    }
  });

  factory.setBatchListener(false);
  factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
  factory.setConsumerFactory(cf);

  return factory;
}
```

- 커스텀 컨슈머 리스너 팩토리 빈 객체를 사용하는 커스텀 리스너 컨테이너를 선언하는 코드
```java
@SpringBootApplication
@Slf4j
public class SpringConsumerApplication_Custom {
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(SpringConsumerApplication_Custom.class);
        application.run(args);
    }


    @KafkaListener(topics = "test",
            groupId = "test-group",
            containerFactory = "customContainerFactory")
    public void customListener(String data) {
        log.info(data);
    }
}
```

## 4.5 정리
- 기본적인 카프카 개념을 아는 것만큼 상세 개념을 아는 것도 중요하다.
- 상세 개념을 통해 파이프라인을 최적화하고 한정된 리소스에서 최고의 성능을 내도록 설정할 수 있기 때문이다.
- 토픽의 적정 파티션 개수를 알아보고 삭제 정책과 ISR에 대한 개념도 알아보았다.
- 프로듀서의 acks 옵션이 파티션의 복제에 어떻게 관여를 하는지도 알아보았고 정확히 한번 전달을 위한 멱등성 프로듀서에 대한 개념도 알아보았다.
- 트랜잭션 프로듀서와 트랜잭션 컨슈머를 통해 트랜잭션 단위의 레코드를 전송할 수 있는 방법도 알아보았다.
- 컨슈머를 멀티 스레드로 운영하기 위한 방법과 운영하면서 필요한 컨슈머 랙 모니터링 그리고 배포 프로세스도 알아보았다.
- 스프링 카프카에 대해서도 알아보았다. 수고했다!

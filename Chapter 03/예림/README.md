# Chapter 03. 카프카 기본 개념 설명

## 3.1 카프카 브로커 / 클러스터 / 주키퍼

- 카프카 브로커는 카르카 클라이언트와 데이터를 주고받기 위해 사용하는 주체이자, 데이터를 분산 저장하여 장애가 발생해도 안전하게 사용할 수 있도록 도와주는 애플리케이션
- 하나의 서버에는 한 개의 카프카 브로커 프로세스가 실행됨
- 한 대로도 기본 기능이 실행되지만 안전하게 보관하고 처리하기 위해 3대 이상의 브로커 서버를 1개의 클러스터로 묶어서 운영
- 카프카 클러스터로 묶인 브로커들은 프로듀서가 보낸 데이터를 안전하게 분산 저장하고 복제

<img src="https://github.com/user-attachments/assets/a5234784-c63f-4365-8dba-d3bc000ceae2" />

### 데이터 저장, 전송
- 프로듀서로부터 데이터를 전달받으면 카프카 브로커는 프로듀서가 요청한 토픽의 파티션에 데이터를 저장하고 컨슈머가 데이터를 요청하면 파티션에 저장된 데이터를 전달한다.
- 프로듀서로부터 전달된 데이터는 파일 시스템에 저장된다.

```bash
$ ls /tmp/kafka-logs 


$ ls /tmp/kafka-logs/hello.kafka-0
```
- 카프카는 메모리나 데이터베이스에 저장하지 않으며 따로 캐시 메모리를 구현하여 사용하지도 않는다. 파일 시스템에 저장하기 때문에 속도 이슈가 발생하지 않을까 의문을 가질 수도 있다.
- 카프카는 페이지 캐시를 사용해 디스크 입출력 속도를 높여 이 문제를 해결했다.
  - 페이지 캐시 : OS에서 파일 입출력의 성능 향상을 위해 만들어 놓은 메모리 영역

### 데이터 복제, 싱크
- 데이터 복제는 카프카를 장애 허용 시스템으로 동작하도록 하는 원동력
- 데이터 복제는 파티션 단위로 이루어진다.


![image](https://github.com/user-attachments/assets/ce7f3391-86b1-4876-8cc2-7e15ce56d7fd)


- 복제 개수의 최솟값은 1(복제 없음), 최댓값은 브로커 개수만큼 설정하여 사용 가능
- 다음은 복제 개수가 3인 경우이다. 복제된 파티션은 리더와 팔로워로 구성된다.
  - 리더 : 프로듀서 또는 컨슈머와 직접 통신
  - 팔로워 : 나머지 복제 데이터를 가지고 있는 파티션

- 팔로워 파티션들은 리더 파티션의 오프셋을 확인하여 현재 자신이 가지고 있는 오프셋과 차이가 나는 경우 리더 파티션으로부터 데이터를 가져와 자신의 파티션에 저장 => 이것이 바로 "복제"!
- 복제되면 저장 용량이 증가하지만 안전하게 사용할 수 있음 (2 이상의 복재 개수를 정하는 것이 중요)
- 안정성이 좋지만 서버는 언제든 장애가 발생할 수 있음

![image](https://github.com/user-attachments/assets/0e23d909-754a-40a7-a72e-115423380865)

### 컨트롤러
- 다수 브로커 중 한 대가 컨트롤러의 역할을 한다.
- 컨트롤러의 역할 : 다른 브로커들 상태 체크, 브로커가 클러스터에서 빠지는 경우 해당 브로커에 존재하는 리더 파티션을 재분배
- 카프카는 지속적으로 데이터를 처리해야 하므로 브로커의 상태가 비정상이라면 빠르게 클러스터에서 빼내는 것이 중요

### 데이터 삭제
- 카프카는 컨슈머가 데이터를 가져가더라도 토픽의 데이터가 삭제되지 않음
- 컨슈머나 프로듀서가 데이터 삭제를 요청할 수도 없고, **오직 브로커만이 데이터 삭제 가능*
- 데이터 삭제는 파일 단위로 이뤄지는데 이 단위를 '로그 세그먼트'라고 함
- 카프카 브로커에 log.segment.bytes 또는 log.segment.ms 옵션에 값이 설정되면 세그먼트 파일이 닫힘.

### 컨슈머 오프셋 저장
- 컨슈머 그룹은 토픽이 특정 파티션으로부터 데이터를 가져가서 처리하고 이 파티션의 어느 레코드까지 가져갔는지 확인하기 위해 오프셋을 커밋한다.


### 코디네이터
- 클러스터의 다수 브로커 중 한 대는 코디네이터의 역할을 수행한다.
- 코디네이터는 컨슈머 그룹의 상태를 체크하고 파티션을 컨슈머와 매칭되도록 분배하는 역할을 한다.
- 카프카 서버에서 직접 주키퍼에 붙으려면 카프카 서버에서 실행되고 있는 주키퍼에 연결해야 하는데, 동일 환경에서 접속하므로 localhost로 접속하며 포트 번호는 주키퍼 기본 포트인 2181을 입력하면 된다.

```bash
$ bin/zookeeper-shell.sh my-kafka:2181
```

## 3.2 토픽과 파티션
- 토픽 : 카프카에서 데이터를 구분하기 위해 사용하는 단위. 1개 이상의 파티션을 소유하고 있다.
- 파티션은 카프카의 병렬 처리의 핵심으로써 그룹으로 묶인 컨슈머들이 레코드를 병렬로 처리할 수 있도록 매칭된다.
- 파티션은 FIFO 구조와 같이 먼저 들어간 레코드는 컨슈머가 먼저 가져가게 된다.

## 3.3 레코드
- 레코드는 타임스탬프, 메시지 키, 메시지 값, 오프셋, 헤더로 구성되어 있다.
- 타임스탬프는 프로듀서에서 해당 레코드가 생성된 시점의 유닉스 타임이 설정된다.
- 메시지 키는 메시지 값을 순서대로 처리하거나 메시지 값의 종류를 나타내기 위해 사용한다.
- 메시지 키의 메시지값은 직렬화되어 브로커로 전송

## 3.4 카프카 클라이언트
- 카프카 클라이언트 라이브러리는 카프카 프로듀서, 컨슈머, 어드민 클라이언트를 제공하는 카프카 클라이언트를 사용하여 애플리케이션을 개발한다.
- 카프카 클라이언트는 라이브러리이기 대문에 자체 라이프사이클을 가진 프레임워크나 애플리케이션 위에서 구현하고 실행해야 한다.

### 3.4.1 프로듀서 API
- 카프카에서 데이터의 시작점은 프로듀서이다.
- 프로듀서 애플리케이션은 카프카에 필요한 데이터를 선언하고 브로커의 특정 토픽과 파티션에 전송한다.
- 프로듀서는 데이터를 전송할 때 리더 파티션을 가지고 있는 카프카 브로커와 직접 통신한다.
- 프로듀서는 데이터를 직렬화하여 카프카 브로커로 보내기 때문에 자바에서 선언 가능한 모든 형태를 브로커로 전송할 수 있다.

#### 카프카 프로듀서 프로젝트 생성
```
dependencies {
  compile 'org.apache.kafka:kafka-clients:2.5.0'
}
```

```java
public class SimpleProducer {
    private final static Logger logger = LoggerFactory.getLogger(SimpleProducer.class);
    private final static String TOPIC_NAME = "test";
    private final static String BOOTSTRAP_SERVERS = "my-kafka:9092";

    public static void main(String[] args) {

        Properties configs = new Properties();
        configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS); // 프로듀서 옵션을 key / value 값으로 선언
        configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        KafkaProducer<String, String> producer = new KafkaProducer<>(configs);

        String messageValue = "testMessage";
        ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, messageValue);
        
        producer.send(record);
        logger.info("{}", record);
        producer.flush();
        producer.close();
    }
}
```
이 프로젝트를 실행하고 로그를 확인해보면 카프카 프로듀서 구동 시 설정한 옵션, 카프카 클라이언트 버전, 전송한 ProducerRecord가 출력된다.

```bash
$ bin/kafka-console-consumer.sh --bootstrap-server my-kafka:9092 --topic test--from-beginning
testMessage
```

#### 프로듀서 중요 개념
- 프로듀서 API를 사용하면 UniformSticyPartitioner와 RoundRobinPartitionar가 있는데 전자가 후자를 발전시킨 것

#### 프로듀서 주요 옵션
- 필수 옵션과 주요 옵션이 있다. 선택 옵션도 중요하다.

- 필수 옵션
  - bootstrap.servers
  - key.serilizer
  - value.serilizer
- 선택 옵션
  - 생략


#### 메시지 키를 가진 데이터를 전송하는 프로듀서
- 메시지 키가 포함된 레코드를 전송하고 싶다면 `ProducerRecord` 생성 시 파라미터로 추가해야 한다.
- 토픽 이름, 메시지 키, 메시지 값을 순서대로 파라미터로 넣고 생성했을 경우 메시지 키가 지정된다.

```java
ProducerRecord<String, String> record = new ProducerRecord<>("test", "Pangyo", "23"); // 토픽 이름, 메시지 키, 메시지 값
```

- 메시지 키가 지정된 데이터는 kafka-console-consumer 명령을 통해 확인할 수 있는데, property 옵션의 print.key, key.separator 값을 주면 출력 화면에서 메시지 키와 메시지 값을 함께 확인할 수 있다.

```bash
$ bin/kafka-console-consumer.sh --bootstrap-server my-kafka:9092 \
--topic test \
--property print.key=true \
--property key.separator="-" \
--from-beginning
null-testMessage
Pangyo-23
```
![스크린샷 2025-06-24 오후 10 15 08](https://github.com/user-attachments/assets/ee76156f-42c0-4528-9958-f1d1e1f750dc)


- 파티션을 직접 지정하고 싶다면 토픽 이름, 파티션 버호, 메시지 키, 메시지 값을 순서대로 파라미터로 넣고 생성하면 된다.
- 파티션 번호는 토픽에 존재하는 파티션 번호로 설정해야 한다.

```java
int partitionNo = 0;
ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, partitionNo, messageKey, messageValue);
```


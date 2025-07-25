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

#### 커스텀 파니셔너를 가지는 프로듀서
- 특정 데이터를 가지는 레코드를 특정 파티션으로 보내야할 때 -> Partitioner 인터페이스 사용
```java
public class CustomPartitioner implements Partitioiner {

  @Override
  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
  if (keyBytes == null) {
    throw new InvalidRecordException("Need message key");
  }
  if ((String)key).equals("Pangyo")) { // 메시지 키가 Pangyo인 경우 파티션 0번으로 지정되도록 0을 리턴
    return 0;
  }

  List<PartitionInfo> partitions = cluser.partitionsForTopic(topic);
  int numPartitions = partitions.size();
  return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions; 
}

... 생략 ...
```
- 리턴값은 주어진 레코드가 들어갈 파티션 번호
- 커스텀 파티셔너를 지정한 경우 ProducerConfig의 PARTITIONAER_CLASS_CONFIG 옵션을 사용자 생성 파티셔너로 설정하여 KafkaProducer 인스턴스를 생성해야 한다.
```java
Properties configs = new Properties();
configs.put (ProducerConfig. BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS) ;
configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.
getName ()) ;
configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.
getName ());
configs.put (ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class);

KafkaProducer<String, String producer = new KafkaProducer<>(configs) ;
```
#### 브로커 정상 전송 여부를 확인하는 프로듀서

- KafkaProducer의 send() 메서드는 Fucture 객체 반환
  - RecordMetadata의 비동기 결과를 표현하는 것
  - ProducerRecord가 카프카 브로커에 정상적으로 적재되었는지에 대한 데이터 포함
- get() 메서드를 사용하면 프로듀서로 보낸 데이터의 결과를 동기적으로 가져올 수 있음
```java
ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, messageValue);
RecordMetadata metadata = producer.send(record).get(); // send()의 결과값은 브로커로부터 응답을 기다렸다가 응답이 오면 RecordMetadata 인스턴스 반환
logger.info(metadata.toString());
```

```java
[main] INFO com.example.ProducerWithSyncCallback - test-2@1
```
- 레코드가 정상적으로 브로커에 적재되었다면 토픽 이름과 파티션 번호, 오프셋 번호가 출력된다.
- 전송된 레코드는 test 토픽 2번 파티션에 적재되었고 오프셋 번호는 1번인 것을 알 수 있다.
- 비동기로 확인할 수도 있다 -> Callback 인터페이스
```java
public class ProducerCallback impelement Callback {
  private final static Logger logger = LoggerFactory.getLogger(ProducerCallback.class);

  @Override
  public void onCompletion(RecordMetadata recordMetadata, Exception e) {
    // 생략
  }
}
```
- onCompletion 메서드는 레코드의 비동기 결과를 받기 위해 사용한다.
- 브로커 적재에 이슈가 생기면 -> Exception에 어떤 에러가 발생했는지 담겨서 메서드 실행
- 에러가 발생하지 않으면 -> RecordMetadata에 해당 레코드가 적재된 토픽 이름과 파티션 번호, 오프셋을 알 수 있음
```java
KafkaProducer<String, String> producer = new KafkaProducer<>(configs);
ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, messageValue);
producer.send(record, new ProducerCallback()); // 여기 주목
```
- 비동기로 결과를 받기 위해서는 ProducerRecord 객체와 함께 사용자 정의 Callback 클래스를 넣으면 된다.
- 주의 : 데이터의 순서가 중요한 경우에는 비동기로 전송 결과를 받아선 안된다.

### 3.4.2 컨슈머 API
- 프로듀서가 전송한 데이터는 카프카 브로커에 적재된다.
- 컨슈머는 적재된 데이터를 사용하기 위해 브로커러부터 데이터를 가져와서 필요한 처리를 한다.

#### 카프카 컨슈머 프로젝트 생성
- 기본 설정으로 생성할 수 있는 오토 커밋 카프카 컨슈머 애플리케이션을 만들어 보자.
```java
public class SimpleConsumer {
  private final static Logger logger = LoggerFactory. getLogger (SimpleConsumer.class);
  private final static String TOPIC_NAME = "test"; // 토픽 이름 지정
  private final static String BOOTSTRAP_SERVERS = "my-kafka:9092";
  private final static String GROUP_ID = "test-group"; // 컨슈머 그룹 이름으로 컨슈머 목적 구분 가능
  public static void main(String[] args) {
    Properties configs = new Properties () ;
    configs.put (ConsumerConfig. BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS) ;
    configs.put (ConsumerConfig. GROUP_ID_CONFIG, GROUP_ID);
    configs.put (ConsumerConfig. KEY_DESERIALIZER_CLASS_CONFIG,
    StringDeserializer.class.getName ()); // 역질렬화 클래스 지정 / 메시지 키, 값에 대해 둘 다 역질렬화 클래스를 지정해야 한다.
    configs.put (ConsumerConfig. VALUE_DESERIALIZER_CLASS_CONFIG,
    StringDeserializer.class.getName ());
    KafkaConsumer<String, String> consumer = new KafkaConsumer<> (configs); 
    consumer. subscribe (Arrays. asList (TOPIC_NAME)) ; // 컨슈머에게 토픽 할당
    while (true) {
      ofSeconds (1));
      ConsumerRecords<String, String> records = consumer .poll (Duration.ofSeconds(1)); // 컨슈머는 poll() 메서드를 호출해 데이터를 가져와서 처리 / 지속적으로 반복 호출해야 하므로 while 루프 사용
      for (ConsumerRecord String, String> record : records) { // for 루프를 통해 poll 메서드가 반환한 ConsumerRecord 데이터 순차 처리
        logger.info ("{}", record);
      }
    }
  }
}
```

#### 컨슈머 중요 개념
- 토픽의 파티션으로부터 데이터를 가져가기 위해 컨슈머를 운영하는 방법은 크게 두 가지
  1. 1개 이상의 컨슈머로 이루어진 컨슈머 그룹을 운영
  2. 토픽의 특정 파티션만 구독하는 컨슈머를 운영

- 1번 방법은 컨슈머 그룹으로부터 격리된 환경에서 안전하게 운영할 수 있도록 도와주는 카프카의 독특한 방식
- 컨슈머 그룹으로 묶인 컨슈머들은 토픽의 1개 이상 파티션들에 할당되어 데이터를 가져갈 수 있다.
  <img width="394" alt="image" src="https://github.com/user-attachments/assets/c716c5bb-99c4-4d24-a8f3-6b06e0c13fc6" />
- 2번 방법은 1개의 파티션은 최대 1개의 컨슈머에 할당 가능하고 1개 컨슈머는 여러 개의 파티션에 할당될 수 있다. 이러한 특징으로 컨슈머 그룹의 컨슈머 개수는 가져가고자 하는 토픽의 파티션 개수보다 같거나 같아야 한다.
  <img width="578" alt="image" src="https://github.com/user-attachments/assets/ec2d1b9b-3c47-4ce5-9cd3-3a3ad412808d" />
- 컨슈머 그룹은 다른 컨슈머 그룹과 격리되는 특징을 갖고 있다.
- 데이터 파이프라인을 운영함에 있어 적절히 컨슈머 그룹을 분리해 운영하는 것은 매우 중요. 자칫하면 지연이 발생할 수 있기 때문.
- 컨슈머 그룹으로 따로 나눌 수 있는 건 최대한 나누자
- 만약 컨슈머 그룹의 컨슈머에 장애가 발생한다면? 장애가 발생한 컨슈머에 할당된 파티션은 장애가 발생하지 않은 컨슈머에 소유권이 넘어간다. => 리밸런싱
- 리밸런싱이 일어나는 상황

  - 컨슈머가 추가되는 상황
  - 컨슈머가 제외되는 상황
- 그룹 조정자 : 리밸런싱을 발동시킴 (카프카 브로커 중 한 대가 역할 수행)
- 데이터 처리의 중복이 발생하지 않게 하기 위해서는 컨슈머 애플리케이션이 오프셋 커밋을 정상적으로 처리했는지 검증해야만 한다.
- 오프셋 커밋은 명시적, 비명시적으로 수행할 수 있는데 기본 옵션은 poll() 메서드가 수행될 때 일정 간격마다 오프셋을 커밋하도록 설정되어 있음 => 비명시 '오프셋 커밋'
  - 편리하지만 호출 이후에 리밸런싱 또는 컨슈머 강제 종료 발생 시 데이터 중복 또는 유실 가능
- 명시적으로 오프셋을 커밋하려면 반환받은 데이터가 처리되고 commitSync() 메서드를 호출하면 된다.
  - 반환된 레코드의 가장 마지막 오프셋을 기준으로 커밋을 수행한다.
- commitSync() 메서드는 응답하는 데 기다리는 시간이 있음 -> commitAsync()도 사용가능. 다만 순서는 보장하지 않고 중복 처리 발생할 수 있음

<img width="512" height="233" alt="image" src="https://github.com/user-attachments/assets/3a8fb9ef-79ad-4119-acd8-06160b5d9505" />

#### 컨슈머 주요 옵션
- 필수 옵션
  - bootstrap.servers
  - key.deserializer
  - value.deserializer
- 선택 옵션
  - 생략

#### 동기 오프셋 커밋
- poll() 호출 이후 commitSync()를 호출하여 오프셋 커밋을 명시적으로 수행할 수 있다.

```java
...(생략)...
consumer.subscribe(Arrays.asList(TOPIC)NAME));
...(생략)...
while(true) {
  ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
  ...(생략)...
  consumer.commitSync();
```
- commitSync는 가장 마지막 레코드 오프셋을 기준으로 커밋 -> 그러므로 poll() 메서드로 받은 모든 레코드의 처리가 끝난 이후 commitSync()를 호출해야 한다.
- 만약 개별 레코드 단위로 매번 오프셋을 커밋하고 싶다면 commitSync()의 파라미터로 Map<TopicPartition, OffsetAndMetadata> 인스턴스를 넣으면 된다.

#### 비동기 오프셋 커밋
- 동기 오프셋 커밋을 사용할 경우 커밋 응답을 기다리는 동안 데이터 처리가 일시적으로 중단되기 때문에 더 많은 데이터를 처리하기 위해서 비동기 오프셋 커밋을 사용할 수 있다.
- 메서드는 commyAsync()

```java
while(true) {
  ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
  ...(생략)...
  consumer.commitAsync();
}
```
- 이것도 poll() 메서드로 리턴된 가장 마지막 레코드를 기준으로 오프셋을 커밋하지만 **커밋이 완료될 때까지 응답을 기다리지 않는다.**
- callback 함수를 파라미터로 받아서 결과를 얻을 수 있다.
```java
consumer.commitAsync(new OffsetCommitCallback() {
  public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception e) {
  if (e != null)
    System.err.printIn("Commit failed");
  else
    System.out.printIn("Commit succeeded"); // 정상 커밋
  if (e != null)
    logger.error ("Commit failed for offsets 0)", offsets, e);
  }
});
```
- OffsetCommitCallback 함수는 commitAsync()의 응답을 받을 수 있도록 도와주는 콜백 인터베이스다.
- 비동기로 받은 커밋 응답은 onComplete() 메서드를 통해 확인할 수 있다.
- 정상 커밋되면 Exception 변수가 null이고, 오프셋 정보가 Map<TopicPartition, OffsetAndMetadata>에 포함되어 있다.

#### 리밸런스 리스너를 가진 컨슈머
- 컨슈머 그룹에서 컨슈머가 추가 또는 제거되면 리밸런스(파티션을 컨슈머에 재할당)가 일어난다.
- poll() 메서드를 통해 반환받은 데이터 처리 전에 리밸런스가 발생하면 데이터를 중복 처리할 수 있다.
- 리밸런스 발생 시 데이터를 중복 처리하지 않게 하기 위해서는 리밸런스 발생 시 처리한 데이터를 기준으로 커밋을 시도해야 한다.
- 리밸런스 감지를 위해 카프카 라이브러리는 ConsumerRebalanceListner 인터페이스를 지원한다.
  - onPartitioinAssigned : 리밸런스가 끝난 뒤에 파티션이 할당 완료되면 호출되는 메서드
  - onPartitionRevoked : 리밸런스가 시작되기 직전에 호출되는 메서드
 
#### 파티션 할당 컨슈머
- 컨슈머를 운영할 때 구독 형태로 사용하는 것 외에도 직접 파티션을 컨슈머에 명시적으로 할당 가능
- assign 메서드 사용
  - 다수의 TopicPartition 인스턴스를 지닌 자바 컬렉션 타입을 파라미터로 받는다.
    TopicPartition : 카프카 라이브러리 내/외부에서 사용되는 토픽, 파티션의 정보를 담는 객체로 사용된다.
- subscribe 메서드가 아닌 assign 메서드 사용

#### 컨슈머에 할당된 파티션 확인 방법
- assignment() 메서드로 확인 가능
- assignment() 메서드는 Set<TopicPartition> 인스턴스를 반환함
    - TopicPartition : 토픽 번호와 파티션 번호가 포함된 객체
```kotlin
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs);
consumer.subscribe(Arrays.asList(TOPIC_NAME));
Set<TopicPartition> assignedTopicPartition = consumer.assignment();
```

#### 컨슈머의 안전한 종료
- 

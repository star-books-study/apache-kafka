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
- 컨슈머를 안전하게 종료하기 위해 KafkaConsumer 클래스는 wakeup() 메서드를 지원한다.
- wakeup() 메서드가 실행된 이후 poll() 메서드가 호출되면 WakeupException 예외가 발생한다.
- WakeupException 예외를 받은 뒤에는 데이터 처리를 위해 사용된 자원들을 해제하면 된다.
- 마지막에는 clos() 메서드를 호출하여 컨슈머가 안전하게 종료되었음을 명시적으로 알려주면 종료 완료

<img width="424" height="254" alt="스크린샷 2025-07-28 오후 11 25 55" src="https://github.com/user-attachments/assets/441fce8e-0e2f-4e34-bf4c-a3190b22845f" />

- wakeup 메서드는 어디서 호출하면 될까?
- 자바 애플리케이션의 경우 코드 내부에서 셧다운 훅을 구현하여 안전한 종료를 명시적으로 구현할 수 있다.
<img width="433" height="191" alt="스크린샷 2025-07-28 오후 11 26 40" src="https://github.com/user-attachments/assets/d4af5a37-2082-4855-9e5e-a4af827f45cc" />

### 3.4.3 어드민 API
- 카프카 클라이언트에서는 내부 옵션들을 설정하거나 조회하기 위해 AdminClient 클래스를 제공한다. AdminClient 클래스를 활용하면 클러스터의 옵션과 관련된 부분을 자동화할 수 있다.

```java
AdminClient admin = AdminClient.create(configs);
```

- 프로듀서 API 또는 컨슈머 API와 다르게 추가 설정 없이 클러스터 정보에 대한 설정만 하면 된다.
- create() 메서드로 KafkaAdminClient를 반환 받는다.
- KafkaAdminClient는 브로커들의 옵션들을 확인, 설정할 수 있는 유틸 클래스다.

## 3.5 카프카 스트림즈
- 카프카 스트림즈 : 토픽에 적재된 데이터를 상태기반 또는 비상태 기반으로 실시간 변환해 다른 토픽에 적재하는 라이브러리
- 스트림즈 애플리케이션 또는 카프카 브로커의 장애가 발생하더라도 정확히 한 번할 수 있도록 장애 허용 시스템을 가지고 있어서 데이터 처리 안정성이 매우 뛰어나다.
- 카프카 클러스터를 운영하면서 실시간 스트림 처리를 해야하는 필요성이 있다면 카프카 스트림즈 애플리케이션으로 개발하는 것을 1순위로 고려하는 것이 좋다.
- 자바 라이브러리로 구현하는 스트림즈 애플리케이션은 JVM 위에서 하나의 프로세스로 실행되기 때문에 분산 시스템이나 스케줄링 프로그램들은 스트림즈 운영에 불필요
<img width="319" height="112" alt="스크린샷 2025-07-31 오후 11 54 30" src="https://github.com/user-attachments/assets/2fb8be9e-da4c-45e1-8a36-fd2ccbd77c90" />
- 스트림즈 애플리케이션은 내부적으로 스레드를 1개 이상 생성할 수 있으며 스레드는 1개 이상의 태스크를 가진다.
- 스트림즈의 태스크는 스트림즈 애플리케이션을 실행하면 생기는 데이터 처리 최소 단위이다.

<img width="335" height="157" alt="스크린샷 2025-07-31 오후 11 55 10" src="https://github.com/user-attachments/assets/8f403afb-234b-4309-a1ed-c7684a3b99d1" />
- 토폴로지란 2개 이상의 노드들과 선으로 이루어진 집합을 뜻한다.
- 토폴로지의 종류로는 링형, 트리형, 성형 등이 있는데 스트림즈에서 사용하는 토폴로지는 트리 형태와 유사하다.
<img width="407" height="152" alt="스크린샷 2025-07-31 오후 11 55 52" src="https://github.com/user-attachments/assets/f7583652-6fb2-4608-978f-9a2c1a73e4ab" />

- `프로세서` : 토폴로지를 이루는 노드
  - 소스 프로세서 : 데이터를 처리하기 위해 최초로 선언해야 하는 노드로, 하나 이상의 토픽에서 데이터를 가져옴
  - 스트림 프로세서 : 다른 프로세서가 반환한 데이터 처리
  - 싱크 프로세서 : 데이터를 특정 카프카 토픽으로 저장하는 역할을 하며 스트림즈로 처리된 데이터의 최종 종착지다.
 <img width="377" height="209" alt="스크린샷 2025-07-31 오후 11 57 52" src="https://github.com/user-attachments/assets/b1b36c65-9065-466f-adaf-b325d3a8d20c" />

    
- `스트림` : 노드와 노드를 이은 선
  - 토픽의 데이터 (컨슈머의 레코드와 동일)

- 스트림즈DSL(Domain Specific Language)과 프로세스 API가 구현할 수 있는 종류
  - 스트림즈DSL로 구현하는 데이터 처리 예시
    - 메시지 값을 기반으로 토픽 분기 처리
    - 지난 10분간 들어온 데이터의 개수 집계
    - 토픽과 다른 토픽의 결합으로 새로운 데이터 생성
  - 프로세서 API로 구현하는 데이터 처리 예시
    - 메시지 값의 종류에 따라 토픽을 가변적으로 전송
    - 일정한 시간 간격으로 데이터 처리

### 3.5.1 스트림즈DSL
- 스트림즈DSL에는 레코드의 흐름을 추상화한 3가지 개념인 KStream, KTable, GlobalTable이 있다.

#### KStream
- 메시지 키와 값으로 구성되어 있고, KStream으로 데이터를 조회하면 **토픽에 존재하는 모든 레코드가 출력된다.**
- 컨슈머로 토픽을 구독하는 것과 동일한 선상에서 사용
<img width="235" height="225" alt="스크린샷 2025-08-01 오후 11 55 28" src="https://github.com/user-attachments/assets/e394f908-4e08-4ac8-b1bf-ca1178020806" />

#### KTable
- KStream과 다르게 **메시지 키를 기준으로 묶어서 사용**
- 유니크한 메시지 키를 기준으로 가장 최신 레코드를 사용
<img width="369" height="238" alt="스크린샷 2025-08-01 오후 11 56 05" src="https://github.com/user-attachments/assets/e8f2429d-1a62-4d3c-92d6-9085e068679c" />

#### GlobalKTable
- KTable과 동일하게 **메시지 키를 기준으로 묶어서 사용**
- 그러나 KTable로 선언된 토픽은 1개 파티션이 1개 데스크에 할당되어 사용되고, GlobalKTable로 선언된 토픽은 모든 파티션 데이터가 각 데스크에 할당되어 사용됨
- 데이터 조인을 수행할 때가 가장 좋은 예
  <img width="301" height="164" alt="스크린샷 2025-08-01 오후 11 57 30" src="https://github.com/user-attachments/assets/74cdf2b5-01a2-49cd-b65e-141212761883" />

- 코파티셔닝 되어있지 않은 2개의 토픽을 조인하는 로직이 담긴 스트림즈 애플리케이션을 실행하면 TopologyException이 발생함
- 조인을 수행하는 KStream과 KTable이 코파티셔닝되어 있지 않으면 KStream 또는 KTable을 리파티셔닝하는 과정을 거쳐야 함
  - 새로운 토픽에 메시지 키를 가지도록 재배열하는 과정
 
<img width="341" height="163" alt="스크린샷 2025-08-01 오후 11 58 11" src="https://github.com/user-attachments/assets/24855971-60dc-43e1-98fa-1481108b2e98" />

- 코파티셔닝 되어 있지 않은 KStream과 KTable을 조인해서 사용하고 싶다면 KTable을 GlobalKTable로 선언하면 됨
- 다만 GlobalKTable로 정의된 모든 데이터를 저장하고 사용하기 때문에 로컬 스토리지 사용량이 네트워크, 브로커에 부하가 생길 수 있으므로 작은 용량에만 사용 권장

#### 스트림즈 DSL - stream(), to()
- 특정 토픽을 KStream 형태로 가져오려면 스트림즈DSL의 stream() 메서드를 사용하면 된다.
  - 소스 프로세서
- KStream의 데이터를 특정 토픽으로 저장하려면 스트림즈DSL의 to() 메서드를 사용한다.
  - 싱크 프로세서
```java
KStream<String, String> streamLog = builder.stream(STREAM_LOG); // stream_log 토픽으로부터 KStream 토픽을 만들기 위해 stream() 메서드 사용
streamLog.to(STREAM_LOG_COPY); // stream_log 토픽을 담은 KStream 객체를 다른 토픽으로 전송하기 위해 to() 메서드 사용
```

#### 스트림즈DSL - filter()
- 메시지 키 또는 메시지 값을 필터링하여 특정 조건에 맞는 데이터를 골라낼 때는 filter() 메서드를 사용하면 된다.
<img width="385" height="309" alt="스크린샷 2025-08-02 오후 11 10 10" src="https://github.com/user-attachments/assets/c18831b3-7b54-4137-a03d-3bf39290503e" />
- filter() 메서드는 Predicate를 파라미터로 받는다.

```java
KStream<String, String> filtereStream = streamLog.filter((key, value) -> value.length() > 5);
filteredStream.to(STREAM_LOG_FILTER);
```
- 더 간소화된 표현 형태(플루언트 인터페이스 스타일)로 작성 가능
```java
streamLog.filter((key, value) -> value.length() > 5).to(STREAM_LOG_FILTER);
```

#### 스트림즈DSL - KTable과 KStream을 join
- 카프카에서는 실시간으로 들어오는 데이터를 조인할 수 있다.
- KTable과 KStream을 소스 프로세서로 가져와서 조인을 수행하는 스트림 프로세서를 거쳐 특정 토픽에 싱크 프로세서로 저장하는 로직을 구현해보자.
<img width="272" height="244" alt="스크린샷 2025-08-02 오후 11 13 07" src="https://github.com/user-attachments/assets/848b97a9-1259-4018-bf3a-36c338d7863d" />

- KTable로 사용할 토픽과 KStream으로 사용할 토픽을 만들 때 둘 다 파티션을 3개로 동일하게 만든다. 파티셔닝 전략은 기본 파티셔너를 사용한다.

> KStream, KTable, GlobalKTable 모두 동일한 토픽이고 다만, 스트림즈 애플리케이션 내부에서 사용할 때 메시지 키와 값을 사용하는 형태를 구분할 뿐인 것이다.


```java
// KStream과 KTable에서 동일한 메시지키를 가진 데이터를 찾았을 경우 각가의 메시지 값을 조합해서 어떤 데이터를 만들지 정의한다.
orderStream.join(addressTable, (order, address) -> order + " send to " + address).to(ORDER_JOIN_STREAM);
```
<img width="389" height="224" alt="스크린샷 2025-08-02 오후 11 15 41" src="https://github.com/user-attachments/assets/3d40dd9a-195f-455f-9bae-f7676408487f" />

#### 스트림즈DSL - GlovalTable과 KStream을 join
- 코파티셔닝되지 않은 데이털르 조인하는 두 가지 방법
  - 리파티셔닝
  - GlobalKTable로 선언하기
- GlobalKTable로 선언하는 방법을 알아보자.
  - 토폴로지
    <img width="309" height="253" alt="스크린샷 2025-08-02 오후 11 16 42" src="https://github.com/user-attachments/assets/0895c381-1b41-42a8-a440-5672fb062bab" />

```java
// GlobalKTable은 KTable의 조인과 다르게 레코드를 매칭할 떄 KStream의 메시지 키와 메시지 값 둘 다 사용할 수 있다. 여기서는 KStream의 메시지 키를 GlobalKTable의 메시지 키와 매칭하도록 설정했다.
orderStream.join(addressGlobalTable, (orderKey, orderValue) -> orderKey, (order, address) -> order + " send to " + address).to(ORDER_JOIN_STREAM);
```
- 결과물을 보면 KTable과 크게 달라보이지 않지만, GlobalKTable로 선언한 토픽은 토픽에 존재하는 모든 데이터를 태스크마다 저장하고 조인 처리를 수행하는 점이 다르다.
- 그리고 조인을 수행할 때 KStream의 메시지 키 뿐만 아니라 메시지 값을 기준으로도 매칭하여 조인할 수 있다는 점도 다르다.

### 3.5.2 프로세서 API
- 프로세서 API는 스트림즈DSL보다 투박한 코드를 가지지만 토폴로지를 기준으로 데이터를 처리한다는 관점에서 동일한 역할을 한다.
- 추가적인 상세 로직 구현이 필요하다면 프로세서 API를 활용할 수 있다. 프로세서 API에서는 스트림즈DSL에서 사용했던 KStream, KTable, GlobalKtable 개념이 없다는 점을 주의해야 한다.
```java
public class FilterProcessor implements Processor<String, String, String, String> {

    private ProcessorContext context;

    @Override
    public void init(ProcessorContext<String, String> context) {
        this.context = context;
    }

    @Override
    public void process(Record<String, String> record) { // 실질적으로 프로세싱 로직이 들어가는 부분
        if (record.value().length() > 5){ // 메시지값이 5 이상인 경우를 필터링하여 처리
            context.forward(record);
        }
        context.commit(); // 명시적으로 데이터가 처리되었음을 선언
    }

    @Override
    public void close() {
    }
}
```
```java
// 사용
public class SimpleKafkaProcessor {

    private final static String BOOTSTRAP_SERVERS = "localhost:9092";
    private final static String APPLICATION_NAME = "processor-application";
    private final static String STREAM_LOG = "test";
    private final static String STREAM_LOG_COPY = "stream_log_copy";

    public static void main(String[] args) {
        var props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG,
                Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,
                Serdes.String().getClass());

        var topology = new Topology();

        topology.addSource("Source", STREAM_LOG)
                .addProcessor("Process",
                        () -> new FilterProcessor(),
                        "Source")
                .addSink("Sink",
                        STREAM_LOG_COPY,
                        "Process");

        var streaming = new KafkaStreams(topology, props);
        streaming.start();
    }
}
```
- 스트림 프로세서를 사용하기 위해 addProcessor() 메서드를 사용
- addProcessor() 메서드의 첫 번째 파라미터는 스트림 프로세서의 이름을 입력한다.
- 두 번쨰 파라미터는 사용자가 정의한 프로세서 인스턴스를 입력한다.
- 세 번째 파라미터는 부모 노드를 입력해야 하는데 여기서 부모 노드는 "Source"이다.
- stream_log_filter를 싱크 프로세서로 사용하여 데이터를 저장하기 위해 addSink() 메서드를 사용했다. 첫 번째 파라미터는 싱크 프로세서의 이름을 입력한다. 두 번째 파라미터는 저장할 토픽의 이름을 입력한다. 세 번째는 부모 노드를 입력하는데 필터링 처리가 완료된 데이터를 저장해야 하므로 "Process"다.


## 3.6 카프카 커넥트
- 카프카 오픈소스에 포함된 둘 중 하나로 데이터 파이프라인 생성 시 반복 작업을 줄이고 효율적인 전송을 이루기 위한 애플리케이션
- 특정한 작업 형태를 템플릿으로 만들어놓은 커넥터를 실행함으로써 반복 작업을 줄일 수 있다.
- 커넥터는 **프로듀서** 역햘을 하는 `소스 커넥터`와 **컨슈머** 역할을 하는 `싱크 커넥터` 두 가지로 나뉜다.
- <img width="267" height="152" alt="스크린샷 2025-08-02 오후 11 25 22" src="https://github.com/user-attachments/assets/46ec665f-ca7c-4e7b-b5c0-45b44b5daae5" />
- 사용자가 커넥트에 커넥터 생성 명령을 내리면 커넥트는 내부에 커넥터와 태스크 생성
  <img width="324" height="159" alt="스크린샷 2025-08-03 오후 11 27 29" src="https://github.com/user-attachments/assets/ad97857e-a139-4f22-9d9e-f8fa13e4a613" />

- 사용자가 커넥터를 사용하여 파이프라인을 생성할 때 컨버터와 트랜스폼 기능을 옵션으로 추가할 수 있다.
- 컨버터는 데이터 처리를 하기 전에 스키마를 변경하도록 도와준다.
- 트랜스폼은 데이터 처리 시 각 메시지 단위로 데이터를 간단하게 변환하기 위한 용도로 사용된다.


### 커넥트를 실행하는 방법
- 커넥트를 실행하는 방법은 크게 두 가지가 있다.
  - 단일 모드 커넥트
  - 분산 모드 커넥트

#### 단일 모드 커넥트
- `connect-standalone.properties` 파일을 수정해야 함
- 해당 파일은 카프카 바이너리 디렉토리의 config 디렉토리에 있음
```properties
# 커넥트와 연동할 카프카 클러스터의 호스트와 포트 설정
# 2개 이상일땐 콤마로 구분하여 설정
bootstrap.servers=my-kafka:9092

# 데이터를 카프카에 저장할 때나 가져올 때 변환할 때 사용
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter

# 스키마 형태를 사용하고 싶지 않다면 enable 옵션을 false로 설정
key.converter.schemas.enable=false
value.converter.schemas.enable=false

# 태스크가 처리 완료한 오프셋을 커밋하는 주기를 설정
offset.storage.file.filename=/tmp/connect.offsets
offset.flush.interval.ms=10000

# 플러그인 형태로 추가할 커넥터의 디렉토리 주소를 입력
plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors,
#plugin.path=
```
- 단일 모드 커넥트를 실행 시에 파라미터로 커넥트 설정파일과 커넥터 설정파일을 차례로 넣어 실행한다.
```bash
$ bin/connect-standalone.sh config/connect-standalone.properties \
config/connect-file-source.properties
```

#### 분산 모드 커넥트
- 분산 모드 커넥트는 단일 모드 커넥트와 다르게 2개 이상의 프로세스가 1개의 그룹으로 묶여 운영된다.
- 1개의 커넥트 프로세스에 이슈 발생하여도 살아있는 나머지 1개 커넥트 프로세스가 이어받아 파이프라인을 지속적으로 실행할 수 있다.
- connect-distributed.properties 를 아래와 같이 설정한다.
```properties
bootstrap.servers=my-kafka:9092
# 다수의 커넥트 프로세스들을 묶을 그룹 이름 지정.
# 동일한 group.id로 지정된 커넥트들은 같은 그룹으로 인식한다.
# 같은 그룹으로 지정된 커넥트들에서 커넥터가 실행되면 커넥트들에 분산되어 실행된다.
# 한 대에 이슈가 발생하더라도 나머지 커넥트가 커넥터를 안전하게 실행할 수 있다.
group.id=connect-cluster

key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter

key.converter.schemas.enable=false
value.converter.schemas.enable=false

# 분산 모드 커넥트는 카프카 내부 토픽에 오프셋 정보를 저장한다.
# 오프셋 정보는 소스 커넥터 또는 싱크 커넥터가 데이터 처리 시점을 저장하기 위해 사용한다.
# 해당 정보는 데이터를 처리하는 데에 있어 중요한 역할을 하므로 실제로 운영할 때는 복제 개수를 3보다 큰 값으로 설정하는 것이 좋다.
offset.storage.topic=connect-offsets
offset.storage.replication.factor=1

config.storage.topic=connect-configs
config.storage.replication.factor=1

status.storage.topic=connect-status
status.storage.replication.factor=1

# 태스크가 처리 완료한 오프셋을 커밋하는 주기를 설정한다.
offset.flush.interval.ms=10000

plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors,
```
- 실행
  ```bash
  $ bin/connect-distributed.sh config/connect-distributed.properties
  ```

### 3.6.1 소스 커넥터
- 소스 커넥터는 소스 애플리케이션 또는 소스 파일로부터 데이터를 가져와 토픽으로 넣는 역할을 한다.
- 소스 커넥터를 만들 때는 connect-api 라이브러리를 추가해야 한다.
- 소스 커넥터를 만들 때 필요한 클래스는 2개다.
  - SourceConnector : 태스크를 실행하기 전 커넥터 설정파일을 초기화하고 어떤 태스크 클래스를 사용할 것인지 정의하는 데에 사용한다. 그렇기 때문에 SourceConnector에는 실질적인 데이터를 다루는 부분이 들어가지 않는다.
  - SourceTask : 실제로 데이터를 다루는 클래스로, 소스 애플리케이션 또는 소스 파일로부터 데이터를 가져와서 토픽으로 데이터를 보내는 역할을 수행한다.
<img width="211" height="143" alt="스크린샷 2025-08-03 오후 11 40 55" src="https://github.com/user-attachments/assets/d452b4b3-f0d8-4e5f-a359-d53fe6fa62c6" />

- 먼저, SourceConnector를 상속받은 사용자의 클래스를 생성해야 한다.
  ```java
  public class TestSourceConnector extends SourceConnector {
    
    // 사용자가 JSON 또는 config 파일 형태로 입력한 설정값을 초기화하는 메서드다.
    // 만약 올바른 값이 아니라면 여기서 ConnectException()을 호출하여 커넥터를 종료할 수 있다.
    // 예를 들어, JDBC 소스 커넥터라면 JDBC 커넥션 URL값을 검증하는 로직을 넣을 수 있다.
    // 만약 비정상적인 URL값이라면 커넥션을 맺을 필요 없이 커넥터를 종료시킬 수 있다.
    @Override
    public void start(Map<String, String> map) {

    }

    // 이 커넥터가 사용할 태스크 클래스를 지정한다.
    @Override
    public Class<? extends Task> taskClass() {
        return null;
    }

    // 태스크 개수가 2개 이상인 경우 태스크마다 각기 다른 옵션을 설정할 때 사용한다.
    @Override
    public List<Map<String, String>> taskConfigs(int i) {
        return List.of();
    }

    // 커넥터가 종료될 때 필요한 로직을 작성한다.
    @Override
    public void stop() {

    }

    // 커넥터가 사용할 설정값에 대한 정보를 받는다.
    // 커넥터의 설정값은 ConfigDef 클래스를 통해 각 설정의 이름, 기본값, 중요도, 설명을 정의할 수 있다.
    @Override
    public ConfigDef config() {
        return null;
    }

    // 커넥터의 버전을 리턴한다.
    // 커넥트에 포함된 커넥터 플러그인을 조회할 때 이 버전이 노출된다.
    // 커넥터를 지속적으로 유지보수하고 신규 배포할 때 이 메서드가 리턴하는 버전 값을 변경해야 한다.
    @Override
    public String version() {
        return "";
    }
  }
  ```
- SourceTask를 상속받은 사용자 클래스는 4개 메서드를 구현해야 한다.
  ```java
  public class TestSourceTask extends SourceTask {
    
    // 태스크의 버전을 저장한다.
    // 보통 커넥터의 version() 메서드에서 지정한 버전과 동일한 버전으로 작성하는 것이 일반적이다.
    @Override
    public String version() {
        return "";
    }

    // 태스크가 시작할 때 필요한 로직을 작성한다.
    // 태스크는 실질적으로 데이터를 처리하는 역할을 하므로 데이터 처리에 필요한 모든 리소스를 여기서 초기화하면 좋다.
    // 예를 들어, JDBC 소스 커넥터를 구현한다면 이 메서드에서 JDBC 커넥션을 맺는다.
    @Override
    public void start(Map<String, String> map) {
        
    }

    // 소스 애플리케이션 또는 소스 파일로부터 데이터를 읽어오는 로직을 작성한다.
    // 데이터를 읽어오면 토픽으로 보낼 데이터를 SourceRecord로 정의한다.
    // SourceRecord클래스는 토픽으로 데이터를 정의하기 위해 사용한다.
    // List<SourceRecord> 인스턴스에 데이터를 담아 리턴하면 데이터가 토픽으로 전송된다.
    @Override
    public List<SourceRecord> poll() throws InterruptedException {
        return List.of();
    }

    // 태스크가 종료될 때 필요한 로직을 작성한다.
    // JDBC 소스 커넥터를 구현했다면 이 메서드에서 JDBC 커넥션을 종료하는 로직을 추가하면 된다.
    @Override
    public void stop() {

    }
  }
  ```
### 3.6.2 싣크 커넥터
- 싱크 커넥터는 토픽의 데이터를 타깃 애플리케이션 또는 타깃 파일로 저장하는 역할을 한다.
- SinkConnector와 SinkTask 클래스를 사용하면 직접 싱크 커넥터를 구현할 수 있다.
  <img width="281" height="154" alt="스크린샷 2025-08-03 오후 11 41 12" src="https://github.com/user-attachments/assets/0d7aa071-5558-4bd0-b815-9a25694d1595" />
- 먼저, SinkConnector를 상속받은 사용자의 클래스를 생성해야 한다.
  ```java
  public class TestSinkConnector extends SinkConnector {
      
      // 사용자가 JSON 또는 config 파일 형태로 입력한 설정값을 초기화하는 메서드다.
      // 만약 올바른 값이 아니라면 여기서 ConnectException()을 호출하여 커넥터를 종료할 수 있다.
      @Override
      public void start(Map<String, String> map) {
  
      }
  
      // 이 커넥터가 사용할 태스크 클래스를 지정한다.
      @Override
      public Class<? extends Task> taskClass() {
          return null;
      }
  
      // 태스크 개수가 2개 이상인 경우 태스크마다 각기 다른 옵션을 설정할 때 사용한다.
      @Override
      public List<Map<String, String>> taskConfigs(int i) {
          return List.of();
      }
  
      // 커넥터가 종료될 때 필요한 로직을 작성한다.
      @Override
      public void stop() {
  
      }
  
      // 커넥터가 사용할 설정값에 대한 정보를 받는다.
      // 커넥터의 설정값은 ConfigDef 클래스를 통해 각 설정의 이름, 기본값, 중요도, 설명을 정의할 수 있다.
      @Override
      public ConfigDef config() {
          return null;
      }
  
      // 커넥터의 버전을 리턴한다.
      // 커넥트에 포함된 커넥터 플러그인을 조회할 때 이 버전이 노출된다.
      // 커넥터를 지속적으로 유지보수하고 신규 배포할 때 이 메서드가 리턴하는 버전 값을 변경해야 한다.
      @Override
      public String version() {
          return "";
      }
  }
  ```
- SinkTask를 상속받은 사용자 클래스는 5개 메서드를 구현해야 한다.
  ```java
  public class TestSinkTask extends SinkTask {
    
    // 태스크의 버전을 저장한다.
    // 보통 커넥터의 version() 메서드에서 지정한 버전과 동일한 버전으로 작성하는 것이 일반적이다.
    @Override
    public String version() {
        return "";
    }

    // 태스크가 시작할 때 필요한 로직을 작성한다.
    // 태스크는 실질적으로 데이터를 처리하는 역할을 하므로 데이터 처리에 필요한 모든 리소스를 여기서 초기화하면 좋다.
    // 예를 들어, mongoDB 싱크 커넥터를 구현한다면 이 메서드에서 mongoDB 커넥션을 맺는다.
    @Override
    public void start(Map<String, String> map) {
        
    }

    // 싱크 애플리케이션 또는 싱크 파일에 저장할 데이터를 토픽에서 주기적으로 가져오는 메서드이다.
    // 토픽의 데이터들은 여러 개의 SinkRecord를 묶어 파라미터로 사용할 수 있다.
    // SinkRecord는 토픽의 한 개 레코드이며 토픽, 파티션, 타임스탬프 등의 정보를 담고있다.
    @Override
    public void put(Collection<SinkRecord> records) {
    
    }
    
    // put() 메서드를 통해 가져온 데이터를 일정 주기로 싱크 애플리케이션 또는 싱크 파일에 저장할 때 사용하는 로직이다.
    // 예를 들어, JDBC 커넥션을 맺어서 MySQL에 데이터를 저장할 때 put() 메서드에서는 데이터를 insert하고 flush() 메서드에서는 commit을 수행하여 트랜잭션을 끝낼 수 있다.
    // put() 메서드에서 레코드를 저장하는 로직을 넣을 수도 있으며 이 경우에는 flush() 메서드에는 로직을 구현하지 않아도 된다.
    @Override
    public void flush(Map<TopicPartition, OffsetAndMetadata> offsets) {
    
    }

    // 태스크가 종료될 때 필요한 로직을 작성한다.
    // 태스크에서 사용한 리소스를 종료해야 할 때 여기에 종료 코드를 구현한다.
    @Override
    public void stop() {

    }
  }
  ```
## 3.7 카프카 미러메이커2
#### 미러메이커2를 활용한 단방향 토픽 복제
- 서로 다른 카프카 클러스터 간에 토픽을 복제하는 애플리케이션
- 원본 Kafka 클러스터와 복제할 Kafka 클러스터를 설정
- MirrorMaker 2의 source.cluster.alias와 target.cluster.alias를 각각 설정하여 원본 클러스터와 복제 클러스터를 지정
- replication.policy.class를 설정하여 토픽 이름의 변환 규칙을 설정할 수 있다. 예를 들어, org.apache.kafka.connect.mirror.DefaultReplicationPolicy를 사용하면 복제된 토픽의 이름이 원본 클러스터와 동일하게 유지된다.
- 원본 클러스터에서 발생하는 모든 데이터 변경 사항이 복제 클러스터로 즉시 전송된다.
- 복제 클러스터는 데이터 소비만 하며, 원본 클러스터로 데이터가 역으로 전송되지 않는다.

### 3.7.1 미러메이커2를 활용한 지리적 복제
- 지리적 복제는 물리적으로 분리된 여러 데이터 센터 간에 Kafka 데이터를 복제하는 전략이다. 이는 재해 복구, 지연 감소 및 지역별 데이터 소비 요구를 충족하기 위해 필수적이다.
- 각 지리적 위치에 Kafka 클러스터를 설치하고, MirrorMaker 2를 사용하여 클러스터 간 데이터를 복제
- 각 지리적 클러스터에 대한 연결 정보를 제공하고, 토픽 및 소비자 그룹을 포함한 메타데이터를 동기화
- MirrorMaker 2에서 active.active.replication 설정을 통해 양방향 복제 구성도 가능하지만, 지리적 복제에서는 주로 단방향 복제를 많이 사용

- 데이터가 각 지역에 분산되어 있어 특정 지역의 클러스터가 다운되더라도 다른 지역의 클러스터가 데이터 가용성을 유지할 수 있다.
- 네트워크 지연이 중요한 요소이며, 이를 최소화하기 위한 네트워크 구성 및 데이터 압축 옵션을 고려해야 한다.


#### 액티브-스탠바이 클러스터 운영
액티브-스탠바이 클러스터 운영은 한 클러스터가 주요 작업을 처리하고, 다른 클러스터는 백업 역할을 하는 구조다. 주로 재해 복구 및 고가용성 보장을 위해 사용한다.
<img width="276" height="108" alt="스크린샷 2025-08-03 오후 11 45 53" src="https://github.com/user-attachments/assets/6f7eab9e-7564-43ef-a793-172f2818a55f" />
<img width="339" height="113" alt="스크린샷 2025-08-03 오후 11 46 00" src="https://github.com/user-attachments/assets/4df1e0dc-2570-4b2a-a722-f79c51b5c541" />


#### 액티브-액티브 클러스터 운영

액티브-액티브 클러스터 운영은 두 개 이상의 클러스터가 동시에 데이터를 처리하고 상호 복제를 통해 일관성을 유지하는 구조다. 이는 고가용성과 데이터 일관성, 그리고 더 나은 성능을 제공하기 위해 사용된다.
<img width="308" height="99" alt="스크린샷 2025-08-03 오후 11 46 20" src="https://github.com/user-attachments/assets/a63e81eb-2c45-49da-a798-dd49a26d2ec2" />



#### 허브 앤 스포크 클러스터 운영
- 허브 앤 스포크 클러스터 운영은 중앙 허브 클러스터가 여러 스포크 클러스터와 데이터를 교환하는 구조다. 이는 중앙 집중형 데이터 수집 및 배포에 유용하다.
<img width="306" height="130" alt="스크린샷 2025-08-03 오후 11 46 27" src="https://github.com/user-attachments/assets/abf2ab6b-8e79-4855-a263-ab71f47de79a" />



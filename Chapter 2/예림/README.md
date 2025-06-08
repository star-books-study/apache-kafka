# Chatper 02. 카프카 빠르게 시작해보기

## 2.1 실습용 카프카 브로커 설치
- Amazon Linux2 AMI는 Amazon에서 직접 만든 템플릿으로서 EC2 성능에 최적화되어 있다.
- 카프카의 기본 포트는 9092이고 주키퍼의 기본 포트는 2181이다. -> 보안그룹에서 열어주기
### 자바 설치
  ```bash
  $ sudo ym install -y java-1.8.0-openjdk-devel.x86_64
  ```

### 주키퍼 & 카프카 브로커 실행
- **카프카 브로커를 실행하기 위해 카프카 바이너리 패키지를 다운로드**
  - 카프카 파이너리 패키지 : 자바 소스코드를 컴파일하여 실행하기 위해 준비해놓은 파이너리 파일들이 들어있음
  - 카프카 공식 홈페이지에서 다운 가능
  - 두 가지 버전(2.12, 2.13)이 있는데 둘 다 실습 환경에서 실행시 차이가 없음
  ```bash
  $ wget https://archive.apache.org/dist/kafka/2.5.0/kafka_2.12-2.5.0.tgz
  $ tar xvf kafka_2.12-2.5.0. tgz
  kafka 2.12-2.5.0/
  kafka_2.12-2.5.0/LICENSE
  $ ll
  $ cd kafka_2.12-2.5.0
  ```
- **카프카 힙 메모리 설정**
  - 카프카 브로커를 실행하기 위해서는 힙 메모리 설정이 필요
  - 카프카 브로커는 레코드의 내용은 페이지 캐시로 시스템 메모리를 사용하고 나머지 객체들을 힙 메모리에 저장하여 사용한다는 특징이 있다.
  - 카프카 패키지의 힙 메모리는 카프카 브로커는 1G, 주키퍼는 512MB로 기본 설정되어 있다.
  - 실습용 인스턴스는 1G 메모리를 갖고 있으므로 카프카 브로커와 주키퍼를 동시에 실행하면 실행 불가
  ➡️ 다음 명령어로 힙 메모리 사이즈 환경변수로 지정 (다음 내용을` ~/.bashrc` 파일에 넣으면 된다)
  ```bash
  # User specific aliases and functions
  $ export KAFKA_HEAP_OPTS="-Xmx400m -Xms400m"
  $ echo $KAFKA_HEAP_OPS
  -Xmx400m -Xms400m
  ```
  - 카프카 브로커 실행 시 메모리를 설정하는 부분은 카프카를 실행하기 위해서 사용하는 `kafka-server-start.sh` 스크립트 내부에서 확인 가능
- **카프카 브로커 실행 옵션 설정**
  ```bash
  $ vi config/server.properties
  ```
- **주키퍼 실행**
  - 카프카 바이너리가 포함된 폴더에는 브로커와 같이 실행할 주키퍼가 준비되어 있다.
  - 주키퍼는 카프카의 클러스터 설정 리더 정보, 컨트롤러 정보를 담고 있어 카프카를 실행하는 데에 필요한 필수 애플리케이션이다.
  - 주키퍼를 상용 환경에서 안전하게 운영하기 위해서는 3대 이상의 서버로 구성하여 사용하지만 실습에서는 동일한 서버에 카프카와 동시에 1대만 실행시켜 사용
    - 1대만 실행하는 주키퍼를 'Quick-and-dirty single-node'라고 부른다.
    - 테스트 목적으로만 해야 한다.
  - 시작 스크립트 실행 (백그라운드)
    ```bash
    $ bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
    ```
  - 실행 확인
    ```bash
    jps -vm
    ```
    - -m 옵션 : main 메서드에 전달된 인자 확인
    - -v 옵션 : JVM에 전달된 인자(힙 메모리 설정, log4j 설정 등) 확인
  - **카프카 브로커 실행 및 로그 확인**
    ```bash
    $ bin/kafka-server-start.sh -daemon config/server.properties
    $ jps -m
    ```
### 로컬 컴퓨터에서 카프카와 통신 확인
![스크린샷 2025-06-08 오후 11 32 29](https://github.com/user-attachments/assets/12e20408-6208-4361-b9a7-332a038429ae)

- 카프카 바이너리 패키지는 카프카 브로커에 대한 정보를 가져올 수 있는 kafka-broker-api-version.sh 명령어를 제공함
  - 이를 사용하기 위해서는 로컬 컴퓨터에 카프카 바이너리 패키지를 다운로드 해야 함
    ```bash
    $ curl https://archive.apache.org/dist/kafka/2.5.0/kafka_2.12-2.5.0.tgz
    $ tar -xvf kafka.tgz
    $ cd kafka_2.12-2.5.0
    $ ls
    $ ls bin
    $ bin/kafka-broker-api-versions.sh --bootstrap-server 13.124.99.218:9092
    ```
  - 테스트 편의를 위한 hosts 설정
    ```bash
    $ vi /etc/hosts
    ```
## 2.2 카프카 커멘드 라인툴 

### kafka-topics.sh
- 이 커맨드 라인 툴을 통해 토픽과 관련도니 명령을 수행할 수 있다.
- 토픽 : 카프카에서 데이터를 구분하는 가장 기본적인 개념 (like 테이블)
- 토픽에는 파티션이 존재하는데 파티션의 개수는 최소 1개부터 시작
  - 파티션을 통해 한 번에 처리할 수 있는 데이터양을 늘릴 수 있고 토픽 내부에서도 파티션을 통해 데이터의 종류를 나누어 처리 가능

#### 토픽 생성
```bash
# 로컬
$ bin/kafka-topics.sh \
--create
-bootstrap-server my-kafka: 9092 \
--topic hello.kafka \
```
#### 토픽 상세 조회
```bash
bin/kafka-topics.sh --bootstrap-server my-kafka:9092 --describe --topic hello.
kafka.2
```

#### 토픽 옵션 수정
```bash
$ bin/kafka-topics.sh --bootstrap-server my-kafka: 9092 \
--topic hello.kafka
--alter \
--partitions 4
$ bin/kafka-topics.sh --bootstrap-server my-kafka: 9092 \

$ bin/kafka-configs.sh --bootstrap-server my-kafka:9092 \
--entity-type topics \
--entity-name hello. kafka \
--alter --add-config retention.ms=86400000

```

### kafka-console-producer.sh
- 생성된 hello.kafka 토픽에 넣을 수 있는 kafka-console-producer.sh 명령어를 실행해보자.
- 토픽에 넣는 데이터는 레코드라고 부르며 메시지 키와 메시지 값으로 이루어져 있다.
- 이번에는 값만 보내도록 하자.
```bash
$ bin/kafka-console-producer.sh --bootstrap-server my-kafka:9092 \ --topic hello.kafka
>hello
>kafka
>0
>1
>2
>3
>4
>5
```
![image](https://github.com/user-attachments/assets/56538ac6-7a1a-4539-a85a-20f4d91449dd)

- 이제는 메시지 키를 가지는 메시지를 전송해보자.

```bash
$ bin/kafka-console-producer.sh --bootstrap-server my-kafka:9092 \
--topic hello.kafka \
--property "parse.keystrue" \
--property "key .separators:"
> key1:no1
> key2:no2
> key3:поз
```
![스크린샷 2025-06-08 오후 11 42 21](https://github.com/user-attachments/assets/7beb1e8e-820c-460c-b845-0e8d7dd53699)

- 메시지 키가 null인 경우에는 프로듀서가 파티션으로 전송할 때 레코드 배치 단위로 라운드 로빈으로 전송된다.
- 메시지 키가 존재하는 경우에는 키의 해시값을 작성하여 존재하는 파티션 중 한 개에 할당된다.
  - 이로 인해 메시지 키가 동일한 경우에는 동일한 파티션으로 전송된다.

### kafka-console-consumer.sh
- hello.kafka 토픽으로 전송한 데이터는 kafka-console-consumer.sh 명령어로 확인할 수 있다.
  - 필수 옵션 --bootstrap-server과 파크파 클러스터 정보, --topic에 토픽 이름이 필요
```bash
$ bin/kafka-console-consumer.sh --bootstrap-server my-kafka:9092 \
•-topic hello.kafka \
-from-beginning
```
- 데이터 메시지 키와 메시지 값을 확인하고 싶다면 --property 옵션을 사용하면 됨
![스크린샷 2025-06-08 오후 11 45 14](https://github.com/user-attachments/assets/1822e238-fc7b-4fc8-a746-ca97a2a57f49)


#### kafka-consumer-groups.sh
- hello-group 이름의 그룹으로 생성된 컨슈머로 hello.kafka 토픽의 데이터를 가져갔다.
- 컨슈머 그룹은 따로 생성하는 명령을 날리지 않고 컨슈머를 동작할 때 컨슈머 그룹 이름을 지정하면 새로 생성된다.
- 생성된 컨슈머 그룹의 리스트는 kafka-consumer-groups.sh 명령어로 확인할 수 있다.

```bash
$ bin/kafka-consumer-groups.sh --bootstrap-server my-kafka:9092 --list hello-group
```

```bash
$ bin/kafka-consumer-group.sh --bootstrap-server my-kafka:9092 \
  --group hello-group \
  --describe
```

### kafak-verifiable-producer, consumer.sh
- katka-verifiable로 시작하는 2개의 스크립트를 사용하면 String 타입 메시지 값을 코드 없이 주고받을 수 있다. 카프카 클러스터 설치가 완료된 이후에 토픽에 데이터를 전송하여 간단한 네트워크 통신 테스트를 할 때 유용하다.
```bash
§ bin/kafka-verifiable-producer.sh--bootstrap-servermy-kafka:9092\
-max-messages 10 \
--topic verify-test
```
전송한 데이터는 kafka-verifiable-consumer.sh로 확인 가능
```bash
§ bin/kafka-verifiable-consumer.sh--bootstrap-servermy-kafka:9092\
--topic verify-test \ =-group-id test-group
```
### kafka-delete-records.sh
- 이미 적재된 토픽의 데이터 중 가장 오래된 데이터부터 특정 시점의 오프셋까지 삭제 가능


- 예를 들어 test 토픽의 0번 파티션에 0부터 100까지 데이터가 들어있을 때 0부터 30 오프셋 데이터까지 지우고 싶다면 다음과 같이 입력
  ```bash
  $ vi delete-topic.json
  $ bin/kafka-delete-records.sh --bootstrap-server my-kafka:9092 \
  --offset-json-file delete-topic.json
  ```
## 2.3 정리
- 실습용 카프카 설치하고 관련 명령어 실행하며 토픽 생성, 수정하고 데이터를 전송(프로듀서)하고 받는(컨슈머) 실습을 진행해봄

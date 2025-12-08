# 6. 클라우드 카프카 서비스
- 운영상 시행착오를 줄이면서 최고의 카프카 클러스터를 빠르게 설치하여 안전하게 운영하기 위해 SaaS를 도입할 수 있다.
<img width="509" height="370" alt="스크린샷 2025-11-06 오후 5 04 25" src="https://github.com/user-attachments/assets/a4fb4a82-3c56-47ad-be14-75f826305d1e" />


- 카프카를 SaaS로 제공하는 업체에는 AWS와 Confluent가 있다.
  - AWS : MSK(Managed Streaming for Apache Kafka)
  - Confluent : Kafka Cloud

### 인프라 관리의 이점
- SaaS를 사용할 경우 인프라 운영 관련 역할에서 자유로울 수 있다.
  - 카프카 SaaS 서비스를 사용하게 되면 브로커가 올라가는 서버는 자동으로 관리되기 때문이다.
- SaaS 서비스가 이슈를 감지하여 이슈가 생긴 서버를 제외하고 신규 장비에 브로커를 실행하여 클러스터를 복구
- 클러스터의 데이터 사용량이 순간적으로 많아지더라도 서비스를 제공하는 업체의 SaaS 대시보드에서 브로커 개수만 설정하여 쉽게 스케일 아웃할 수 있다.

### 모니터링 대시보드 제공
- 브로커들이 제공하는 지표들을 수집하고 적재하고 대시보드화하여 **데이터를 시각화**해야 카프카 클러스터를 효과적으로 운영하기 위해 필요한 설정들을 수정하고 적용할 수 있다
- SaaS형 카프카에서는 자동화되어 만들어진 클러스터로부터 운영에 필요한 지표들을 수집하고 그래프로 보여주는 옵션이 제공된다. 만약 직접 카프카 클러스터를 운영했다면 수집한 지표를 저장할 저장소를 구축하고 대시보드를 운영하기 위해 신규로 추가 플랫폼을 설치, 운영해야 한다.

### 보안 설정
- 카프카 브로커는 SSL, SASL, ACL과 같이 불특정 다수의 침입을 막기 위해 다양한 종류의 보안 설정 방안을 제공하고 있다.
- 보안 설정 옵션을 고르는 일이 쉽지 않으므로 SaaS형 카프카에서는 클러스터 접속 시 보안 설정을 기본으로 제공하고  있다.

### 서비스 사용 비용
- 직접 서버를 발급하여 설치하고 운영한는 것이 AWS MSK를 사용하는 것에 비해 2배 더 저렴하다.

### 커스터마이징의 제한
- 인프라부터 애플리케이션 설치까지 모든 부분이 자동화되어 있어 커스터마이징이 어렵다. 특히 멀티 클라우드나 하이브라우드 클라우드 형태로 카프카 클러스터를 구성하는 것은 SaaS형 카프카에서는 불가능하다.


### 클라우드 종속성
- SaaS형 카프카를 구축하기 위해서는 클라우드 서비스 업체를 선택하고 클러스터를 운영해야 한다.
- 그러나 모든 애플리케이션이 해당 서비스에 종속될 수 있다는 걸 유의해야 한다.


## 6.1 컨플루언트 클라우드
- 컨플루언트(Confluent)는 카프카에 대한 개념을 최초로 생각하고 아키텍처를 제안, 개발한 인물인 제이 크랩스와 그의 동료들이 설립한 회사이다.
- 컨플루언트는 아파치 카프카의 프리미엄 비전을 고객들에게 제공하고 있다. 컨플루언트에서 제공하는 카프카는 기존 아파치 카프카 오픈소스 버전을 자체 기준에 맞게 커스터마이징하고 운영에 필요한 각종 기능을 추가하여 제공한다.
- 외부 인터넷과 연동이 되지 않는 폐쇄망에서 컨플루언트의 커스텀 카프카를 사용하는 것을 고려하여 설치형 카프카를 제공하고 있다.
- 컨플루언트에서 제공하는 설치형 카프카는 '컨플루언트 플랫폼'이라는 이름으로 제공하며 1대의 브로커에 설치할 경우 해당 기능들을 무료로 사용해볼 수 있다.
- 외부 인터넷과 연동되는 환경에서는 '컨플루언트 클라우드'라고 불리는 클라우드 기반 컨플루언트 카프카를 사용할 수 있다.
<img width="572" height="558" alt="스크린샷 2025-11-14 오후 11 26 14" src="https://github.com/user-attachments/assets/afbff42c-dd67-4cc3-833f-52a63b989308" />

### 6.1.1 컨플루언트 클라우드 활용
컨플루언트 클라우드는 직접 클러스터를 관리하지 않고 컨플루언트에 위임함으로써 카프카 클러스터를 안전하고 빠르게 구축, 운영할 수 있다.
### 6.1.1.1 클러스터 생성
<img width="535" height="320" alt="스크린샷 2025-11-15 오후 11 36 01" src="https://github.com/user-attachments/assets/fa1ad61c-27f9-4c2c-a0e7-3aae54635958" />

#### 6.1.1.2 토픽 생성
- 새로운 토픽을 만들기 위해서 토픽 이름과 파티션만 있으면 된다.

#### 6.1.1.3 API 키 발급
- 생성된 토픽을 연동하기 위해서는 API 키가 필요하다.
- 컨플루언트 쿨라우드를 통해 생성한 카프카 클러스터는 SASL_SSL 보안 설정이 기본으로 설정된다.
- SASL_SSL 보안 인증 방식은 SSL 통신을 통한 인증으로 탈취 위협으로부터 가장 안전한 방식이다.

#### 6.1.1.4 프로듀서 애플리케이션 연동
- 기존 방식처럼 프로듀서를 만들기 위한 kafka-client 라이브러리를 build.gradle에 추가한다. 추가로 카프카 클라이언트 로그를 확인하기 위해 slf4j-simple 라이브러리도 추가한다.
- 컨플루언트 클라우드를 통해 생성된 카프카 클러스터의 비전은 컨플루언트 플랫폼에서 제공하는 가장 최신 버전으로 항상 업데이트 된다.

```java
public class ConfluentCloudProducer {
  ...

  private final static String SECURITY_PROTOCOL = "SASL_SSL"; // SSAL 보안 접속을 하기 위해 보안 프로토콜들을 선언한다.
  private final static String JAAS_CONFIG = "org.apache.kafka.common.security.plain.PlainLoginModule  required username=\"2MA2CQM3Y6GVATMX\" password=\"생략"\;
  private final static String SSL_ENDPOOINT = "https";
  private final static String SASL_MECHANISM = "PLAIN";

  public static void main(String[] args) {
    Properties configs = new Properties();
    configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
    ...

    KafkaProducer<String, String> producer = new KafkaProducer<>(configs);
    String messageKey = "hellokafka"; 
    String messageValue = "helloConfluentCloud";
    ProduceRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, messageKey, messageValue);
    try {
      RecordMetadata metadata = producer.send(record).get(); // 프로듀서가 전송한 데이터가 정상적으로 브로커에 적재되었는지 여부를 확인하기 위해 RecordMetadata로 리턴받는다.
      logger.info(metadata.toString());
    } catch (Exception e) {
      logger.error(e.getMessage(), e);
    } finally {
      producer.flush();
      producer.close(); // 프로듀서에 존재하는 배치를 모두 전송한 이후에 안전하게 종료한다.
    }
  }
}
```
#### 6.1.1.5 컨슈머 애플리케이션 연동
- 컨슈머도 프로듀서와 동일하게 보안 설정
<img width="445" height="475" alt="스크린샷 2025-11-23 오후 8 55 19" src="https://github.com/user-attachments/assets/49e8456c-2b60-4935-ba90-bd17798e6719" />
<img width="446" height="299" alt="스크린샷 2025-11-23 오후 8 55 30" src="https://github.com/user-attachments/assets/a4c81ab5-8103-4c8c-ad82-7936f60942e5" />

- ConfluentCloudConsumer.java를 실행하면 컨슈머 애플리케이션이 동작하면서 test.log 토픽의 데이털르 가져올 것이다. 만약 보안 설정이 정상적으로 맺어지고 데이터를 가져온다면 컨슈머 로그에 프로듀서가 보낸 레코드의 메시지 키와 메시지 값이 출력될 것이다.
- Consumers 탭을 활용하면 편리하게 랙을 확인할 수 있다.
- 컨플루언트 클라우드에서는 클러스터 운영에 획기적으로 도움을 주는 데이터 흐름 시각화를 제공한다.

#### 6.1.1.6 커넥터 S3 적재 파이프라인
- 다만, 컨플루언트 클라우드에서 제공하는 커넥터는 컨플루언트 클라우드의 클러스터를 기준으로 연동할 수 있으며 온프레미스나 다른 퍼블릭 클라우드에서 생성한 클러스터는 컨플루언트 클라우드의 커넥터와 연동할 수 없다.
- S3를 생성하여 파일을 저장한다.
- S3에 적재할 데이터는 1개 이상의 토픽을 선택할 수 있다.
## 6.2 AWS MSK
- MSK : AWS에서 제공하는 SaaS형 아파치 카프카 서비스
- MSK로 생성한 클러스터는 AWS에서 제공하는 인프라 영역에 구축된다. MSK는 AWS에서 운영하는 애플리케이션과 쉽게 연동할 수 있기 때문에 AWS를 이미 사용 중인 기업에서는 어렵지 않게 아키텍처에 포함시킬 수 있다.
- 컨플루언트 클라우드와 다르게 카프카 브로커 로그를 AWS S3에 적재하여 브로커 로그를 확인할 수 있다.
- 클러스터를 모니터링하기 위해 따로 저장소나 수집 애플리케이션을 연동하지 않더라도 기본 모니터링 데이터를 AWS CloudWatch로 확인할 수 있도록 제공한다.
- 클라우드와치에서 클라이언트의 지표와 브로커의 지표를 함께 모니터링할 수 있기 때문에 효율은 더욱 늘어난다.
- 프로메테우스와 연동할 수 있는 JMX 익스포터, 노드 익스포터도 제공

### 6.2.1 MSK 활용
- 클러스터를 생성하고 AWS 인프라에서 클러스터가 어떤 아키텍처로 구축되는지 상세히 알아본다.
  - 카프카 클러스터 생성
  - 토픽 생성
  - 모니터링을 위한 프로메테우스, 그라파나 연동
  - 콘솔 프로듀서, 컨슈머 연동
#### 6.2.1.1 클러스터 생성
- 3개 AZ에 3개 브로커로 운영하는 것이 2개 AZ에 4개 브로커로 클러스터를 운영하는 것보다 더 안전하게 운영할 수 있다.
  <img width="786" height="486" alt="image" src="https://github.com/user-attachments/assets/7d933323-3506-46f0-85ac-edd4c0bf2d48" />

- 3개 AZ에 3개 브로커로 운영하는 것이 안전
<img width="415" height="296" alt="스크린샷 2025-12-07 오후 10 42 29" src="https://github.com/user-attachments/assets/4e7bac2f-725f-4199-b7b9-9f2200558911" />

#### 6.2.1.2 MSK 클러스터 연동 인스턴스 생성
- 만약 vpc 외부 네트워크에서 MSK 크러스터에 연결하기 위해서는 VPN, VPC 피어링, VPC 전송 케이트웨이, AWs Direct Connect를 사용해야만 한다.
- 클러스터로 접근하는 가장 쉬운 방법은 VPC 내부에 MSK 클러스터 연동을 위한 인스턴스를 생성하는 것이다.

<img width="312" height="274" alt="스크린샷 2025-12-07 오후 10 46 53" src="https://github.com/user-attachments/assets/76a7c0f1-e5ab-424c-b7bf-578935cd24aa" />
- 실행한 모니터링 도구름 확인하기 위해 퍼블럭 서브넷을 생성하고 퍼블릭 서브넷에 신규 인스턴스가 위치하도록 설정 한다.
- 이번에 설치할 모니터링 도구는 프로메테우스와 그라파나
  - 프로메테우스는 JMX 익스 포터와 노드 익스포터로부터 지표를 수집하고 저장한다. 그라파나는 프로메테우스에 저장된 데이터를 시각화하여 그래프로 조회할 수 있다.
#### 6.2.1.3 토픽 생성
- 생성한 인스턴스로 접속해 카프카 바이너리 파일을 다운 받고 실행
- 토픽 생성 명령 `kafka-topics.sh --create`
#### 6.2.1.4 프로메테우스 설치 및 연동
- 프로메테우스가 설치된 인스턴스의 퍼블릭 IP 그리고 9090 포트를 사용하여 웹 브라우저를 통해 접속
- 프로메테우스 웹 브라우저는 현재 모니터링하고 있는 익스포터를 확인하고 익스포터를 통해 수집한 데이터를 PromQL로 조회할 수 있다.
- 대시보드는 그라파타를 통해 구성
- 프로메테우스가 수집한 데이터는 <지표. 이름><레이블 키>=<레이블 값기 형태로 이루어 진다. 지표 이름은 데이터의 종류를 뜻한다. 레이블 키는 데이터의 특징, 레이블 값은 해당 데이터에 대한 값을 뜻한다. 지표와 레이블로 이루어진 값은 PromOL 을 동해 원하는 값을 뽑아낼 수 있다.

#### 6.2.1.5 그라파나 설치 및 연동
- 웹브라우저에서 인스턴스의 퍼블릭 IP를 호스트로 하고 그라파나의 기본 포트인 3000번 포트로 접속
- 그라파나는 웹 대시보드 애플리케이션으로서 자체적으로 지표를 저장하거나 수집하는 기능은 가지고 있지 않음
  - 다양한 데이터 소스들(mysql, elasticsearch, influxdh, prometheus 등) 을 그라파나로 연결 설정하고 쿼리를 통해 그래프들을 그리는 기능이 핵심
  - 이미 우리는 MSK 클러스터의 지표들을 프로메테우스로 수집, 저장했다.
  - 그라파나의 데이터 소스로 프로 메테우스를 설정하면 MSK 클러스터의 지표들을 그래프로 그릴 수 있다.
- 인스턴스에 띄운 프로메테우스와 연동하기 위해 그라파나에서는 localhost의 9090 포트(프로메테우스 통신 포트)로 설정한다.
- 만약 프로메테우스를 다른 인스턴스에서 실행했다면 그라 파나가 통신할 수 있도록 해당 인스턴스의 사설IP 또는 공인IP를 입력하여 데이터 소스를 연결할 수 있다.
- 그라파나에서 MSK 클러스터를 운영하기 위해 필요한 그래프는 매우 다양하다.
  - 대표적인 모니터링 지표 : 파티션 개수, 토픽 개수, 브로커빌 스토리지 남은 용량, 메모리 등
- 대시보드에 그리는 그래프
  • 브로커별 CPU 사용량
  • 브로커별 메모리 사용량
  • 토픽별 클러스터로 들어오는 메시지 개수
  • 토픽별 클러스터에서 나가는 메시지 개수
#### 6.2.1.6 콘솔 프로듀서, 컨슈머 연동
- MSK 클러스터와 통신하려면 SSL 인증 필수

  - 클라이언트가 MSK(TLS 활성화)와 통신하려면 truststore 필요
  
  - JDK에 기본 제공되는 cacerts 파일을 truststore로 사용 가능

  - /jre/lib/security/cacerts 를 복사해서
  kafka.client.truststore.jks 이름으로 저장한다.

- 클라이언트 설정 파일(client.properties) 생성
```
security.protocol=SSL
ssl.truststore.location=./kafka.client.truststore.jks
```


- 이 파일을 producer와 consumer 실행할 때 --producer.config, --consumer.config 옵션으로 사용.

- 콘솔 프로듀서 실행
```
kafka-console-producer.sh \
  --broker-list <MSK broker list> \
  --producer.config client.properties \
  --topic test.log
```

- 메시지 입력하면 MSK로 전송됨.

- 콘솔 컨슈머 실행
```
kafka-console-consumer.sh \
  --bootstrap-server <MSK broker list> \
  --consumer.config client.properties \
  --topic test.log \
  --from-beginning
```

- 프로듀서가 보낸 메시지를 정상적으로 읽어올 수 있음.

- MSK 모니터링

  - CloudWatch 또는 Prometheus로 카프카 입출력량 확인 가능.

  - 장애 브로커 자동 교체 등 운영 편의성이 뛰어남.

- MSK 장단점 정리
  - 장점

  AWS 환경에 자연스럽게 통합 → 운영 난도 매우 낮음.

  CloudWatch 기반의 강력한 모니터링.

- 브로커 장애 시 자동 복구

  - 카프카의 대부분 옵션 그대로 사용 가능 → 사실상 serverless 느낌.

  - 운영자는 인프라 대신 카프카 활용(비즈니스 로직) 에 집중 가능.

- 단점

  - MSK가 제공하는 인스턴스 타입만 사용 가능
  
  - Reserved Instance 미지원 → 직접 EC2에 구성할 때보다 다소 비쌈

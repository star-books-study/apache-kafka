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


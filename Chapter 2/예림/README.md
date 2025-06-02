# Chatper 02. 카프카 빠르게 시작해보기

## 2.1 실습용 카프카 브로커 설치
- Amazon Linux2 AMI는 Amazon에서 직접 만든 템플릿으로서 EC2 성능에 최적화되어 있다.
- 카프카의 기본 포트는 9092이고 주키퍼의 기본 포트는 2181이다. -> 보안그룹에서 열어주기
- 자바 설치
  ```bash
  sudo ym install -y java-1.8.0-openjdk-devel.x86_64
  ```

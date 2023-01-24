# Chapter8 - 카프카 버전 업그레이드와 확장

## 목차

1. 카프카 버전 업그레이드를 위한 준비
2. 주키퍼 의존성이 있는 카프카 롤링 업그레이드
3. 카프카의 확장

카프카 업그레이드 배경

- 오래된 버전으로 운영하면 원하는 기능을 사용할 수 없거나, 치명적인 버그가 발견되어 지속적인 문제가 발생할 수 있다.

---

## 카프카 버전 업그레이드를 위한 준비

### 현재 버전 확인

버전 확인하는 두 가지 방법

1. 명령어를 이용

```bash
$ /usr/local/kafka/bin/kafka-topics.sh --version
2.6.0 (Commit:62abe01bee039651)
```

1. 설치 경로의 jar 파일들을 확인

```bash
$ cd /usr/local/kafka/libs/
$ ls kafka_*
kafka_2.12-2.6.0-javadoc.jar
kafka_2.12-2.6.0-javadoc.jar.asc
kafka_2.12-2.6.0-scaladoc.jar
kafka_2.12-2.6.0-scaladoc.jar.asc
kafka_2.12-2.6.0-sources.jar
kafka_2.12-2.6.0-sources.jar.asc
kafka_2.12-2.6.0-test-sources.jar
kafka_2.12-2.6.0-test-sources.jar.asc
kafka_2.12-2.6.0-test.jar
kafka_2.12-2.6.0-test.jar.asc
kafka_2.12-2.6.0.jar
kafka_2.12-2.6.0.jar.asc
```

### 업그레이드 하려는 버전 확인

버전을 확인 했으면 업그레이드 하려는 버전을 정하고, 릴리즈 노트 등을 살펴보고, 업그레이드 시 문제가 될 만한 부분은 없는 지 확인

- 카프카 브로커의 상위 버전은 클라이언트들의 하위 호환성을 갖고 있어서 대부분 클라이언트 이슈는 없음
    - 가끔 있음: 스칼라 프로듀서, 컨슈머는 서비스 종료
- 메이저 버전 업그레이드는 스펙의 변화가 크게 일어날 수 있으므로, 꼼꼼하게 체크해야 된다.
    - 포맷 변경, 브로커 기본값 변화, 명령어 지원 종료, 일부 JMX 메트릭 변화 등
- 마이너 버전 업그레이드는 비교적 용이하다.

### 업그레이드 시 간단한 주의 사항

- 업그레이드시 카프카가 다운타임을 가져도 된다면, 모두 종료하고 최신 버전으로 업그레이드 하면 된다.
- 하지만 대부분 다운타임을 갖기 어려우므로, 브로커를 한 대씩 롤링 업그레이드를 해야한다.

---

## 주키퍼 의존성이 있는 카프카 롤링 업그레이드

목표: 2.1 → 2.6 업그레이드

### 카프카 2.1 설치 및 동작 확인

```bash
# 카프카 2.1 설치
$ cd chapter2/ansible_playbook
$ ansible-playbook -i hosts kafka2.1.yml

## 2.1 버전 옵션으로 수행 (--bootstrap-server 옵션 없음)

# 토픽 생성 (--zookeeper 옵션 사용)
$ /usr/local/kafka/bin/kafka-topics.sh --zookeeper peter-zk01.foo.bar --create --topic peter-version2-1 --partitions 1 --replication-factor 3
Created topic "peter-version2-1".

# 프로듀서 메시지 전송 (--broker-list 옵션 사용)
$ /usr/local/kafka/bin/kafka-topics.sh --broker-list peter-kafka01.foo.bak:9092 --topic peter-version2-1
>version2-1-message1
>version2-1-message2
>version2-1-message3

# 컨슈머 메시지 읽기
$ /usr/local/kafka/bin/kafka-topics.sh --bootstrap-server peter-kafka01.foo.bar:9092 --topic peter-version2-1 --from-beginning --group peter-consumer
version2-1-message1
version2-1-message2
version2-1-message3
```

### 카프카 2.6 버전 다운로드와 설정

```bash
# 2.6 다운로드는 생략

# kafka 심볼릭 링크 확인
$ cd /usr/local/
$ ll
lrwxrwxrwx 1 root root   27  1월 01 00:08 kafka -> /usr/local/kafka_2.12-2.1.0

# 2.1 설정 파일을 2.6 디렉토리로 복사 (2.1 버전 설정 유지를 위해)
$ sudo cp kafka_2.12-2.1.0/config/server.properties kafka_2.12-2.6.0/config/server.properties

# 2.6 버전 설정 파일에 설정 추가
$ sudo vim kafka_2.12-2.6.0/config/server.properties
inter.broker.protocol.version=2.1  # 브로커 간의 내부 통신을 2.1 버전으로 
log.message.format.version=2.1     # 메시지 포맷을 2.1 버전으로
```

위 설정을 수정하지 않고, 브로커 한 대를 2.6 버전으로 실행하면, 나머지 2.1 버전 브로커와 통신이 불가하다.

### 브로커 버전 업그레이드

방식은 **한 대씩 롤링 업그레이드**

- 브로커를 종료하면, 해당 브로커에 속한 리더 파티션이 다른 브로커로 변경된다.
    - 이 때 일시적으로 카프카 클라이언트에서 리더를 찾지 못하는 에러가 발생하거나, 타임아웃 등이 발생할 수 있으나 당연한 현상임
    - 내부적으로 클라이언트는 재시도하여 새로운 리더가 있는 브로커를 찾음

**브로커의 업그레이드 과정**은 아래와 같다.

1. 브로커 한 대 접속
2. /usr/local 경로 이동
3. 브로커 종료
4. kafka 링크 삭제
5. kafaka 링크 2.6 버전으로 재생성
6. 설정 파일 복사 및 옵션 설정
7. 브로커 시작

위 과정을 **모든 브로커에 적용**해야 한다.

과정을 좀 더 자세히 보면,

1. 브로커 한 대씩 2.6 버전으로 업그레이드

```bash
# 카프카 종료
$ cd /usr/local/
sudo systemctl stop kafka-server

# 심볼릭 링크 2.6버전 카프카로 변경
$ sudo rm -rf kafka
$ sudo ln -sf kafka_2.12-2.6.0 kafka
$ ll
lrwxrwxrwx 1 root root   27  1월 01 00:08 kafka -> /usr/local/kafka_2.12-2.6.0

# 카프카 시작
$ sudo systemctl start kafka-server
```

<img width="816" alt="Untitled" src="https://user-images.githubusercontent.com/58337059/214269926-abf05f31-924f-4edd-a3f7-d8f3001d29d8.png">

아직 통신 버전과 메시지 포멧은 2.1 버전이다.

1. 나머지 브로커도 2.6 버전으로 업그레이드

<img width="818" alt="Untitled (1)" src="https://user-images.githubusercontent.com/58337059/214270089-c1a57b11-168d-4106-a84a-62e8b4894b0d.png">


### 브로커 설정 변경

브로커를 모두 2.6 버전으로 업그레이드를 했으나, 2.1 버전으로 통신 중이므로 2.1 통신 설정을 제거한다. 이후 한 대씩 다시 재시작하여 업그레이드를 완료한다.

```bash
# 2.1 버전 설정 제거
$ sudo vim /usr/local/kafka/config/server.properties
~~inter.broker.protocol.version=2.1
log.message.format.version=2.1~~

# kafka 한 대씩 재시작
~~~~sudo systemctl restart kafka-server
## ...

# 재시작 후 프로듀서가 메시지를 잘 전송하는지 확인
$ /usr/local/kafka/bin/kafka-console-producer.sh --bootstrap-server peter-kafka01.foo.bar:9092 --topic peter-version2-1
> version2-1-message4
> version2-1-message5

# 기존 컨슈머 그룹의 오프셋을 기억하고, 이후 메시지를 잘 수신하는지 확인
$ /usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server peter-kafka01.foo.bar:9092 --topic peter-version2-1 --group peter-consumer
version2-1-message4
version2-1-message5

# 버전 업그레이드 전의 메시지를 가져올 수 있는지 확인
$ /usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server peter-kafka01.foo.bar:9092 --topic peter-version2-1 --from-beginning
version2-1-message1
version2-1-message2
version2-1-message3
version2-1-message4
version2-1-message5
```

### 업그레이드 작업 시 주의사항

1. 운영환경과 동일한 테스트 환경 구축 후, 버전 업그레이드를 수행하여 특이사항을 미리 체크
2. 카프카 트래픽이 적은 시간에 업그레이드
    - 브로커 종료, 시작 시 리더가 변경되고 팔로워들은 새로운 리더의 로그 세그먼트를 일치시키는 작업(리플리케이션)이 일어나므로, 트래픽이 적어야 빨리 완료됨
3. 프로듀서에서 ack=1 옵션을 사용하면, 롤링 업데이트시 일부 메시지가 손실될 수 있음

---

## 카프카의 확장

카프카는 확장이 쉽도록 디자인되었다.

카프카의 확장 과정과 방법을 살펴보자.

```bash
# 브로커 3대, 파티션 4개
$ /usr/local/kafka/bin/kafka-topic.sh --bootstrap-server peter-kafka01.foo.bar:9092 --describe --topic peter-scaleout1
Topic: peter-scaleout1   PartitionCount: 4   ReplicationFactor: 1   Configs: segment.bytes=1073741822
Topic: peter-scaleout1   **Partition: 0     Leader: 3     Replicas: 3     Isr: 3**
Topic: peter-scaleout1   Partition: 1     Leader: 1     Replicas: 1     Isr: 1
Topic: peter-scaleout1   Partition: 2     Leader: 2     Replicas: 2     Isr: 2
Topic: peter-scaleout1   **Partition: 3     Leader: 3     Replicas: 3     Isr: 3**
```

![Untitled (2)](https://user-images.githubusercontent.com/58337059/214270244-d550b09f-fc49-4525-bbe7-a30158d79e8c.png)


브로커 3대에, 파티션 4개가 분배가 되어있다. 3번 브로커에 파티션이 두 개 있다.

이후 브로커를 한 대 추가한다. 새로운 브로커는 server.properties 설정 파일에서 **broker.id=4** 로 설정한다. (기존 broker.id와 중복되지 않는 값으로 부여해야한다.)

![Untitled (3)](https://user-images.githubusercontent.com/58337059/214270505-e395cdfb-da67-4de6-bb69-52a9c0e9d224.png)


브로커를 추가해도 파티션의 분배는 자동으로 일어나지 않는다. 분배하려면 관리자가 수동으로 파티션을 고르게 분산시켜야 한다.

이 때 파티션이 4개인 새로운 토픽을 추가한다면, 새로운 파티션들은 고르게 분산된다.

```bash
$ /usr/local/kafka/bin/kafka-topic.sh --bootstrap-server peter-kafka01.foo.bar:9092 --create --topic peter-scaleout2 --partitions 4 --replication-factor 1
Created topic peter-scaleout2.

$ /usr/local/kafka/bin/kafka-topic.sh --bootstrap-server peter-kafka01.foo.bar:9092 --describe --topic peter-scaleout2
Topic: peter-scaleout2   PartitionCount: 4   ReplicationFactor: 1   Configs: segment.bytes=1073741824
Topic: peter-scaleout2   **Partition: 0     Leader: 1     Replicas: 1     Isr: 1**
Topic: peter-scaleout2   **Partition: 1     Leader: 2     Replicas: 2     Isr: 2**
Topic: peter-scaleout2   **Partition: 2     Leader: 3     Replicas: 3     Isr: 3**
Topic: peter-scaleout2   **Partition: 3     Leader: 4     Replicas: 4     Isr: 4**
```

![Untitled (4)](https://user-images.githubusercontent.com/58337059/214270594-58006ce8-30a9-4b18-ac65-136b0ec24157.png)


여전히 1번 토픽의 파티션 2개가 3번 브로커에 있어 브로커 3번이 부담을 받고 있다.

### 브로커의 부하 분산

카프카에서 제공하는 kafka-reassign-partitions.sh 라는 도구를 이용하여 수동으로 파티션을 이동시킬 수 있다. 실습에서는 scaleout1 토픽을 분산 대상으로 지정한다.

1. 먼저 파티션 이동 작업을 위해 정해진 JSON 포맷으로 파일을 생성해야 한다.

```json
{"topics":
    [{"topic": "peter-scaleout1"}],
    "version":1
}
```

1. 이 후 kafka-reassign-partitions.sh 명령어로 분산시킬 브로커 리스트를 지정한다.

```bash
$ /usr/local/kafka/bin/kafka-reassign-partitions.sh --bootstrap-server peter-kafka01.foo.bar:9092 --generate --topics-to-move-json-file reassign-partitions-topic.json --broker-list "1,2,3,4"
Current partition replica assignment
{"version":1,"paritions":[
{"topic":"peter-scaleout1","partition":0,"replicas":[3],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":1,"replicas":[1],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":2,"replicas":[2],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":3,"replicas":[3],"log_dirs":["any"]}]}

Proposed partition reassignment configuration
{"version":1,"paritions":[
{"topic":"peter-scaleout1","partition":0,"replicas":[2],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":1,"replicas":[3],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":2,"replicas":[4],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":3,"replicas":[1],"log_dirs":["any"]}]}
```

실행 스크립트 결과를 보면 **현재 설정된 파티션 배치**를 보여주고, 이후 **제안하는 파티션 배치**를 보여준다.

1. 제안하는 파티션 배치 json 부분을 move.json 이라는 파일을 생성하여 담고, 실제 파티션 배치를 실행한다.

```json
{
	"version": 1,
	"partitions": [
		{
			"topic": "peter-scaleout1",
			"partition": 0,
			"replicas": [
				2
			],
			"log_dirs": [
				"any"
			]
		},
		{
			"topic": "peter-scaleout1",
			"partition": 1,
			"replicas": [
				3
			],
			"log_dirs": [
				"any"
			]
		},
		{
			"topic": "peter-scaleout1",
			"partition": 2,
			"replicas": [
				4
			],
			"log_dirs": [
				"any"
			]
		},
		{
			"topic": "peter-scaleout1",
			"partition": 3,
			"replicas": [
				1
			],
			"log_dirs": [
				"any"
			]
		},
	]
}
```

```bash
$ /usr/local/kafka/bin/kafka-reassign-partitions.sh --bootstrap-server peter-kafka01.foo.bar:9092 --reassignment-json-file move.json --execute
Current partition replica assignment

{"version":1,"paritions":[
{"topic":"peter-scaleout1","partition":0,"replicas":[3],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":1,"replicas":[1],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":2,"replicas":[2],"log_dirs":["any"]},
{"topic":"peter-scaleout1","partition":3,"replicas":[3],"log_dirs":["any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started partition reassignments for peter-scaleout1-0,peter-scaleout1-1,peter-scaleout1-2,peter-scaleout1-3
```

![Untitled (5)](https://user-images.githubusercontent.com/58337059/214270725-ff2c9f60-bcff-49cb-8db7-8f7db8473d86.png)

3번 브로커에 있던 2번 파티션이 4번 브로커에 배치되었다.

```bash
$ /usr/local/kafka/bin/kafka-topic.sh --bootstrap-server peter-kafka01.foo.bar:9092 --describe --topic peter-scaleout1
Topic: peter-scaleout1   PartitionCount: 4   ReplicationFactor: 1   Configs: segment.bytes=1073741824
Topic: peter-scaleout1   **Partition: 0     Leader: 2     Replicas: 2     Isr: 2**
Topic: peter-scaleout1   **Partition: 1     Leader: 3     Replicas: 3     Isr: 3**
Topic: peter-scaleout1   **Partition: 2     Leader: 4     Replicas: 4     Isr: 4**
Topic: peter-scaleout1   **Partition: 3     Leader: 1     Replicas: 1     Isr: 1**
```

정리하면,

1. 브로커를 추가해도 기존 파티션은 자동 분산되지 않는다.
2. 신규 토픽은 파티션을 고루 분산한다.
3. 분산 밸런스를 맞추려면 수동으로 분산 작업을 진행해야 한다.

### 분산 배치 작업 시 주의사항

1. 카프카의 사용량이 낮은 시간에 진행하는 것을 추천한다.
    - 파티션 재배치 과정은 내부적으로 리플리케이션이 일어나기 때문
    - ![Untitled (6)](https://user-images.githubusercontent.com/58337059/214270804-de6839ef-b63d-45fd-9f85-6efe2514ab35.png)
      - 3번 파티션을 2번 브로커로 리플리케이션을 시키고, 완료되면 1번 브로커에서 파티션을 삭제한다.
    - 파티션의 개수와 리플리케이션 팩터가 많으면 부하가 급증한다.
2. 확장할 때 보관 주기를 줄이는 것은 부하를 크게 줄일 수 있다.
    1. 컨슈머가 최근 내용까지 모두 컨슘했고, 재처리 할 일이 없다면, 최근 메시지를 삭제해도 무방하다.
    2. 토픽의 보관주기를 1주일에서 1일로 줄이면 700GB였던 파티션이 100GB로 줄 수 있다.
        - 네트워크 비용이 크게 준다.
3. 재배치 작업은 여러 개의 토픽을 동시에 진행하지 않고, 단 하나씩만 진행한다면 부하를 줄일 수 있다.

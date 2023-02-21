# 카프카 커넥트

# 0. 카프카 커넥트란?

아파치 카프카의 오픈소스 프로젝트 중 하나

데이터베이스 같은 외부 시스템과 카프카를 연결하기 위한 프레임워크

코드를 작성하지 않고 간단하게 사용 가능

장점

- 데이터 중심 파이프라인
    - 커넥트를 이용해 카프카로 데이터의 입출력 가능
- 유연성과 확장성
    - 일회성 작업을 위한 단독 모드와 대규모 운영 환경을 위한 분산 모드로 사용 가능
- 재사용성과 기능 확장
    - 이미 만들어진 기본 커넥터를 활용할 수 있고 요구사항에 맞춰 확장 가능
- 장애 및 복구
    - 분산 모드로 실행하면 워커 노드의 장애 상황에도 유연하게 대응 가능→고가용성 보장

# 1. 카프카 커넥트의 핵심 개념

![https://turkogluc.com/content/images/2020/09/Screenshot-2020-09-12-at-15.35.45.png](https://turkogluc.com/content/images/2020/09/Screenshot-2020-09-12-at-15.35.45.png)

출처 : [https://turkogluc.com/apache-kafka-connect-introduction/](https://turkogluc.com/apache-kafka-connect-introduction/)

카프카 클러스터를 먼저 구성한 후 카프카 클러스터의 양쪽 옆에 배치 할 수 있는데 소스 방향에 있는 커넥트를 소스 커넥트(프로듀서 역할), 싱크 방향에 있는 커넥트를 싱크 커넥트(컨슈머 역할)라고 함

## 워커

워커는 카프카 커넥트 프로세스가 실행되는 서버 또는 인스턴스 등을 의미, 커넥터나 태스크들이 워커에서 실행

단일 모드는 하나의 워커에서만 동작하고 분산 모드에서는 특정 워커에 장애가 발생해도 해당 워커에서 동작 중인 커넥터나 태스크들이 다른 워커로 이동해 연속해서 동작할 수 있음

## 커넥터

커넥터는 데이터를 어디에서 어디로 복사해야 하는지의 작업을 정의하고 관리하는 역할

커넥트와 동일하게 소스 커넥터와 싱크 커넥터가 있음

ex)

RDBMS→JDBC 소스 커넥터→HDFS 싱크 커넥터→HDFS

## 태스크

커넥터가 정의한 작업을 직접 수행하는 역할

커넥터는 데이터 전송에 관한 작업을 정의한 후 각 태스크들을 워커에 분산

# 2. 카프카 커넥트의 내부 동작

분산 배치된 각 태스크들은 메시지들을 소스에서 카프카로 혹은 카프카에서 싱크로 이동

→ 커넥트는 파티셔닝 개념을 적용해 데이터들을 하위 집합으로 나눔

<aside>
💡 카프카에서도 병렬 처리를 위해 토픽을 파티션으로 나누는데 커넥트도 이와 동일함,
다만 커넥트에서 나눈 파티션과 토픽의 파티션은 용어만 같을 뿐 서로 관계는 없음

</aside>

→ 파티션들에는 오프셋과 같이 순차적으로 레코드들이 정렬됨

![https://docs.confluent.io/platform/current/_images/connector-model.png](https://docs.confluent.io/platform/current/_images/connector-model.png)

출처 : [https://docs.confluent.io/platform/current/connect/devguide.html#partitions-and-records](https://docs.confluent.io/platform/current/connect/devguide.html#partitions-and-records)

Stream 영역 : 데이터가 파티셔닝된 것을 나타냄

Connector 영역 : 최대 태스크 수는 2로 정의→스트림에서 나뉜 각 파티션들은 2개의 태스크에 할당

각 파티션들에는 오프셋도 함께 포함되어 있어서 커넥트의 장애나 실패가 발생할 경우 지정된 위치부터 데이터를 이동할 수 있음.

커넥터에 따라 오프셋의 기준이 달라질 수 있음

- 파일 전송시 : 파일의 위치
- 데이터 베이스 : 타임스탬프, 시퀀스 ID등

# 3. 단독 모드 카프카 커넥트

카프카 커넥트를 이용하는 실습은 카프카 클러스터와 연결해 진행하므로, 새로운 클러스터 구성하기

```bash
$ cd kafka2/
$ cd chapter2/ansible_playbook
$ ansible-playbook -i host kafka1.yml
```

- kafka1.yml
    
    ```bash
    ---
    - hosts: kafkahosts
      become: true
      connection: ssh
      vars:
        - zookeeperinfo: peter-zk01.foo.bar:2181,peter-zk02.foo.bar:2181,peter-zk03.foo.bar:2181/kafka1
        - dir_path: /data/kafka1-logs
      roles:
        - common
        - kafka
    ```
    

## 파일 소스 커넥터 실행

파일 

```bash
name=local-file-source # 커넥터에서 식별하는 이름
connector.class=FileStreamSource # 커넥터에서 사용하는 클래스
task.max=1 # 실제 작업을 처리하는 태스크의 최대 수를 1로 지정(단독 모드)
file=/home/ec2-user/test.txt # 파일 소스 커넥터가 읽을 파일
topic=connect-test # 파일 소스 커넥터가 일근 내용을 카프카의 connect-test 토픽으로 전송
```

```bash
bootstrap.servers=localhost:9092 # 브로커 주소
# 카프카로 데이터를 보내거나 가져올 때 사용하는 포맷 지정, 키와 밸류를 각각 지정
# JSON 외에도 Avro, String, ByteArray등이 있음
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
# 스키마가 포함된 구조를 사용할 수 있음 여기서는 false
key.converter.schemas.enable=false
value.converter.schemas.enable=false
# 재처리 등을 목적으로 오프셋을 파일로 저장
offset.storage.file.filename=/tmp/connect.offsets
offset.flush.interval.ms=10000 # 오프셋 플러시 주기를 설정
```

<aside>
💡 **카프카 커넥트의 컨버터**는
소스에서 카프카로 전송할 때의 직렬화와 카프카에서 싱크로 전송할 때의 역직렬화를 담당

소스→소스 카프카 커넥트[커넥터→컨버터]→카프카→싱크 카프카 커넥트[컨버터→커넥터]→싱크

</aside>

설정 파일을 작성 후 단독 모드 커넥터를 실행

```bash
$ sudo /usr/local/kafka/bin/connect-standalone.sh -daemon \
usr/local/kafka/config/connect-standalone.properties \
usr/local/kafka/config/connect-file-source.properties
```

REST API로 설정이 잘 됐는지 확인 가능

```bash
$ curl http://localhost:8083/connectors/local-file-source | python -m json.tool
```

## 파일 싱크 커넥터 실행

파일 소스 커넥터는 설정 파일을 로드하면서 실행했지만 카프카 커넥트의 REST API를 이용해 실행할 수도 있음

```bash
$ curl --header "COntent-Type: application/json" --header \
"Accept: application/json" --request PUT --data '{
"name": "local-file-sink", # 커넥터에서 식별하는 이름
"connector.class": "FileStreamSink", # 커넥터에서 사용하는 클래스
"tasks.max": "1", # 태스크의 최대 수
"file": "/home/ec2-user/test.sink.txt", # 가져온 메시지를 저장할 파일
"topics": "connect-test" # 메세지를 가져올 토픽
}' http://localhost:8083/connectors/local-file-sink/config
```

테스트를 마무리 하려면

```bash
$ sudo pkill -f connect
```

## 카프카 커넥트 REST API

| API 옵션 | 설명 |
| --- | --- |
| GET / | 커넥트의 버전과 클러스터 ID 확인 |
| GET /connectors | 커넥터 리스트 확인 |
| GET /connectors/[커넥터 이름] | 커넥터 이름의 상세 내용 확인 |
| PUT /connectors/[커넥터 이름]/config | 커넥터 이름의 config 정보 확인 |
| PUT /connectors/[커넥터 이름]/status | 커넥터 이름의 상태 확인 |
| PUT /connectors/[커넥터 이름]/pause | 커넥터의 일시 중지 |
| PUT /connectors/[커넥터 이름]/resume | 커넥터의 다시 시작 |
| DELETE /connectors/[커넥터 이름] | 커넥터의 삭제 |
| GET /connectors/[커넥터 이름]/tasks | 커넥터의 태스크 정보 확인 |
| GET /connectors/[커넥터 이름]/tasks/[태스크 ID]/status | 커넥터에서 특정 태스크의 상태 확인 |
| GET /connectors/[커넥터 이름]/tasks/[태스크 ID]/restart | 커넥터에서 특정 태스크 시작 |

# 4. 분산 모드 카프카 커넥트

단독 모드의 카프카 커넥트는 데모용이나 테스트용으로 적합

운영 환경에서는 분산 모드 카프카를 권장

단독 모드와 분산 모드의 가장 중요한 차이점은 메타 정보의 저장소 위치

- 분산 모드는 메타 정보의 저장소로 카프카 내부 토픽을 이용

커넥트의 메타 정보를 워커 자신의 로컬 디스크에 저장한다고 가정하고, 장애가 발생시 메타 정보를 다른 워커에 공유할 수 없기 때문에 분산 모드의 장점을 얻지 못함

카프카 커넥트에서 사용하는 토픽들은 커넥트 운영에서 중요한 정보가 저장되어 있으므로, 리플리케이션 팩터 수는 반드시 3으로 설정해야 함→확장성, 장애 허용, 자동 리밸런싱 등 운영에 필요한 필수 기능을 제공

## 리밸런싱

카프카 커넥트의 부하가 높아져 확장이 필요한 상황이라고 가정 후 워커를 추가하면 즉시 확장됨가 동시에 내부적으로 자동 리밸런싱 동작

자동 리밸런싱은 워커들 안에서 태스크와 커넥터가 최대한 균등하게 배치될 수 있게 함

ex)

워커1[커넥터A, 태스크A1], 워커2[태스크A2, 커넥터B], 워커3[태스크B1], 워커4[]

→워커1[커넥터A, 태스크A1], 워커2[태스크A2], 워커3[태스크B1], 워커4[커넥터B]

## 장애허용

카프카 커넥트의 워커들은 안정적으로 운영 중이지만 하드웨어의 장애 등이 발생하면 언제든 종료 가능

ex)

워커1[커넥터A, 태스크A1], 워커2[태스크A2], ~~워커3[태스크B1]~~, 워커4[커넥터B]

→워커1[커넥터A, 태스크A1], 워커2[태스크A2,태스크B1], 워커4[커넥터B]

워커3 장애발생→워커3 종료→태스크B1은 워커2로 이동

```bash
bootstrap.servers=peter-kafka01.foo.bar:9092,peter-kafka02.foo.bar:9092,\
peter-kafka03.foo.bar:9092
group.id=peter-connect-cluster # 분산 모드의 그룹 아이디 컨슈머 그룹의 그룹 아이디와 동일한 개념
key.converter=org.apache.kafka.connect.convertes.ByteArrayConverter
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter.schemas.enalbe=false
value.converter.schemas.enable=false
# 커넥터들의 오프셋 추적을 위해 저장하는 카프카 내부 토픽
offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=25
# 커넥터들의 설정을 저장하는 카프카 내부 토픽
config.storage.topic=connect-configs
config.storage.replication.factor=3
config.storage.partitions=1
# 커넥터들의 상태를 저장하는 카프카 내부 토픽
status.storage.topic=connect-status
status.storage.replication.factor=3
status.sotrage.prtitions=5

offset.flush.interval.ms=10000
```

분산모드 커넥트 설정 파일 점검이 완료되면 모든 브로커 서버에서 [connect-distributed.sh](http://connect-distributed.sh) 파일을 실행

위의 설정 파일을 로드하도록 명시

```bash
$ sudo systemctl start kafka-connect
$ sudo systemctl status kafka-connect # Active: active (running)이 확인되면 정상 실행
```

카프카 커넥트를 분산 모드로 실행하려면 장애 대응 및 리밸런싱 동작 등을 위해 최소 2대 이상의 워커로 구성

# 5. 커넥터 기반의 미러 메이커 2.0

카프카를 용도별로 구분하는 경우도 있음 이 경우 실시간 용도의 업스트림 카프카와 배치 작업을 위한 다운스트림 카프가로 구분하는데 업스트림 카프카에서 다운스트림 카프카로 리플리케이션 구성을 해 데이터를 전송

카프카와 카프카간 리플리케이션을 하기 위한 도구 중 하나가 바로 미러 메이커

아파치 카프카에서는 미러 메이커를 기본 도구로 제공

but, 초기 1.0 버전의 미러 메이커는 간단한 컨슈머와 프로듀서가 내장된 도구일 뿐이었음

- 원격 토픽 생성 시 기본 옵션으로 생성
- 소스 토픽의 옵션을 원격 토픽에 적용하지 못함
- 한번 적용한 미러 메이커의 설정을 변경하려면 미러 메이커를 재시작 해야함

따라서 일부 기업들이 개선된 버전을 공개함

- 브루클린(Brooklin 링크드인, 2016)
- 유리플리게이터(uReplicator 우버, 2015)
- 리플리케이터(Replicator 컨플루언트)

카프카 오픈소스 진영에서는 2019년 12월 16일 카프카 2.4 버전과 함께 미러메이커 2.0을 공개

## 원격 토픽과 에일리어스 기능

미러 메이커 1.0에서는 미러링하는 대상의 토픽명이 소스 카프카와 원격 카프카 둘 다 동일

단반향의 미러링에서는 동일한 토픽명에서 비롯되는 문제가 없었으나,

양방향 미러링의 경우는 동일한 토픽명으로 인해 무한루프와 순서가 뒤섞이는 경우 발생

미러 메이커 1.0의 이러한 한계점을 개선한 미러 메이커 2.0은

**에일리어스**를 추가해 서로의 토픽명을 구분할 수 있게 함

![https://cwiki.apache.org/confluence/download/attachments/95650722/Screen%20Shot%202018-10-11%20at%203.21.13%20PM.png?version=1&modificationDate=1539289287000&api=v2](https://cwiki.apache.org/confluence/download/attachments/95650722/Screen%20Shot%202018-10-11%20at%203.21.13%20PM.png?version=1&modificationDate=1539289287000&api=v2)

출처 : [https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0)

## 카프카 클러스터 통합

미러메이커 2.0을 이용하면 다중 클러스터로부터 미러링된 토픽들을 다운스트림 컨슈머가 통합할 수 있음

- us-west → us-west.topic1
- us-east → us-east.topic1
- local → topic1

각각의 토픽 이름이 제각기 다르므로 원본 토픽의 내용과 정확하게 일치하며, 관리자는 데이터를 처리함에 있어 원하는 형태로 토픽을 컨슘 가능 → 굳이 통합을 목적으로 하는 별도의 카프카 클러스터를 구성하지 않아도 됨

## 무한 루프 방지

미러메이커 2.0에서는 관리자가 동시에 2개의 클러스터를 서로 리플리케이션할 수 있도록 구성 가능한데 토픽의 이름 앞에 에일리어스를 추가함으로써 무한 루프를 방지할 수 있음

## 토픽 설정 동기화

미러메이커 2.0에서는 소스 토픽을 모니터링하고 토픽의 설정 변경사항을 원격의 대상 토픽으로 전파함

이러한 기능으로 원격 토픽의 설정을 실수로 누락해도 자동으로 적용

## 안전한 저장소로 내부 토픽 활용

미러 메이커 2.0은 내부적으로 health check를 하며, 주기적으로 미러링 관련 토픽, 파티션, 오프셋 정보등을 저장하기 위해 가장 안전한 저장소인 카프카의 내부 토픽을 사용

- 하트비트
- 컨슈머 그룹의 오프셋 정보를 위한 체크포인트
- 파티션 리플리케이션 체크를 위한 오프셋 싱크

## 카프카 커넥트 지원

미러 메이커 2.0은 카프카 커넥트 프레임워크를 기반으로 성능, 신뢰성과 확장성을 높임

미러 메이커 2.0을 실행하는 방법은 다음과 같은 네 가지가 있음

- 전용 미러 메이커 클러스터
- 분산 커넥트 클러스터에서 미러 메이커 커넥터 이용
- 독립형인 커넥터 워커
- 레거시 방식인 스크립트 사용

태스크는 실제 데이터를 이동, 복사하는 역할을 하므로 태스크 수가 많은편이 좋기는 함

태스크 개수는 REST API를 이용해 운영 중에 언제라도 다이내믹하게 조정할 수 있음

모니터링을 통해 적절한 태스크를 찾아 설정하면 됨

# 6. 정리

실습용 커넥터로 미러 메이커 커넥터를 소개했지만,

데비지움(debezium)에서

MySQL, 몽고DB, PostgreSQL 등의 데이터베이스 데이터와 카프카를 연동하기 위한 커넥터를 공개하고 있음

변경 데이터 캡쳐를 고민중이라면 데비지움의 커넥터를 이용하는걸 추천

****[Confluent Hub](https://www.confluent.io/hub/?_ga=2.106604426.826714862.1672895555-1259293887.1672895555)****에서 다양한 허브들을 지원중이니 확인해보고 필요한게 있으면 사용해보면 좋을듯

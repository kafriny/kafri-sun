# 4. 카프카의 내부 동작 원리와 구현
## 카프카 리플리케이션
### 리플리케이션 동작 개요
- `replication factor` 옵션을 이용해서 리플리케이션 설정
- 토픽 생성
    ``` sh
    [centos@grm-kafka4 bin]$ ./kafka-topics.sh --bootstrap-server grm-kafka4.novalocal:9092 --create --topic garimoo-test01 --partitions 1 --replication-factor 3
    Created topic garimoo-test01.
    ```
- 토픽 상세 정보
    ```
    [centos@grm-kafka4 bin]$ ./kafka-topics.sh --bootstrap-server grm-kafka4.novalocal:9092 --topic garimoo-test01 --describe
    Topic: garimoo-test01	PartitionCount: 1	ReplicationFactor: 3	Configs: segment.bytes=1073741824
        Topic: garimoo-test01	Partition: 0	Leader: 6	Replicas: 6,4,5	Isr: 6,4,5
    ```
     - 파티션 수는 1, 리플리케이션 팩터는 3
     - 파티션0의 리더는 브로커6, 리플리케이션들은 6,4,5에 각각 존재한다는 것을 의미 

- 메세지 전송
    ```
    [centos@grm-kafka4 bin]$ ./kafka-console-producer.sh --bootstrap-server grm-kafka4.novalocal:9092 --topic garimoo-test01
    >test message1
    ```
 - 세그먼트 파일 내용 확인
     ```
     [centos@grm-kafka4 ~]$ /usr/local/kafka/bin/kafka-dump-log.sh --print-data-log --files /data/kafka-logs/garimoo-test01-0/00000000000000000000.log
    Dumping /data/kafka-logs/garimoo-test01-0/00000000000000000000.log
    Starting offset: 0
    baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 0 CreateTime: 1667386691239 size: 81 magic: 2 compresscodec: NONE crc: 1631414717 isvalid: true
    | offset: 0 CreateTime: 1667386691239 keysize: -1 valuesize: 13 sequence: -1 headerKeys: [] payload: test message1
     ```
     -  시작 오프셋 위치는 0
     -  메세지 카운트 수 1
     -  프로듀서를 통해 보낸 메세지 내용도 확인 가능 (payload)
  - 다른 브로커에서(6번장비) 확인해도 동일
      ```
      [centos@grm-kafka6 kafka]$ /usr/local/kafka/bin/kafka-dump-log.sh --print-data-log --files /data/kafka-logs/garimoo-test01-0/00000000000000000000.log
    Dumping /data/kafka-logs/garimoo-test01-0/00000000000000000000.log
    Starting offset: 0
    baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 0 CreateTime: 1667386691239 size: 81 magic: 2 compresscodec: NONE crc: 1631414717 isvalid: true
    | offset: 0 CreateTime: 1667386691239 keysize: -1 valuesize: 13 sequence: -1 headerKeys: [] payload: test message1
      ```

> `replication factor` 수만큼 데이터를 복제해서 가지고 있기 때문에 브로커 장애가 발생해도 메세지 손실 없이 안정적으로 메세지를 주고받을 수 있음
### 리더와 팔로워
- 복제본중 하나가 리더로 선정되며, 모든 읽기와 쓰기는 리더를 통해서만 가능
![image](https://user-images.githubusercontent.com/20420626/199745787-e5b5db1a-18f2-4323-b95c-6ec1334dc9a6.png)
- 팔로워는 지속적으로 리더가 새로운 메세지를 받았는지 확인하고, 있다면 리더로부터 복제함
### 복제 유지와 커밋
- 리더와 팔로워는 `ISR(InSyncReplica)`라는 논리적 그룹으로 묶여 있음
    - ISR에 속하지 않은 팔로워는 새로운 리더의 자격을 가질 수 없음
- 리더는 **팔로워가 리플리케이션을 잘 따라가고 있는지** 판단하는 주체
    - 특정 주기의 시간만큼 복제 요청을 하지 않는다면 문제가 발생했다고 판단한 뒤 ISR에서 제외
    - 토픽 상세보기 커맨드를 이용해 ISR 상태를 점검하고, 토픽의 상태가 양호한지 등을 확인할 수 있음
- **커밋**: 리플리케이션 팩터 수의 모든 리플리케이션이 전부 메세지를 저장했음을 의미 
    - 커밋된 메세지만 컨슈머가 읽어갈 수 있도록 해서 메세지의 일관성을 유지
    - 마지막 커밋 오프셋 위치는 `하이 워터마크(high water mark)`라 부름
    ![image](https://user-images.githubusercontent.com/20420626/199745851-27846fc4-8e2d-4330-ad31-6e59a42543ad.png)
    - 예시)커밋되기 전 데이터를 컨수머가 읽을 수 있다면?
    ![image](https://user-images.githubusercontent.com/20420626/199745891-e2a32d03-da0c-4d87-a002-877646812e1a.png)
    - 컨수머 A,B는 같은 토픽을 읽고 있지만 같은 데이터를 읽지 못한 현상이 발생함

- 커밋된 위치는 로컬의 `replication-offset-checkpoint` 파일에 저장됨
    - 특정 토픽이나 파티션에 복제가 되지 않을 때에는 이 파일의 내용을 확인하고 비교하면 파악 가능
```
[centos@grm-kafka6 kafka-logs]$ cat replication-offset-checkpoint
0
1
garimoo-test01 0 1
```

- 메세지를 하나 더 보내고 확인
```
[centos@grm-kafka4 ~]$ /usr/local/kafka/bin/kafka-console-producer.sh --bootstrap-server grm-kafka4.novalocal:9092 --topic garimoo-test01
>hello world
```
```
[centos@grm-kafka6 kafka-logs]$ cat replication-offset-checkpoint
0
1
garimoo-test01 0 2 --> 0은 파티션 번호 / 2는 커밋된 오프셋 번호
```
### 리더와 팔로워의 단계별 리플리케이션 동작
> 리더의 성능 향상을 위해 리플리케이션시 통신을 최소화 할 수 있도록 설계

![image](https://user-images.githubusercontent.com/20420626/199745946-2b97990f-6ea3-485e-add7-28bf7fbe4d5c.png)
- 메세지 생성 (복제 전)
![image](https://user-images.githubusercontent.com/20420626/199745970-0b062156-825f-41aa-b8aa-cd6920f959a5.png)
- 팔로워 -> 리더: `fetch` 요청
- 리더는 팔로워가 요청을 보냈다는 것만 인지하며, 리플리케이션의 성공/실패 여부는 알지 못함
    - rabbitMQ 등과 같이 ACK를 하지 않음으로써 성능 향상
![image](https://user-images.githubusercontent.com/20420626/199745993-c4e5eac6-052a-498e-95db-d412a2aeaffc.png)
- 0번 오프셋에 대한 리플리케이션을 마친 팔로워는 리더에게 1번 오프셋에 대한 리플리케이션을 요청함으로써 0번이 성공했다는 것을 인지시킴
- 리더는 0에 대해 커밋처리를 한 뒤 hwm 증가
    -  만약 팔로워가 0번에 대한 복제에 실패했다면 0번에 대한 리플리케이션 재요청
- 1번 오프셋 메세지에 대한 리플리케이션 요청에 대한 응답으로 리더는 0번 오프셋이 커밋되었다는 내용을 함께 전달
![image](https://user-images.githubusercontent.com/20420626/199746029-e0721a58-61d0-49c0-8722-2a6958b8e722.png)
- 리더의 응답을 받은 모든 팔로워는 리더와 동일하게 커밋을 변경
- 1번 오프셋 메시지를 리플리케이션

#### 카프카 성능
- ACK를 제거함으로써 대량의 메세지를 처리할 때의 부하를 줄일 수 있게 됨
- 리더가 푸시하는 방식이 아니라 팔로워가 pull하는 방식으로 동작하기 때문에 리더의 부하를 줄일 수 있게 됨
### 리더에포크와 복구
- 리더에포크는 장애 복구시 메세지의 일관성을 유지하기 위한 용도로 이용됨
- 리더에포크도 리플리케이션 프로토콜에 의해 전파되며, 리더가 변경되면 변경된 리더에 대한 정보도 팔로워에 전파

#### 리더에포크가 없을 때의 장애 복구-1 
![image](https://user-images.githubusercontent.com/20420626/199746073-459d2a8e-2f50-4e92-9d70-5ce2359e11c6.png)
- 리플리케이션 팩터는 2, 파티션은 1, `min.insync.replicas`는 1
- 1번 오프셋까지 복제를 완료했지만, (2번 오프셋 요청을 하지 않았기 때문에) high watermark를 2로 올리라는 내용은 전달받지 못한 상태에서 팔로워 다운
![image](https://user-images.githubusercontent.com/20420626/199746091-87b2b053-e5d1-4bdd-bc57-89b998b5fefa.png)
- 팔로워는 자신의 워터마크보다 높은 메세지를 신뢰할 수 없는 메세지로 판단하고 삭제
![image](https://user-images.githubusercontent.com/20420626/199746116-e006ebab-7ec8-4e63-89b9-019b8153e7fa.png)
- 이 순간 리더였던 브로커가 다운되어, 팔로워가 리더로 승격됨
- message2 데이터는 유실됨

#### 리더에포크 사용시 장애 복구-1
![image](https://user-images.githubusercontent.com/20420626/199746136-6ce8f78e-a4b2-4a8f-92df-d3bf2913d07c.png)
- 그림 4-8에서 팔로워가 장애 복구된 이후의 상태
- 자신의 하이워터마크보다 높은 메세지를 바로 삭제하지 않고, 리더에게 리더에포크 요청을 보냄
- 리더는 `1번오프셋의 message2`까지 전송했다고 응답
- 팔로워는 1번 오프셋을 삭제하지 않고, 자신의 하이워터마크를 상향 조정
![image](https://user-images.githubusercontent.com/20420626/199746163-daac7b9e-5d25-4a54-bd5e-e4fc2734a498.png)
- 팔로워가 리더로 승격된 경우 메세지의 손실이 발생하지 않음


#### 리더에포크가 없을 때의 장애 복구-2
![image](https://user-images.githubusercontent.com/20420626/199746208-619e5024-e415-47ba-aeb9-6ed193148c4d.png)
- 리더만 오프셋 1, 팔로워는 복제를 완료하지 못한 상태
![image](https://user-images.githubusercontent.com/20420626/199746234-6127d6bf-964b-44f2-955c-28b22c7a5c04.png)
- 리더와 팔로워 모두 다운된 뒤 팔로워가 있던 브로커만 복구
- 새로운 메세지 message03을 전달받은 뒤 1번 오프셋에 저장, 하이워터마크를 상향 조정
![image](https://user-images.githubusercontent.com/20420626/199746262-0e46b6e8-8a11-4a3c-a1d4-9ed52f14e97e.png)
- 구 리더가 장애에서 복구되면 팔로워가 됨
- 현 리더와 자신의 하이워터마크를 비교했을 때 동일하기 때문에 자신이 갖고 있던 메세지를 삭제하지 않음 (데이터 불일치 발생)

#### 리더에포크 사용시 장애 복구-2

![image](https://user-images.githubusercontent.com/20420626/199746289-78d61870-8d91-4700-b307-e68ef136af18.png)
- 팔로워가 먼저 복구된 뒤, 구 리더가 팔로워가 됨
- 뉴 리더는 **자신이 팔로워일 때의 하이워터마크와 리더일 때의 하이워터마크를 둘 다 알고 있음**
    - 살아난 팔로워(구리더)가 리더에게 리더에포크 요청을 보냄
    - 뉴리더는 0번 오프셋까지 유효하다고 응답
    - 팔로워는 메세지 일관성을 위해 1번 오프셋을 삭제한 뒤 리더의 1번 오프셋 데이터를 받아옴

#### 리더에포크
```
0
1 -> 현재의 리더 에포크 번호
0 0 -> 리더 에포크 번호: 0 / 새로운 메세지를 전송받게 될 오프셋 번호
```
![image](https://user-images.githubusercontent.com/20420626/199746310-2bbb9189-8e8a-461b-baf9-1383494e4fd9.png)
```
0
1 -> 현재의 리더 에포크 번호
0 0 -> 리더 에포크 번호: 0 / 새로운 메세지를 전송받게 될 오프셋 번호
```
![image](https://user-images.githubusercontent.com/20420626/199746334-331d1de5-a96e-4a31-a46f-a60c7ecb6dec.png)
- 브로커 종료 이후
```
0
2 -> 현재의 리더 에포크 번호
0 0 -> 리더 에포크 번호: 0 / 새로운 메세지를 전송받게 될 오프셋 번호: 0
1 1 -> 리더 에포크 번호: 1 / 새로운 메세지를 전송받게 될 오프셋 번호: 1
```
![image](https://user-images.githubusercontent.com/20420626/199746356-b153487f-eb35-45e7-b7a9-d7021a4f60ae.png)
- 다운되었던 브로커가 살아나면 자신의 마지막 리더 에포크 번호가 1이므로 새로운 리더에게 1번 리더에포크에 대한 요청을 보냄
- 새로운 리더는 현재 준비된 오프셋 위치가 1이라는 응답을 보냄

> 팔로워는 자신의 하이워터마트보다 높은 오프셋의 메세지를 무조건 삭제하지 않고, 먼저 리더에게 리더 에포크 요청을 보내 응답을 받아서 최종 커밋된 오프셋 위치를 확인
## 컨트롤러
- 카프카 클러스터 중 하나의 브로커가 컨트롤러 역할
    - 리더를 선출하기 위한 ISR 리스트 정보는 주키퍼에 저장되어있음
    - 브로커 상태를 체크하며, 실패가 감지되면 ISR 리스트 중 하나를 새로운 파티션 리더로 선출
    - 새로운 리더의 정보를 주키퍼에 기록
    - 변경된 정보를 모든 브로커에 전달
### 장애 상황에서의 리더 선출 과정
![image](https://user-images.githubusercontent.com/20420626/199746387-138f54b1-b3fb-40a3-a720-172fd90fdd6c.png)
1. 리더 브로커 다운
2. 주키퍼는 ISR에 변화가 생겼음을 감지
3. 컨트롤러는 주키퍼 워치를 통해 파티션에 변화가 생겼음을 감지, 해당 파티션 ISR 중 3번을 새로운 리더로 선출
4. 컨트롤러는 파티션의 새로운 리더가 3이라는 것을 주키퍼에 기록
5. 갱신된 정보는 모든 브로커에 전파

- 불필요한 로깅을 없애고 주키퍼 비동기 API의 개발을 통해 리더 선출 과정이 개선됨

### 제어된 종료 과정
![image](https://user-images.githubusercontent.com/20420626/199746408-8db7e603-f385-47a6-ade4-2a04d8eb927d.png)
1. 브로커가 종료됨 (SIG_TERM)
2. 브로커가 컨트롤러에게 전달
3. 컨트롤러는 리더 선출 작업을 진행, 해당 정보를 주키퍼에 기록
4. 컨트롤러는 새로운 리더 정보를 다른 브로커들에게 전송
5. 컨트롤러는 종료 요청을 보낸 브로커에게 정상 종료한다는 응답 전달
6. 응답을 받은 브로커는 캐시에 있는 내용을 디스크에 저장하고 종료

#### 제어된 종료와 급작스러운 종료의 차이
- 제어된 종료
    - 브로커가 종료되기 전 해당 브로커가 리더로 할당된 전체 파티션에 대해 리더 선출 작업을 진행하기 때문에 파티션의 다운타임이 최소화됨
    - 로그를 디스크에 동기화한 다음 종료되기 때문에 복구 시간이 짧음
    - `controlled.shutdown.enable = true` 설정이 `server.properties`에 적용되어있어야 함
- 브로커 장애로 인한 종료
    - 리더가 종료된 상태에서 컨트롤러는 순차적으로 리더 선출
    - 첫 대상 파티션의 다운타임은 짧을 수 있어도, 마지막 대상 다운타임은 오래 걸릴 수 있음

## 로그(로그 세그먼트)
- 메세지는 세그먼트라는 파일에 저장됨
    - 메세지의 키, 밸류, 오프셋, 메세지 크기 데이터 
    - 브로커의 로컬 디스크에 보관됨
    - 기본값은 1GB, 1GB에 도달하면 새로운 로그 세그먼트를 생성


### 로그 세그먼트 삭제
- `server.properties`의 `log.cleanup.policy`를 `delete`로 명시 (기본값)
- 삭제 작업은 5분 주기(기본값)로 동작
- 로그 세그먼트 파일은 오프셋 시작 번호를 이용해 파일 이름을 생성

#### 로그 세그먼트 삭제 예제
- `retention.ms` 값 변경하여 로그 세그먼트 삭제되는 예제
- 기본 설정값은 7일
- `retention.bytes`를 설정하면 지정된 크기를 기준으로 삭제할 수 있음
- 설정 내용 변경
```
/usr/local/kafka/bin/kafka-configs.sh ~~~~  --add-config retention.ms=0 --alter
```
- 설정 내용 확인
```
/usr/local/kafka/bin/kafka-topics.sh ~~~~  --describe
```
- 설정 내용 변경
```
/usr/local/kafka/bin/kafka-configs.sh ~~~~  --delete-config retention.ms --alter
```
### 로그 세그먼트 컴팩션
- 로그를 삭제하지 않고 컴팩션해서 보관
- 활성화된 세그먼트를 제외한 나머지 세그먼트를 대상으로 컴팩션
- 용량을 줄이기 위해 메세지의 키값을 기준으로 마지막 데이터만 보관
- 메세지의 키값을 기준으로 **과거 정보는 중요하지 않고 가장 마지막 값만 필요할 경우에 사용 가능**
    - ex) 주문자의 구매 현황 상태
- 키를 기준으로 압축하므로 키가 없는 경우 사용할 수 없음

#### `__consumer_offset` 토픽
- 로그 컴팩션 기능을 사용하는 대표적 예제 (카프카의 내부 토픽)
- 해당 컨수머 그룹이 어디까지 읽었는지를 나타내는 오프셋 커밋 정보

![image](https://user-images.githubusercontent.com/20420626/199746453-252e033f-2109-4a57-905a-88f2f85b269d.png)
- 빠른 장애 복구
- 특정 상황에서만 사용 가능
- 로그 컴팩션 작업동안 브로커에는 과도한 I/O 부하가 발생할 수 있으니 브로커 모니터링을 병행해야 함

| 옵션 이름 | 옵션값 | 적용 범위 | 설명|
| --- | --- | --- | --- |
| `cleanup.policy`| `compact`| 토픽의 옵션으로 적용| 토픽 레벨에서 로그 컴팩션을 설명할 때 적용하는 옵션|
| `log.cleanup.policy`| `compact` | 브로커의설정 파일에 적용 |브로커 레벨에서 로그 컴팩션을 설정할 때 적용하는 옵션 |
| `log.cleaner.min.compaction.lag.ms`|0 |브로커의 설정 파일에 적용 |메세지가 기록된 후 컴팩션하기 전 경과되어야 할 최소 시간을 지정. 이 옵션을 설정하지 않으면 마지막 세그먼트를 제외하고 모든 세그먼트를 컴팩션 할 수 있음 |
| `log.cleaner.max.compaction.lag.ms`|9223372036854775807 |브로커의 설정 파일에 적용 |메세지가 기록된 후 컴팩션하기 전 경과되어야 할 최대 시간 | 
|`log.cleaner.min.cleanable.ratio`|0.5 |브로커의 설정 파일에 적용|로그에서 압축이 되지 않은 부분을 `dirty`라 표현. 전체 로그 대비 dirty의 비율이 50%가 넘으면 컴팩션이 실행됨|


## 스터디 회의록
> 책 내용에 오류가 있는것은 아닐지 확인 필요
- (p.119) 커밋되었다는 것은 리플리케이션 팩터 수의 모든 리플리케이션이 전부 메세지를 저장했음을 의미합니다.
  - -> `min.insync.replica` 이상의 ISR이 전부 메세지를 저장했음을 의미합니다. 
- (p.126~7) 예제는 `min.insync.replica = 1` 이기 때문에 3번처럼 팔로워의 요청 없이 리더만 데이터를 쓴 경우 하이워터마크를 올릴 수 있을 것 같다. 하지만 7번에서는 팔로워가 보낸 요청을 받은 리더가 하이워터마크를 2로 올린다는 설명이 있음. 그리고 8번에서는 `팔로워는 2번 오프셋인 message2`라고 되어있지만, message2의 오프셋은 1이기 때문에 수정이 필요해보인다.
- (p.134) 리더에포크는 새로운 리더 선출이 발생하면, 변경된 정보가 업데이트 됩니다.
  - -> 리더 브로커가 아닌 팔로워 브로커를 shutdown시켰을 때에도 리더에포크 정보가 업데이트됨

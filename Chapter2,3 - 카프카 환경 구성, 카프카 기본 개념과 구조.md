# 2. 카프카 환경구성

### AWS EC2

EC2 사용시 인스턴스를 바로 On/Off 해야 돈을 절약 할 수 있음

미사용 중인 AWS 리소스 일부를 사용하는 스팟 EC2 인스턴스의 가격이 온디맨드 인스턴스 비용보다 저렴함

(대신 On/Off 불가능)

### 서버구성

| AMI | Amazon Linux 2 AMI (Linux kernel 4.14) |
| --- | --- |
| Flavor | t2.medium * 6 / t2.small * 1(ansible)  |

### 클러스터 구성

배포 자동화 도구 중 하나인 앤서블을 사용

> 앤서블 공식 가이드 - [https://docs.ansible.com/ansible/latest/installation_buide/intro_installation.html](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
> 

| 호스트네임 | 서버 운영체제 | 서버사양 |
| --- | --- | --- |
| peter-zk01.foo.bar | 아마존 리눅스 2 | t2.medium(vCPU 2, MEM 4GB) |
| peter-zk02.foo.bar | “ | “ |
| peter-zk03.foo.bar | “ | “ |
| peter-kafka01.foo.bar | “ | " |
| peter-kafka02.foo.bar | “ | " |
| peter-kafka03.foo.bar | “ | " |

```bash
# ansible
# ansible 설치
sudo amazon-linux-extras install -y ansible2

# git 설치 & kafka 예제 다운로드
sudo yum install -y git
git clone https://github.com/onlybooks/kafka2

# key 복사
scp -i keypair.pem keypair.pem ec2-user@13.125.20.117:~

# key 등록
# 귀찮으면 공개 키 등록
chmod 600 keypair.pem
ssh-agent bash
ssh-add keypair.pem

# zookeeper 설치
cd kafka2/chapter2/ansible_playbook
ansible-playbook -i hosts zookeeper.yml

# sudo systemctl status zookeeper-server #프로세스 상태 확인
# sudo systemctl stop zookeeper-server #프로세스 중지
# sudo systemctl start zookeeper-server #프로세스 시작
# sudo systemctl restart zookeeper-server #프로세스 재시작

# kafka 설치
ansible-playbook -i hosts kafka.yml
```

- zookeeper.yml
    
    ```yaml
    ---
    - hosts: zkhosts
      become: true
      connection: ssh
      roles:
        - common
        - zookeeper
    ```
    
- kafka.yml
    
    ```yaml
    ---
    - hosts: kafkahosts
      become: true
      connection: ssh
      roles:
        - common
        - kafka
    ```
    

- 카프카 : 데이터를 받아서 전달하는 데이터 버스의 역할
- 프로듀서 : 데이터(메시지)를 만들어서 주는 역할
- 컨슈머 : 데이터(메시지)를 소비하는 역할
- 주키퍼 : 카프카의 정상 동작을 보장하기 위해 메타데이터를 관리하는 코디네이터

![2-1](https://user-images.githubusercontent.com/58289607/199203508-fd1e3214-78b4-48ba-a247-28b203311347.png)


<aside>
💡 카프카는 애플리케이션의 이름, 브로커는 카프카 애플리케이션이 설치된 서버 또는 노드

</aside>

```bash
# 토픽 생성
/usr/local/kafka/bin/kafka-topics.sh \
--bootstrap-server peter-kafka01.foo.bar:9092 \
--create --topic peter-overview01 \
--partitions 1 --replication-factor 3
```

```bash
# 컨슈머 실행
/usr/local/kafka/bin/kafka-console-consumer.sh \
--bootstrap-server peter-kafka01.foo.bar:9092 \
--topic peter-overview01
```

```bash
# 프로듀서 실행
/usr/local/kafka/bin/kafka-console-producer.sh \
--bootstrap-server peter-kafka01.foo.bar:9092 \
--topic peter-overview01
```

# 3. 카프카 기본 개념과 구조

- 분산 시스템
- 페이지 캐시
- 배치 전송
- 주키퍼의 역할

> **“카프카, 데이터 플랫폼의 최강자”**
> 

## 카프카 기초 다지기

---

### 카프카를 구성하는 주요 요소

| 주키퍼  | 메타데이터 관리 및 브로커의 정상상태 점검(health check) |
| --- | --- |
| 카프카/카프카 클러스터 | 여러대의 브로커를 구성한 클러스터 |
| 브로커  | 카프카 애플리케이션이 설치된 서버 또는 노드 |
| 프로듀서  | 카프카로 메시지를 보내는 역할 |
| 컨슈머  | 카프카에서 메시지를 꺼내는 역할 |
| 토픽  | 메시지 피드들을 토픽으로 구분, 고유한 토픽 이름을 가짐 |
| 파티션  | 병렬 처리 및 고성능을 얻기 위해 하나의 토픽을 여러 개로 나눈 것 |
| 세그먼트 | 실제 메시지가 브로커에 저장되는 파일 |
| 메시지/레코드 | 데이터 조각 |

### 리플리케이션

<aside>
💡 각 메시지들을 여러 개로 복제해서 카프카 클러스터 내 브로커들에 분산

</aside>

replication-factor : 카프카 내 몇 개의 리플리케이션을 유지

토픽이 리플리케이션이 되는것이 아닌 토픽의 파티션이 리플리케이션 되는것!

리플리케이션 팩터 수가 커지면 안정성은 높아지지만 그만큼 브로커 리소스를 많이 사용

| phase | replication-factor |
| --- | --- |
| 테스트나 개발 환경 | 1 |
| 운영 환경(로그 메시지로서 약간의 유실 허용) | 2 |
| 운영 환경(유실 허용하지 않음) | 3 |

리플리케이션 팩터 수가 3일 경우에 안정성 보장 / 적절한 디스크 공간

### 파티션

<aside>
💡 토픽 하나를 여러 개로 나눠 병렬 처리가 가능하게 만든 것을 파티션이라고 함.

</aside>

- 파티션 수도 토픽을 생성할 때 옵션으로 설정
- 파티션 수를 정하는 기준은 다소 모호함
    - 각 메시지 크기나 초당 메시지 건순 등에 따라 달라짐
- 파티션 수는 추기 생성 후 언제든지 늘릴 수 있지만, 반대로 한 번 늘린 파티션 수는 절대로 줄일 수 없음
    - 초기에 토픽을 생성할 때 파티션 수를 작게, 즉 2 또는 4 정도로 생성한 후, 메시지 처리량이나 컨슈머의 LAG 등을 모니터링하면서 조금씩 늘려가는 방법이 가장 좋음
- 컨슈머의 LAG
    
    $$
    LAG = Publish.message.length() - Subscribe.message.length()
    $$
    
- 파티션 수 가이드 사이트(https://eventsizer.io)
    
    ![3-1](https://user-images.githubusercontent.com/58289607/199203657-dc0ce412-0dce-4826-b5e0-6530e62bf819.png)


### 세그먼트

<aside>
💡 메시지들은 세그먼트라는 로그 파일의 형태로 브로커의 로컬 디스크에 저장

</aside>

![3-2](https://user-images.githubusercontent.com/58289607/199207096-91e0d71c-6577-4d8d-ba88-de866f0a62ea.png)

## 카프카의 핵심 개념

---

### 분산 시스템

최초 구성한 클러스터의 리소스가 한계치에 도달해 도욱 높은 메시지 처리량이 필요한 경우, 브로커를 추가하는 방식으로 확장이 가능

### 페이지 캐시

페이지 캐시는 직접 디스크에 읽고 쓰는 대신 물리 메모리 중 애플리케이션이 사용하지 않는 일부 잔여 메모리를 활용

![3-3](https://user-images.githubusercontent.com/58289607/199203787-c1b4470a-be7e-4bf4-a3c1-b210ee5283e8.png)

### 배치 전송 처리

수많은 통신을 묶어서 처리할 수 있다면, 단건으로 통신할 때에 비해 네트워크 오버헤드를 줄일 수 있을 뿐만 아니라 장기적으로는 더욱 빠르고 효율적으로 처리

![3-4](https://user-images.githubusercontent.com/58289607/199203835-6d708444-35f1-4545-91ee-1a612ea830f7.png)

- EX)
    - 상품의 재고 수량 업데이트 작업은 지연 없이 실스간으로 처리돼야 함
    - 구매 로그를 저장소로 보내는 작업은 이미 로그가 서버에 기록되어 있으므로 실시간보다는 배치처리가 효율

### 압축전송 (cpu)

메시지 전송 시 좀 더 성능이 높은 압축 전송을 사용하는 것을 권장

배치 전송과 결합해 사용하면 높은 효과를 얻을 수 있다.

지원타입 : gzip, snappy, lz4, zstd

높은 압축률이 필요 : gzip / zstd

빠른 응답속도 필요 : lz4 / snappy

### 토픽/파티션/오프셋

- 토픽 : 데이터를 저장하는 곳 (이메일의 주소 개념)
- 파티션 : 병렬 처리를 위해 여러 개의 파티션으로 나눔
- 오프셋 : 파티션의 메시지가 저장되는 위치 (64비트의 숫자로 순차적 증가)
    - 오프셋을 통해 메시지의 순서를 보장
    - 컨슈머에서는 마지막까지 읽은 위치를 알 수 있음

![3-5](https://user-images.githubusercontent.com/58289607/199203875-972538bb-4ada-445b-ae91-6d0145400ca9.jpg)

### 고가용성 보장

분산시스템이기 때문에 하나의 서버나 노드가 다운되어도 다른 서버 또는 노드가 장애가 발생한 서버의 역할을 대신해 안정적인 서비스 가능

- 카프카에서 제공하는 리플리케이션 기능은 토픽 자체를 복제하는 것이 아니라 토픽의 파티션을 복제하는 것
- master - 리더 / mirror - 팔로워
- 리더는 1을 유지한 채 리플리케이션 팩터 수에 따라 팔로워 수만 증가

### 주키퍼의 의존성

아파치 산하의 프로젝트인 Hadoop, NiFi, HBase 등 많은 분산 애플리케이션에서 코디네이터 역할

- 여러대의 서버를 앙상블로 구성하고, 살아 있는 노드 수가 과반수로 유지된다면 지속적인 서비스 가능
- 반드시 홀수로 구성해야 함
- 지노드를 이용해 카프카의 메타 정보가 주키퍼에 기록
- 지노드를 이용해 브로커의 노드 관리, 토픽 관리, 컨트롤러 관리등의 중요 역할

<aside>
💡 하지만 카프카의 지속적인 성장으로 주키퍼의 성능한계가 드러나기 시작→의존성을 제거하려는 움직임이 진행 중

</aside>

### 프로듀서 디자인

레코드

- 토픽
- 밸류
- [파티션] : 특정 파티션 지정
- [키] : 레코드들을 정렬하기 위한 키

![3-6](https://user-images.githubusercontent.com/58289607/199203939-b17cd7b0-fa64-4747-bf6b-e4a3bcce616b.png)

send() 메소드

시리얼라이저→파티셔너

- 파티션을 지정했으면 파티셔너는 아무 동작도 하지 않고 지정된 파티션으로 레코드를 전달
- 파티션을 지정하지 않으면 키를 가지고 파티션을 선택해 레코드 전달(default : RR)

send() 메소드 이후 레코드들을 파티션 별로 잠시 모아두었다가 배치 전송

실패시 재시도를 하며 지정된 횟수만큼 재시도 최종실패시 실패 리턴 성공시 메타데이터 리턴

### 프로듀서 옵션

| 옵션 | 설명 |
| --- | --- |
| bootstrap.servers | - 클라이언트가 카프카 클러스터에 처음 연결하기 위한 호스트와 포트 정보 |
| client.dns.lookup | - 하나의 호스트에 여러 IP를 매핑해 사용하는 일부 환경에서 클라이언트가 하나의 IP와 연결하지 못할 경우에 다른 IP로 시도하는 설정<br> - use_all_dns_ips가 기본값<br> - resolve_canonical_bootstrap_servers_only는 커버로스 FQDN을 얻기위함 |
| acks | - 프로듀서가 카프카 토픽의 리더 측에 메시지를 전송한 후 요청을 완료하기를 결정하는 옵션<br> - 0 : 빠른전송, 일부메시지 손실 가능성<br> - 1 : 리더가 메시지를 받았는지 확인<br> - all(-1): 팔로워가 메시지를 받았는지 여부 확인, 다소 느림, 하나의 팔로워가 있는 한 메시지는 손실되지 않음 |
| buffer.memory | - 잠시대기 할 수 있는 전체 메모리 바이트 |
| compression.type | - 메시지 전송시 압축 타입 (none, gzip, snappy, lz4, zstd) |
| enable.idempotence | - 중복없는 전송 가능<br> - max.flight.requests.per.connection ≤ 5 && retries ≥ 0 && acks == all |
| max.in.flight.requests.per.connection | - 하나의 커넥션에서 프로듀서가 최대한 ACK 없이 전송할 수 있는 요청수 |
| retries | - 일시적인 오류로 인해 전송에 실패한 데이터를 다시 보내는 횟수 |
| batch.size | - 배치 작업 사이즈 |
| linger.ms | - 배치 크기에 도달하지 못한 상황에서 linger.ms 제한 시간에 도달했을 때 메시지를 전송 |
| transactional.id | - 정확히 한 번 전송을 위해 사용하는 옵션<br> - 옵션을 사용하기 전 enable.idempotence를 true로 (자동으로)설정<br> - 보낼때마다 |

<aside>
💡 FQDN은 '절대 도메인 네임' 또는 '전체 도메인 네임' 이라고도 불리는 도메인 전체 이름을 표기하는 방식을 의미한다.

</aside>

<aside>
💡 FQDN와 달리 전체 경로명이 아닌 하위 일부 경로만으로 식별 가능하게 하는 이름을 PQDN(Partially~)라 한다.

</aside>

### 컨슈머의 기본 동작

- 컨슈머 그룹은 하나 이상의 컨슈머들이 모여 있는 그룹을 의미, 컨슈머는 반드시 컨슈머 그룹에 속함
- 파티션 수와 컨슈머 수(하나의 컨슈머 그룹 안에 있는 컨슈머 수)는 일대일로 매핑되는 것이 이상적
- 파티션 수보다 컨슈머 수가 많게 구현하는 것은 바람직한 구성이 아님!
    - 컨슈머 수가 파티션 수보다 많다고 해서 토픽 메시지를 더 빠르게 가져오는게 아님
    - 그냥 대기상태로 존재
- 장애 발생시 컨슈머 그룹내에서 리밸런싱을 통해 역할이 대체, 추가 컨슈머 리소스를 할당하지 않아도 됨

### 컨슈머의 주요 옵션

| 옵션 | 설명 |
| --- | --- |
| bootstrap.servers | - 프로듀서와 동일하게 프로커의 정보를 입력 |
| fetch.min.bytes | - 한 번에 가져올 수 있는 최소 데이터 크기<br> - 만약 지정한 크기보다 작은 경우, 요청에 응답하지 않고 누적 될 때까지 기다림 |
| fetch.max.bytes | - 한 번의 가져오기 요청으로 가져올 수 있는 최대 크기<br> - default : 50MB |
| max.partition.fetch.bytes | - 파티션당 가져올 수 있는 최대 크기<br> - default : 1MB |
| session.timeout.ms | - 컨슈머가 종료된 것인지 판단<br> - 하트비트를 보내지 않았다면 해당 컨슈머는 종료된것으로 간주하고 그룹에서 제외 후 리밸런싱 |
| heartbeat.interval.ms | - active 상태 체크<br> - session.timeout.ms 보다 낮은값으로 설정<br> - 일반적으로 session.timeout.ms의 1/3으로 설정 |
| group.id | - 컨슈머가 속한 컨슈머 그룹을 식별하는 식별자 |
| group.instance.id | - 컨슈머의 고유한 식별자<br> - 설정시 static 멤버로 간주, 불필요한 리밸런싱 지양 |
| enable.auto.commit | - 백그라운드로 주기적으로 오프셋 커밋 |
| auto.offset.reset | - 현재 오프셋이 더 이상 존재하지 않는 경우 다음 옵션으로 reset<br> - consumer group 새로 만들때 / 없어졌을때<br> - earliest : 가장 초기의 오프셋값으로 설정<br> - latest : 가장 마지막의 오프셋값으로 설정<br> - none : 이전 오프셋값을 찾지 못하면 에러 |
| isolation.level | - 트랙잭션 컨슈머에서 사용하는 옵션<br> - read_uncommitted : 기본값으로 모든 메시지를 읽음<br> - read_committed : 트랜잭션이 완료된 메시지만 읽음 |
| max.poll.records | 한번의 poll() 요청으로 가져오는 최대 메시지 수 |
| partition.assignment.strategy | - 파티션 할당 전략이며<br> - 기본 값은 range<br> - RR / Sticky<br> - https://velog.io/@hyun6ik/Apache-Kafka-Partition-Assignment-Strategy |
| fetch.max.wait.ms | - fetch.min.bytes에 의해 설정된 데이터보다 적은 경우 요청에 대한 응답시간을 기다리는 최대 시간 |

<aside>
💡 RangeAssignor : Topic 별로 작동하는 Default Assignor

</aside>

<aside>
💡 RoundRobinAssginor : Round Robin 방식으로 Consumer에게 Partition을 할당한다.

</aside>

<aside>
💡 StickyAssignor : 최대한 많은 기존 Partition 할당을 유지하면서 최대 균형을 이루는 할당을 보장한다.

</aside>

### 컨슈머 그룹의 이해

컨슈머는 컨슈머 그룹 안에 속한 것이 일반적인 구조, 하나의 컨슈머 그룹 안에 여러 개의 컨슈머가 구성

파티션과 일대일로 매핑되어 메시지를 가져옴
![3-7](https://user-images.githubusercontent.com/58289607/199203992-bafddfb2-da8e-40c0-9225-0f88115452fb.png)

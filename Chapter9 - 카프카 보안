# 카프카 보안

- 카프카는 대다수 기업의 중앙 데이터 파이프라인 허브로서 역할을 수행
- 주로 내부 시스템으로 활용하기 때문에 대부분 사내 보안에 대해서는 큰 신경을 쓰지 않음
- 래빗MQ는 자체적인 보안 기능을 초기 구성부터 즉시 적용할 수 있으나, 카프카는 관리자가 직접 적용해야 하고 웹 인터페이스도 없어서 불편함

## 9.1 카프카 보안의 세 가지 요소
- 설치 과정에서 기본적으로 보안이 적용되지 않음
    - 어떤 클라이언트이든지 카프카와 통신만 가능하다면 자유롭게 카프카와 연결 가능
- 사내 환경에 적용된 경우가 많다 보니 보안이 적용되어 있지 않은 경우가 대부분
- 세 가지의 보안 요소가 필요
    1. 암호화
        - 인증, 권한 설정을 모두 적용했음에도 누군가 악의적인 목적으로 중간에서 데이터를 가로챌 수 있음
        - ex) 웹에서 http 대신 https를 적용
    2. 인증
        - 안전하다고 확인된 클라이언트만 접근할 수 있도록 설정
        - ex) 웹에서 로그인된 사용자만 페이지에 접근할 수 있도록 함
    3. 권한
        - 인증받은 클라이언트라고 하더라도 권한을 서로 다르게 부여함
        - ex) 웹에서 일반 사용자, 관리자로 로그인했을 때 접근할 수 있는 페이지를 구분함

### 9.1.1 암호화(SSL)
- 카프카와 카프카 클라이언트 사이의 통신 내용을 중간에 가로채는 데 성공하더라도 내용을 전혀 파악할 수 없도록 암호화 적용
- 일반적으로 SSL(Secure Socket Layer) 사용
- SSL 동작
    - 인증기관(Certificate Authority, CA)에서 인증서 발급
    - 인증서를 이용한 퍼블릭 키(공개키) 프라이빗 키(비밀키) 방식으로 서버와 클라이언트가 암/복호화를 하면서 통신
    - 암/복호화를 위해 서버와 클라이언트는 미리 지정된 키를 주고받아야 함
    - SSL은 대칭키, 비대칭키 두 방식을 모두 혼용하여서 효율성 및 성능을 강화함
- https://opentutorials.org/course/228/4894

### 9.1.2 인증(SASL)
- SASL(Simple Authentication and Security Layer)
    - 인터넷 프로토콜에서 인증과 데이터 보안을 위한 프레임워크
- SASL 메커니즘 중 다음과 같은 네 가지 메커니즘을 지원함
    - SASL/GSSAPI
        - 0.9 버전부터 지원
        - 커버로스 인증 방식
        - 커버로스 설정 중 렐름(Realm)은 되도록 하나의 렐름으로 모든 애플리케이션을 적용
            - 크로스 렐름 설정으로 인해 클라이언트들의 재인증, 인증 실패가 종종 발생할 수 있음
    - SASL/PLAIN
        - 0.10.0 버전부터 지원
        - 아이디와 비밀번호를 텍스트 형태로 사용
        - 운영 환경보다는 개발 환경에서 테스트할 목적으로 활용
    - SASL/SCRAM-SHA-256, SASL/SCRAM-SHA-512
        - SCRAM(Salted Challenge Response Authentication Mechanism)
            - 본래의 암호에 해시된 내용을 추가해서(Salted) 암호가 유출되어도 본래의 암호를 알 수 없게 함
        - 인증 정보를 주키퍼에 저장해 사용하는 방식이고 토큰 방식도 지원
        - 커버로스 서버가 구성되어 있지 않는 환경에서는 가장 좋은 인증 방식
    - SASL/OAUTHBEARER
        - 2.0 버전부터 지원
        - 최근 인증에서 많이 사용되는 방식이지만 카프카에서 제공하는 기능은 매우 한정적
        - 운영 환경에서 사용하기 위해서는 별도의 핸들러 구성 필요
        - 개발환경에만 적용 가능한 수준
### 9.1.3 권한(ACL)
- ACL(Access Control List)이라는 규칙 기반의 리스트를 만들어 접근 제어
- 모든 사용자에게 동일한 권한을 부여한다면 불필요한 커뮤니케이션을 줄일 수 있음
- 담당자의 실수로 인한 상황을 막지 못함
    - A 서비스 에서는 A 토픽, B 서비스에서는 B 토픽을 읽어들여야 하는 상황
    - 실수로 A 서비스에서 B 토픽을 읽게 된다면 오류가 발생하거나 예기치 않은 종료를 맞을 수 있음
- 권한관리를 위해 ACL을 사용하며, CLI로 ACL을 추가하거나 삭제할 수 있음
- 모든 ACL 규칙은 주키퍼에 저장
- ACL은 리소스 타입별로 구체적인 설정이 가능
    - 토픽
    - 그룹
    - 클러스터
    - 트랜잭셔널 ID
    - 위임 토큰
- 9.4.2 예제 제공

## 9.2 SSL을 이용한 카프카 암호화
- 자바 기반 애플리케이션에서는 키스토어(Keystore)라는 인터페이스를 통해 퍼블릭키, 프라이빗키, 인증서를 추상화해 제공
- 카프카도 자바 기반 애플리케이션이므로 자바의 `keytool` 이라는 명령어를 이용해 카프카에 SSL 적용
    - `$JAVA_HOME/bin/keytool`

![image.png](/files/3427900682757097035)
- 책에서는 클러스터 환경을 기준으로 적용 설명

### 9.2.1 브로커 키스토어 생성
![image.png](/files/3429250163971790703)
- 키스토어, 트러스트스토어 모두 `keytool`을 이용해 관리
- 키스토어
    - 일반적으로 서버 측면
    - 프라이빗키와 인증서를 저장
    - 자격 증명을 제공
- 트러스트스토어
    - 일반적으로 클라이언트 측면
    - 서버가 제공하는 인증서를 검증하기 위한 퍼블릭키 저장
    - 서버와 SSL 연결에서 유효성을 검사하는 서명된 인증서를 저장
    - 민감정보는 저장하지 않음

- 앞으로의 SSL 적용 예제에서 사용할 `ssl` 디렉터리 생성 및 초기 설정 진행
- `ssl` 디렉터리 생성
    ```sh
    ubuntu@jh-01:~/apps$ mkdir ssl
    ubuntu@jh-01:~/apps$ cd ssl
    ubuntu@jh-01:~/apps/ssl$ pwd
    /home/ubuntu/apps/ssl
    ```
- 비밀번호 환경변수 설정
    - 모든 비밀번호를 통일해서 사용하는 것을 가정
    - 실제 운영 환경에서는 필요에 따라 다르게 설정할 것을 권장
    ```sh
    ubuntu@jh-01:~/apps/ssl$ export SSLPASS=jhkimpass
    ```
- 키스토어 생성
    - 키스토어 생성은 **모든 브로커**에서 진행하고, 도메인명은 서버에 따라 변경
    - CN은 서버에 따라 변경
    ```sh
    ubuntu@jh-01:~/apps/ssl$ keytool -keystore kafka.server.keystore.jks -alias localhost -keyalg RSA -validity 365 -genkey -storepass $SSLPASS -keypass $SSLPASS -dname "CN=jhkim-kafka01.foo.bar" -storetype pkcs12
    ```
    ```sh
    ubuntu@jh-01:~/apps/ssl$ ls -al
    total 12
    drwxrwxr-x 2 ubuntu ubuntu 4096 Dec 13 15:09 .
    drwxrwxr-x 6 ubuntu ubuntu 4096 Dec 13 15:01 ..
    -rw-rw-r-- 1 ubuntu ubuntu 2575 Dec 13 15:09 kafka.server.keystore.jks
    ```
    - 옵션 설명
        - keystore
            - 키스토어 이름
        - alias
            - 별칭
        - keyalg
            - 키 알고리즘
        - genkey
            - 키 생성
        - validity
            - 유효 일자
        - storepass
            - 저장소 비밀번호
        - keypass
            - 키 비밀번호
        - dname
            - 식별 이름
        - storetype
            - 저장 타입
- 키스토어 정보 확인
    ```
    ubuntu@jh-01:~/apps/ssl$ keytool -list -v -keystore kafka.server.keystore.jks
    Enter keystore password:
    Keystore type: PKCS12
    Keystore provider: SUN

    Your keystore contains 1 entry

    Alias name: localhost
    Creation date: Dec 13, 2022
    Entry type: PrivateKeyEntry
    Certificate chain length: 1
    Certificate[1]:
    Owner: CN=jhkim-kafka01.foo.bar
    Issuer: CN=jhkim-kafka01.foo.bar
    Serial number: 279bdd55
    Valid from: Tue Dec 13 15:15:44 UTC 2022 until: Wed Dec 13 15:15:44 UTC 2023
    Certificate fingerprints:
         SHA1: 21:AD:7B:45:B4:0D:45:CD:F6:9B:24:04:C4:27:8C:0A:23:D7:02:42
         SHA256: 40:42:40:14:0C:98:BE:CB:1F:83:A9:53:65:4B:87:B6:E5:16:91:F4:A8:E5:57:E3:3E:20:07:B4:42:46:7D:EA
    Signature algorithm name: SHA256withRSA
    Subject Public Key Algorithm: 2048-bit RSA key
    Version: 3

    Extensions:

    #1: ObjectId: 2.5.29.14 Criticality=false
    SubjectKeyIdentifier [
    KeyIdentifier [
    0000: C1 59 AF F9 DC 44 34 DC   4C 0C 4B FC 00 16 99 82  .Y...D4.L.K.....
    0010: C1 11 17 FD                                        ....
    ]
    ]



    *******************************************
    *******************************************
    ```

- 이후 CA 인증서, 트러스트스토어 생성은 1번 서버에서 진행한다고 가정
### 9.2.2 CA 인증서 생성
- 공인 인증서를 사용하는 이유는 위조된 인증서를 방지해 클라이언트-서버 간 안전한 통신을 하기 위함
- 인증기관(CA)의 역할
    - 정부가 인증된 여권을 발급해주고, 이 인증받은 여권을 이용해 다른 나라에서도 인증받을 수 있음
    - 정부와 비슷한 역할을 하는 것이 CA
    - CA가 인증서를 서명하면 공인 인증된 자격을 갖게 되고 클라이언트들이 신뢰할 수 있다는 사실을 입증
- 일반적으로는 CA에 비용을 지급하고 인증서를 발급받음
- 테스트나 개발환경에서는 자체 서명된(self-signed) CA 인증서나 사설 인증서를 생성하기도 함
- 책에서는 자체 서명된 인증서로 테스트
- CA 인증서 생성
    ```
    ubuntu@jh-02:~/apps/ssl$ openssl req -new -x509 -keyout ca-key -out ca-cert -days 365 -subj "/CN=foo.bar" -nodes
    ```
    ```
    ubuntu@jh-01:~/apps/ssl$ ls -al
    total 20
    drwxrwxr-x 2 ubuntu ubuntu 4096 Dec 13 16:11 .
    drwxrwxr-x 6 ubuntu ubuntu 4096 Dec 13 15:01 ..
    -rw-rw-r-- 1 ubuntu ubuntu 1107 Dec 13 16:11 ca-cert
    -rw------- 1 ubuntu ubuntu 1704 Dec 13 16:11 ca-key
    -rw-rw-r-- 1 ubuntu ubuntu 2575 Dec 13 15:15 kafka.server.keystore.jks
    ```
    - 옵션 설명
        - new
            - 새로 생성 요청
        - x509
            - 표준 인증서 번호
        - keyout
            - 생성할 키 파일 이름
        - out
            - 생성할 인증서 파일 이름
        - days
            - 유효 일자
        - subj
            - 인증서 제목
        - nodes
            - 프라이빗 키 파일을 암호화하지 않음

### 9.2.3 트러스트스토어 생성
- 일반적으로 서버가 공인 인증서를 발급받은 경우 클라이언트는 서버 측으로 접속해서 서버가 갖고 있는 인증서가 신뢰할 수 있는 것인지 여부를 확인
- 사설(자체 서명) 인증서는 CA를 통해 발급받지 않았으므로, 서버에 접속한 클라이언트는 이 인증서를 신뢰할 수 없음
- 앞서 생성한 자체 서명된 인증서를 클라이언트가 신뢰할 수 있도록 트러스트스토어에 추가
    ```sh
    ubuntu@jh-01:~/apps/ssl$ keytool -keystore kafka.server.truststore.jks -alias CARoot -importcert -file ca-cert -storepass $SSLPASS -keypass $SSLPASS
    ...

    Trust this certificate? [no]:  yes
    Certificate was added to keystore
    ```
    - 옵션 설명
        - keytool
            - 키스토어 이름
        - alias
            - 별칭
        - importcert
            - 인증서를 임포트
        - file
            - 인증서 파일
        - storepass
            - 저장소 비밀번호
        - keypass
            - 키 비밀번호
    ```sh
    ubuntu@jh-01:~/apps/ssl$ ls -al
    total 24
    drwxrwxr-x 2 ubuntu ubuntu 4096 Dec 13 16:27 .
    drwxrwxr-x 6 ubuntu ubuntu 4096 Dec 13 15:01 ..
    -rw-rw-r-- 1 ubuntu ubuntu 1107 Dec 13 16:11 ca-cert
    -rw------- 1 ubuntu ubuntu 1704 Dec 13 16:11 ca-key
    -rw-rw-r-- 1 ubuntu ubuntu 2575 Dec 13 16:27 kafka.server.keystore.jks
    -rw-rw-r-- 1 ubuntu ubuntu 1143 Dec 13 16:27 kafka.server.truststore.jks
    ```
- 트러스트스토어 내용 확인
    - CN 정보, 유효기간 확인
    ```sh
    ubuntu@jh-01:~/apps/ssl$ keytool -list -v -keystore kafka.server.truststore.jks
    Enter keystore password:
    Keystore type: PKCS12
    Keystore provider: SUN

    Your keystore contains 1 entry

    Alias name: caroot
    Creation date: Dec 13, 2022
    Entry type: trustedCertEntry

    Owner: CN=foo.bar
    Issuer: CN=foo.bar
    Serial number: 176950602fe5127befc014730bbdac046a1481f6
    Valid from: Tue Dec 13 16:11:12 UTC 2022 until: Wed Dec 13 16:11:12 UTC 2023
    Certificate fingerprints:
         SHA1: 73:DD:CE:34:01:26:91:03:44:51:53:3C:A2:18:EE:AE:A5:D7:4F:6A
         SHA256: 51:65:0F:D9:78:E8:2A:B0:1C:22:0A:3E:B1:33:A6:D1:4E:9C:FF:08:90:76:A8:BC:13:B2:8D:9B:67:AA:D3:B7
    Signature algorithm name: SHA256withRSA
    Subject Public Key Algorithm: 2048-bit RSA key
    Version: 3

    Extensions:

    #1: ObjectId: 2.5.29.35 Criticality=false
    AuthorityKeyIdentifier [
    KeyIdentifier [
    0000: E4 51 68 DA 03 1B AD 66   D2 51 D3 1C 53 AA 13 D8  .Qh....f.Q..S...
    0010: 7C 1B 97 98                                        ....
    ]
    ]

    #2: ObjectId: 2.5.29.19 Criticality=true
    BasicConstraints:[
      CA:true
      PathLen:2147483647
    ]

    #3: ObjectId: 2.5.29.14 Criticality=false
    SubjectKeyIdentifier [
    KeyIdentifier [
    0000: E4 51 68 DA 03 1B AD 66   D2 51 D3 1C 53 AA 13 D8  .Qh....f.Q..S...
    0010: 7C 1B 97 98                                        ....
    ]
    ]



    *******************************************
    *******************************************
    ```

### 9.2.4 인증서 서명
- 키스토어에 저장된 모든 인증서는 자체 서명된 CA의 서명을 받아야 함
- 키스토어에서 인증서 추출
    ```sh
    ubuntu@jh-01:~/apps/ssl$ keytool -keystore kafka.server.keystore.jks -alias localhost -certreq -file cert-file -storepass $SSLPASS -keypass $SSLPASS
    ```
    ```sh
    ubuntu@jh-01:~/apps/ssl$ ls -al
    total 28
    drwxrwxr-x 2 ubuntu ubuntu 4096 Dec 13 16:33 .
    drwxrwxr-x 6 ubuntu ubuntu 4096 Dec 13 15:01 ..
    -rw-rw-r-- 1 ubuntu ubuntu 1107 Dec 13 16:11 ca-cert
    -rw------- 1 ubuntu ubuntu 1704 Dec 13 16:11 ca-key
    -rw-rw-r-- 1 ubuntu ubuntu  993 Dec 13 16:33 cert-file
    -rw-rw-r-- 1 ubuntu ubuntu 2575 Dec 13 16:27 kafka.server.keystore.jks
    -rw-rw-r-- 1 ubuntu ubuntu 1143 Dec 13 16:27 kafka.server.truststore.jks
    ```
- 인증서 파일(cert-file)에 대해 자체 서명된 CA 서명 적용
    ```
    ubuntu@jh-01:~/apps/ssl$ openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:$SSLPASS
    Certificate request self-signature ok
    subject=CN = jhkim-kafka01.foo.bar
    ```
    - 옵션 설명
        - x509
            - 표준 인증서 번호
        - req
            - 인증서 서명 요청
        - ca
            - 인증서 파일
        - cakey
            - 프라이빗 키 파일
        - in
            - 인풋 파일
        - out
            - 아웃풋 파일
        - days
            - 유효 일자
        - passin
            - 소스의 프라이빗 키 비밀번호
    ```sh
    ubuntu@jh-01:~/apps/ssl$ ls -al
    total 32
    drwxrwxr-x 2 ubuntu ubuntu 4096 Dec 13 16:45 .
    drwxrwxr-x 6 ubuntu ubuntu 4096 Dec 13 15:01 ..
    -rw-rw-r-- 1 ubuntu ubuntu 1107 Dec 13 16:11 ca-cert
    -rw------- 1 ubuntu ubuntu 1704 Dec 13 16:11 ca-key
    -rw-rw-r-- 1 ubuntu ubuntu  993 Dec 13 16:33 cert-file
    -rw-rw-r-- 1 ubuntu ubuntu 1005 Dec 13 16:45 cert-signed
    -rw-rw-r-- 1 ubuntu ubuntu 2575 Dec 13 16:27 kafka.server.keystore.jks
    -rw-rw-r-- 1 ubuntu ubuntu 1143 Dec 13 16:27 kafka.server.truststore.jks
    ```
- 키스토어에 자체 서명된 CA 인증서인 ca-cert와 서명된 cert-signed 추가
    ```sh
    ubuntu@jh-01:~/apps/ssl$ keytool -keystore kafka.server.keystore.jks -alias CARoot -importcert -file ca-cert -storepass $SSLPASS -keypass $SSLPASS
    ...
    
    Trust this certificate? [no]:  yes
    Certificate was added to keystore
    ```
- 키스토어 내용 확인
    - 2개 인증서 저장 확인
    ```sh
    ubuntu@jh-01:~/apps/ssl$ keytool -list -v -keystore kafka.server.keystore.jks
    Enter keystore password:
    Keystore type: PKCS12
    Keystore provider: SUN

    Your keystore contains 2 entries

    Alias name: caroot
    Creation date: Dec 13, 2022
    Entry type: trustedCertEntry

    Owner: CN=foo.bar
    Issuer: CN=foo.bar
    Serial number: 176950602fe5127befc014730bbdac046a1481f6
    Valid from: Tue Dec 13 16:11:12 UTC 2022 until: Wed Dec 13 16:11:12 UTC 2023
    Certificate fingerprints:
         SHA1: 73:DD:CE:34:01:26:91:03:44:51:53:3C:A2:18:EE:AE:A5:D7:4F:6A
         SHA256: 51:65:0F:D9:78:E8:2A:B0:1C:22:0A:3E:B1:33:A6:D1:4E:9C:FF:08:90:76:A8:BC:13:B2:8D:9B:67:AA:D3:B7
    Signature algorithm name: SHA256withRSA
    Subject Public Key Algorithm: 2048-bit RSA key
    Version: 3

    Extensions:

    #1: ObjectId: 2.5.29.35 Criticality=false
    AuthorityKeyIdentifier [
    KeyIdentifier [
    0000: E4 51 68 DA 03 1B AD 66   D2 51 D3 1C 53 AA 13 D8  .Qh....f.Q..S...
    0010: 7C 1B 97 98                                        ....
    ]
    ]

    #2: ObjectId: 2.5.29.19 Criticality=true
    BasicConstraints:[
      CA:true
      PathLen:2147483647
    ]

    #3: ObjectId: 2.5.29.14 Criticality=false
    SubjectKeyIdentifier [
    KeyIdentifier [
    0000: E4 51 68 DA 03 1B AD 66   D2 51 D3 1C 53 AA 13 D8  .Qh....f.Q..S...
    0010: 7C 1B 97 98                                        ....
    ]
    ]



    *******************************************
    *******************************************


    Alias name: localhost
    Creation date: Dec 13, 2022
    Entry type: PrivateKeyEntry
    Certificate chain length: 1
    Certificate[1]:
    Owner: CN=jhkim-kafka01.foo.bar
    Issuer: CN=jhkim-kafka01.foo.bar
    Serial number: 3abac3b9
    Valid from: Tue Dec 13 16:27:25 UTC 2022 until: Wed Dec 13 16:27:25 UTC 2023
    Certificate fingerprints:
         SHA1: 69:3E:48:6F:94:CD:FD:A2:91:D3:A6:06:79:29:F6:C6:70:E1:56:67
         SHA256: 70:7F:C8:83:49:75:CA:85:C3:E5:3B:62:21:F9:91:E4:46:81:59:F9:60:4F:8B:74:72:92:0C:47:90:B3:5F:A5
    Signature algorithm name: SHA256withRSA
    Subject Public Key Algorithm: 2048-bit RSA key
    Version: 3

    Extensions:

    #1: ObjectId: 2.5.29.14 Criticality=false
    SubjectKeyIdentifier [
    KeyIdentifier [
    0000: CE 1B C8 8C 3C 06 5E 04   2D BA 5C 76 AE 01 74 6D  ....<.^.-.\v..tm
    0010: 2F 94 E5 9A                                        /...
    ]
    ]



    *******************************************
    *******************************************
    ```

### 9.2.5 나머지 브로커에 대한 SSL 구성
- CA 인증서, 키 파일, 트러스트스토어를 생성한 서버에서 나머지 브로커에 복사
    - ca-cert, ca-key, kafka.server.truststore.jks
- 나머지 브로커에서 키스토어 서명 작업 진행
    ```sh
    ubuntu@jh-02:~/apps/ssl$ keytool -keystore kafka.server.keystore.jks -alias localhost -certreq -file cert-file -storepass $SSLPASS -keypass $SSLPASS
    ubuntu@jh-02:~/apps/ssl$ openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:$SSLPASS
    Certificate request self-signature ok
    subject=CN = jhkim-kafka01.foo.bar
    ```
- 나머지 브로커에서 키스토어에 CA 인증서 추가 진행
    ```sh
    ubuntu@jh-02:~/apps/ssl$ keytool -keystore kafka.server.keystore.jks -alias CARoot -importcert -file ca-cert -storepass $SSLPASS -keypass $SSLPASS
    ...
    Trust this certificate? [no]:  yes
    Certificate was added to keystore
    ```
    ```sh
    ubuntu@jh-02:~/apps/ssl$ keytool -keystore kafka.server.keystore.jks -alias localhost -importcert -file cert-signed -storepass $SSLPASS -keypass $SSLPASS
    Certificate reply was installed in keystore
    ```
- 키스토어 내용 확인
    - 9.2.4에서와 비슷하게 2개 인증서 저장 확인
    ```sh
    ubuntu@jh-02:~/apps/ssl$ keytool -list -v -keystore kafka.server.keystore.jks
    Enter keystore password:
    Keystore type: PKCS12
    Keystore provider: SUN
    ...
    ```
### 9.2.6 브로커 설정에 SSL 추가
- server.properties 설정 변경
    - `listeners`, `advertised.listeners`, `security.inter.broker.protocol`
    - 기존 통신 방식인 `PLAINTEXT://` 는 남겨두고 `SSL://` 을 추가
    - 기존의 평문 통신과 SSL 통신 모두 가능
    ```properties
    listeners=PLAINTEXT://192.168.1.18:9092,SSL://192.168.1.18:9093

    advertised.listeners=PLAINTEXT://211.37.148.236:9092,SSL://jhkim-kafka01.foo.bar:9093

    ssl.truststore.location=/home/ubuntu/apps/ssl/kafka.server.truststore.jks
    ssl.truststore.password=jhkimpass
    ssl.keystore.location=/home/ubuntu/apps/ssl/kafka.server.keystore.jks
    ssl.keystore.password=jhkimpass
    ssl.key.password=jhkimpass
    security.inter.broker.protocol=SSL
    ```
- SSL 적용 시 암/복호화로 인한 오버헤드 발생
    - `security.inter.broker.protocol` 은 브로커 간 통신(내부 통신)에 대한 암호화 여부
    - 브로커가 모두 신뢰할 수 있는 네트워크 공간에 존재한다면 내부 통신은 평문으로 설정하여 오버헤드 줄일 수 있음
### 9.2.7 SSL 기반 메시지 전송
- 클라이언트용 트러스트스토어 생성
    - 9.2.3과 동일
    ```sh
    ubuntu@jh-01:~/apps/ssl$ keytool -keystore kafka.server.truststore.jks -alias CARoot -importcert -file ca-cert -storepass $SSLPASS -keypass $SSLPASS
    ...

    Trust this certificate? [no]:  yes
    Certificate was added to keystore
    ```
- 토픽 생성
    ```sh
    ubuntu@jh-01:~/apps/kafka$ bin/kafka-topics.sh --bootstrap-server jhkim-kafka01.foo.bar:9092 --create --topic jhkim-test07 --partitions 1 --replication-factor 3

    Created topic jhkim-test07
    ```
- SSL 통신이 적용된 콘솔 프로듀서를 사용하기 위해, 별도의 클라이언트 설정 파일 생성
- ssl.config 파일 생성
    ```properties
    security.protocol=SSL
    ssl.truststore.location=/home/ubuntu/apps/ssl/kafka.client.truststore.jks
    ssl.truststore.password=jhkimpass
    ```
- 콘솔 프로듀서 실행 및 메시지 전송
    ```sh
    ubuntu@jh-01:~/apps/kafka$ bin/kafka-console-producer.sh --bootstrap-server jhkim-kafka01.foo.bar:9092 --topic jhkim-test07 --producer.config ssl.config
    > hello SSL message
    ```
- 콘솔 컨슈머 실행
    ```sh
    ubuntu@jh-01:~/apps/kafka$ bin/kafka-console-consumer.sh --bootstrap-server jhkim-kafka01.foo.bar:9092 --topic jhkim-test07 --from-beginning --consumer.config ssl.config

    hello SSL message
    ```

## 9.3 커버로스(SASL)를 이용한 카프카 인증
- SASL 메커니즘 중 가장 흔히 사용됨
- ![image.png](/files/3429124658941019663)
    - 카프카 클라이언트는 인증/티켓 서버(커버로스 서버)로부터 인증을 확인받고 완료되면 티켓 발급받음
    - 클라이언트는 발급받은 티켓을 가지고 카프카 서버에 인증할 수 있음
    - 별도의 커버로스 서버 필요

### 9.3.1 커버로스 구성

- 커버로스 설치
    - Chapter2의 앤서블 참조
    - https://github.com/onlybooks/kafka2/tree/main/chapter2/ansible_playbook
        ```
        $ ansible-playbook -i hosts kerberos.yml
        ```
- 디폴트 렐름(REALM)은 `FOO.BAR`
- 프린시펄(principal)은 커버로스에서 인증 대상이 되는 개체(계정과 유사)
- 프린시펄을 생성하고, 해당 프린시펄의 키탭(keytab)을 발급받아서 인증에 사용
    - 키탭이 있는 경우 비밀번호를 별도로 입력하지 않아도 됨
- 프린시펄 생성
    ```sh
    $ kadmin.local -q "add_principal -randkey peter01@FOO.BAR"
    $ kadmin.local -q "add_principal -randkey peter02@FOO.BAR"
    $ kadmin.local -q "add_principal -randkey admin@FOO.BAR"
    ```
- kafka 서비스를 위한 프린시펄 생성
    ```sh
    $ kadmin.local -q "add_principal -randkey kafka/peter-kafka01.foo.bar@FOO.BAR"
    $ kadmin.local -q "add_principal -randkey kafka/peter-kafka02.foo.bar@FOO.BAR"
    $ kadmin.local -q "add_principal -randkey kafka/peter-kafka03.foo.bar@FOO.BAR"
    ```
- 키탭 생성
    ```sh
    kadmin.local -q "ktadd -k /home/ubuntu/keytabs/peter01.user.keytab peter01@FOO.BAR"
    kadmin.local -q "ktadd -k /home/ubuntu/keytabs/peter02.user.keytab peter02@FOO.BAR"
    kadmin.local -q "ktadd -k /home/ubuntu/keytabs/admin.user.keytab admin@FOO.BAR"
    kadmin.local -q "ktadd -k /home/ubuntu/keytabs/peter-kafka01.service.keytab kafka/peter-kafka01.foo.bar@FOO.BAR"
    kadmin.local -q "ktadd -k /home/ubuntu/keytabs/peter-kafka02.service.keytab kafka/peter-kafka02.foo.bar@FOO.BAR"
    kadmin.local -q "ktadd -k /home/ubuntu/keytabs/peter-kafka03.service.keytab kafka/peter-kafka03.foo.bar@FOO.BAR"
    ```
- 각 브로커로 키탭 파일 복사

### 9.3.2 키탭을 이용한 인증
- 모든 브로커 서버의 `/etc/krb5.conf`에 렐름 정보 추가
- 앤서블로 설치한 경우 이미 krb5.conf가 구성되어 있음
    - https://github.com/onlybooks/kafka2/blob/main/chapter2/ansible_playbook/roles/kerberos/templates/krb5.conf.j2
- 키탭을 사용해 티켓 발급
    ```sh
    $ kinit -kt /home/ubuntu/keytabs/peter01.user.keytab peter01
    ```
- 티켓 발급 확인
    ```sh
    $ klist
    Ticket cache: FILE:/tmp/krb5cc_p30193
    Default principal: peter01@FOO.BAR

    Valid starting     Expires            Service principal
    12/15/22 16:57:09  12/16/22 16:57:09  krbtgt/FOO.BAR@FOO.BAR
            renew until 12/22/22 16:57:09
    ```
- 다른 사용자 키탭으로도 인증이 잘 되는지 확인

### 9.3.3 브로커 커버로스 설정
- 브로커의 server.properties 설정 변경
    - `listeners`, `advertised.listeners`, `security.inter.broker.protocol`
    - SSL 적용 후 수정했던 내용과 유사
    - SASL 관련 설정 추가
    ```properties
    listeners=PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093,SASL_PLAINTEXT://0.0.0.0:9094

    advertised.listeners=PLAINTEXT://peter-kafka01.foo.bar:9092,SSL://peter-kafka01.foo.bar:9093,SASL_PLAINTEXT://peter-kafka01.foo.bar:9094

    security.inter.broker.protocol=SASL_PLAINTEXT
    sasl.mechanism.inter.broker.protocol=GSSAPI
    sasl.enabled.mechanism=GSSAPI
    sasl.kerberos.service.name=kafka
    ```
- jaas.conf 생성
    - JAAS(Java Authentication and Authorization)
    - 자바 애플리케이션 전용 사용자 인증 API
    - JAAS 설정을 수정하는 것만으로 인증을 적용할 수 있음
- kafka_server_jaas.conf
    - 모든 브로커에서 설정해야 함
    - 키탭(keyTab) 설정은 각 브로커의 호스트네임에 맞게 변경
    - 인증 시 커버로스 모듈 사용, 키탭 사용, 키탭의 경로, 프린시펄 지정
    ```
    KafkaServer {
        com.sun.security.auth.module.Krb5LoginModule required
        useKeyTab=true
        storeKey=true
        keyTab="/usr/local/kafka/keytabs/peter-kafka01.service.keytab"
        principal="kafka/peter-kafka01.foo.bar@FOO.BAR";
    };
    ```
- 실행 시 kafka_server_jaas.conf 파일을 로드할 수 있도록 `KAFKA_OPTS` 환경변수에 설정을 추가
    ```
    KAFKA_OPTS="-Djava.security.auth.login.config=/home/ubuntu/apps/kafka/config/kafka_server_jaas.conf"
    ```
- kafka 서버 재시작 후 9094 포트 오픈 확인

### 9.3.4 클라이언트 커버로스 설정
- (콘솔 프로듀서와 콘솔 컨슈머를 사용한다는 가정)
- 클라이언트도 동일하게 jaas 설정 필요
- kafka_client_jaas.conf
    - 서버에서와 동일하게 키탭을 지정해줄 수도 있으나, 클라이언트 예제에서는 `useTicketCache=true` 를 통해 캐시된 티켓을 활용
    - 즉, 클라이언트를 실행하기 전 `kinit`으로 티켓을 발급받아놓은 상태여야 함
    ```
    KafkaClient {
        com.sun.security.auth.module.Krb5LoginModule required
        useTicketCache=true;
    };
    ```
- `KAFKA_OPTS` 환경변수에 설정을 추가
    ```
    KAFKA_OPTS="-Djava.security.auth.login.config=/home/ubuntu/apps/kafka/config/kafka_client_jaas.conf"
    ```
- 콘솔 프로듀서와 콘솔 컨슈머에서 사용할 kerberos.config 파일 생성
    ```
    sasl.mechanism=GSSAPI
    security.protocol=SASL_PLAINTEXT
    sasl.kerberos.service.name=kafka
    ```
- 키탭을 사용해 티켓 발급(9.3.2 참고)
    ```sh
    $ kinit -kt /home/ubuntu/keytabs/peter01.user.keytab peter01
    ```
- 콘솔 프로듀서로 메시지 전송
    ```sh
    $ bin/kafka-console-producer.sh --bootstrap-server peter-kafka01.foo.bar:9094 --topic peter-test08 --producer.config kerberos.config
    ```
- 콘솔 컨슈머로 메시지 수신
    ```sh
    $ bin/kafka-console-consumer.sh --bootstrap-server peter-kafka01.foo.bar:9094 --topic peter-test08 --from-beginning --consumer.config kerberos.config
    ```
- 티켓 삭제
    ```sh
    $ kdestroy
    ```
- 이후 콘솔 컨슈머 실행 시 티켓이 없으므로 익셉션 발생

### 주키퍼를 커버로스와 연동하는 방법
- https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zookeeper+and+SASL

## 9.4 ACL을 이용한 카프카 권한 설정
- 유저별로 특정 토픽에 접근을 허용할지 또는 거부할지 설정
- 브로커의 설정파일을 수정하고, `kafka-acls.sh` 명령을 이용하 유저별 권한 설정

### 9.4.1 브로커 권한 설정
- server.properties 설정 수정
    - 모든 브로커에 동일하게 적용
    - `admin`, `kafka` 유저를 슈퍼유저로 지정
    ```properties
    security.inter.broker.protocol=SASL_PLAINTEXT
    sasl.mechanism.inter.broker.protocol=GSSAPI
    sasl.enabled.mechanism=GSSAPI
    sasl.kerberos.service.name=kafka
    # 아래 내용 추가
    authorizer.class.name=kafka.security.authorizer.AclAuthorizer
    super.users=User:admin;User:kafka
    ```
- 브로커 재시작 후 `peter-test09`, `peter-test10` 토픽 추가
    ```sh
    # 주키퍼에는 커버로스가 적용되어 있지 않으므로, 앞서 설정한 커버로스 설정 비활성화
    $ unset KAFKA_OPTS
    
    $ bin/kafka-topics.sh --zookeeper peter-zk01.foo.bar:2181 --create --topic peter-test09 --partitions 1 --replication-factor 1
    $ bin/kafka-topics.sh --zookeeper peter-zk01.foo.bar:2181 --create --topic peter-test10 --partitions 1 --replication-factor 1
    ```

### 9.4.2 유저별 권한 설정
- 9.3에서 `peter01`, `peter02`, `admin` 3명의 유저 프린시펄 생성
- 이 유저들에게 각각 ACL 규칙을 만들고 테스트 수행
- ![image.png](/files/3429198596200224910)
- 목표
    - `peter01`
        - `peter-test09` 토픽에 대해 읽기/쓰기 가능
    - `peter02`
        - `peter-test10` 토픽에 대해 읽기/쓰기 가능
    - `admin`
        - `peter-test09`, `peter-test10` 토픽에 대해 읽기/쓰기 가능
- `kafka-acls.sh` 명령을 이용해 `peter-01` 유저 ACL 규칙 생성
    ```sh
    $ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=peter-zk01.foo.bar:2181 --add --allow-principal User:peter01 --operation Read --operation Write --operation DESSCRIBE --topic peter-test09
    ```
    - READ, WRITE, DESCRIBE 오퍼레이션 승인 메시지 확인
    - 필요에 따라 접근을 허용할 호스트를 `--allow-host 123.45.1.2` 와 같은 형식으로 추가할 수 있음
    - 그룹 리소스 타입(resourceType)에 대한 ACL은 지금 적용하지 않음
- 동일한 과정으로 `peter-02` 유저 ACL 규칙 생성
    ```sh
    $ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=peter-zk01.foo.bar:2181 --add --allow-principal User:peter02 --operation Read --operation Write --operation DESSCRIBE --topic peter-test10
    ```
- ACL 규칙 리스트 조회
    ```sh
    $ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=peter-zk01.foo.bar:2181 -list
    ```
- `peter01` 유저에 대한 티켓 발급
    ```sh
    $ kinit -kt /home/ubuntu/keytabs/peter01.user.keytab peter01
    ```
- 콘솔 프로듀서로 `peter-test09` 토픽에 메시지 전송
    ```sh
    $ bin/kafka-console-producer.sh --bootstrap-server peter-kafka01.foo.bar:9094 --topic peter-test09 --producer.config kerberos.config
    ```
    - 전송 성공
- 콘솔 프로듀서로 `peter-test10` 토픽에 메시지 전송
    ```sh
    $ bin/kafka-console-producer.sh --bootstrap-server peter-kafka01.foo.bar:9094 --topic peter-test10 --producer.config kerberos.config
    ```
    - 전송 실패
- 콘솔 컨슈머로 `peter-test09` 토픽에서 메시지 수신
    ```sh
    $ bin/kafka-console-consumer.sh --bootstrap-server peter-kafka01.foo.bar:9094 --topic peter-test09 --from-beginning --consumer.config kerberos.config
    ```
    - **수신 실패**
    - `console-consumer-###` 컨슈머 그룹에 대한 접근 권한이 없음
    - 카프카에서는 리소스 타입으로 권한을 설정하도록 구분되어 있고, 컨슈머 그룹에 대한 권한 설정은 그룹 리소스 타입의 권한이 필요
- 모든 컨슈머 그룹이 가능하도록 ACL 규칙 추가
    - 필요에 따라 `'*'` 이 아니라 특정 컨슈머 그룹으로 한정
    ```sh
    $ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=peter-zk01.foo.bar:2181 --add --allow-principal User:peter01 --operation Read --group '*'
    ```
- 이후 콘솔 컨슈머로 메시지 가져오면 성공
- `peter02`사용자에 대해서도 동일
- 슈퍼유저인 `admin`은 별도의 ACL을 설정하지 않아도 `peter-test09`, `peter-test10` 토픽에 대해 메시지 전송 및 수신이 가능

## 9.5 정리
- 카프카 보안은 적용도 트러블슈팅도 어려움
- 여러가지 상황을 잘 검토하여 적용

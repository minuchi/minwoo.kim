---
author: minwoo.kim
categories:
  - kafka
date: 2024-10-21T05:32:00Z
tags:
  - kafka
title: "Apache Kafka란?"
showToc: true
cover:
  image: "/assets/img/kafka.jpg"
  alt: "Coding"
  relative: false
---

해당 게시글은 Apache Kafka 3.8.0 기준으로 작성되었습니다.
Apache Kafka의 기본 개념에 대해 다룹니다.

## Apache Kafka란?

- Apache Kafka는 이벤트 스트림 처리, 실시간 데이터 파이프라인, 대규모 데이터 통합에 사용되는 오픈 소스 분산 스트리밍 시스템이다.
- Kafka의 이름은 [Franz Kafka](https://en.wikipedia.org/wiki/Franz_Kafka)에서 왔다.
  - [What is the relation between Kafka, the writer, and Apache Kafka, the distributed messaging system?](https://www.quora.com/What-is-the-relation-between-Kafka-the-writer-and-Apache-Kafka-the-distributed-messaging-system)

## Apache Kafka 사용 사례

Event(메시지/데이터)가 사용되는 모든 곳에서 사용

- Messaging System
- IOT 디바이스로부터 데이터 수집
- 애플리케이션에서 발생하는 로그 수집
- Reatime Event Stream Processing (Fraud Detection, 이상 감지 등)
- DB 동기화 (MSA 기반의 분리된 DB간 동기화)
- 실시간 ETL (Extract, Transform, Load)
- Spark, Flink, Storm, Hadoop 과 같은 빅데이터 기술과 같이 사용

## Kafka, Pulsar, RabbitMQ 속도 비교

|                                   | **Kafka**               | **Pulsar**               | **RabbitMQ**<br>**(Mirrored)**   |
| --------------------------------- | ----------------------- | ------------------------ | -------------------------------- |
| **Peak Throughput**<br>**(MB/s)** | 605<br>MB/s             | 305<br>MB/s              | 38<br>MB/s                       |
| **p99 Latency**<br>**(ms)**       | 5 ms<br>(200 MB/s load) | 25 ms<br>(200 MB/s load) | 1 ms\*<br>(reduced 30 MB/s load) |

- \*30MB/s 이상의 처리량에서는 RabbitMQ 지연 시간이 크게 저하된다. 또한 Mirroring 영향은 처리량이 높을수록 크게 나타나며, Mirroring 없이 기존 대기열만 사용해도 지연 시간을 개선할 수 있다. [(참고)](https://www.confluent.io/blog/kafka-fastest-messaging-system/)

## Apache Kafka 주요 요소

### Apache Kafka Clients

![apache-kafka-clients](/assets/post/2024/what-is-apache-kafka/apache-kafka-clients.png)

- Producer: 메시지를 생산(Produce)해서 Kafka의 Topic으로 메시지를 보내는 애플리케이션
- Consumer: Topic의 메시지를 가져와서 소비(Consume)하는 애플리케이션
- Consumer Group: Topic의 메시지를 사용하기 위해 협력하는 Consumer들의 집합

하나의 Consumer는 하나의 Consumer Group에 포함되며, Consumer Group내의 Consumer들은 협력하여 Topic의 메시지를 분산 병렬 처리함

- Commit Log: 추가만 가능하고 변경이 불가능한 데이터 스트럭처
  데이터(Event)는 항상 로그 끝에 추가되고 변경되지 않음
- Offset: Commit Log에서 Event의 위치
  아래 그림에서는 0부터 10까지의 Offset을 볼 수 있음

![apache-kafka-commit-log](/assets/post/2024/what-is-apache-kafka/apache-kafka-commit-log.png)

Producer가 Write하는 `LOG-END-OFFSET` 과 Consumer Group의 Consumer가 Read하고 처리한 후에 Commit한 `CURRENT-OFFSET` 과의 차이(`Consumer Lag`)가 발생할 수 있음

![apache-kafka-consumer-lag](/assets/post/2024/what-is-apache-kafka/apache-kafka-consumer-lag.png)

### Topic, Partition, Segment

- Topic: Kafka 안에서 메시지가 저장되는 장소, 논리적인 표현
- Partition: Commit Log, 하나의 Topic은 하나 이상의 Partition으로 구성
- Segment: 메시지(데이터)가 저장되는 실제 물리 File
  Segment File이 지정된 크기보다 크거나 지정된 기간보다 오래되면 새 파일이 열리고 메시지는 새 파일에 추가됨

### Broker, Zookeeper

#### Broker

Kafka Broker는 Partition에 대한 Read 및 Write를 관리하는 소프트웨어

- Kafka Server라고 부르기도 함
- Topic 내의 Partition 들을 분산, 유지 및 관리
- 각각의 Broker들은 ID로 식별됨 (단, ID는 숫자)
- Topic 데이터의 일부분(Partition)을 갖을 뿐 데이터 전체를 갖고 있지 않음
- Kafka Cluster
  - 여러 개의 Broker들로 구성됨
  - Client는 특정 Broker에 연결하면 전체 클러스터에 연결됨
  - 최소 3대 이상의 Broker를 하나의 Cluster로 구성해야함

#### Bootstrap Servers

![apache-kafka-brokers(/assets/post/2024/what-is-apache-kafka/apache-kafka-brokers.png)

- 모든 Kafka Broker는 Bootstrap(부트스트랩) 서버라고 부름
- 하나의 Broker에 연결하면 Cluster 전체에 연결됨
  → 하지만, 특정 Broker 장애를 대비하여, 전체 Broker List(IP, port)를 파라미터로 입력 권장
- 각각의 Broker는 모든 Broker, Topic, Partition에 대해 알고 있음 (Metadata)

#### Zookeeper

![apache-kafka-with-zookeeper](/assets/post/2024/what-is-apache-kafka/apache-kafka-with-apache-zookeeper.png)

Zookeeper는 Broker를 관리 (Broker 들의 목록/설정을 관리)하는 소프트웨어

- Zookeeper는 변경사항에 대해 Kafka에 알림
  → Topic 생성/제거, Broker 추가/제거 등
- Zookeeper 없이는 Kafka가 작동할 수 없었음
  - 2024-10-20 기준으로, KRaft(Kafka Raft)를 이용할 수 있음
- Zookeeper에는 Leader(writes)가 있고 나머지 서버는 Follower(reads)

[KIP-833](https://cwiki.apache.org/confluence/display/KAFKA/KIP-833%3A+Mark+KRaft+as+Production+Ready)에 따르면 Kafka 3.3 버전부터 KRaft를 production-ready 로 선언하였다.
**Kafka 4.0 부터 ZooKeeper를 사용할 수 없고, KRaft만 지원한다.**
2024-10-20 기준으로 Kafka의 최신 버전은 3.8.0이다.

#### Zookeeper 아키텍처

![apache-kafka-zookeeper-ensemble](/assets/post/2024/what-is-apache-kafka/apache-zookeeper-ensemble.png)

Leader/Follower 기반 Master/Slave 아키텍처

Zookeeper는 분산형 Configuration 정보 유지, 분산 동기화 서비스를 제공하고 대용량 분산 시스템을 위한 네이밍 레지스트리를 제공하는 소프트웨어

분산 작업을 제어하기 위한 Tree 형태의 데이터 저장소

→ Zookeeper를 사용하여 멀티 Kafka Broker들 간의 정보(변경 사항 포함) 공유, 동기화 등을 수행

#### Zookeeper Failover

Quorum 알고리즘 기반

Ensemble은 Zookeeper 서버의 클러스터
Quorum(쿼럼)은 **정족수**이며, 합의체가 의사를 진행하거나 의결을 하는데 필요한 최소 한도의 인원수를 뜻함
분산 코디네이션 환경에서 예상치 못한 장애가 발생해도 분산 시스템이 일관성을 유지시키기 위해서 사용

- Ensemble이 3대로 구성되었다면 Quorum은 2, 즉 Zookeeper 1대가 장애가 발생하더라도 정상 동작
- Ensemble이 5대로 구성되었다면 Quorum은 3, 즉 Zookeeper 2대가 장애가 발생하더라도 정상 동작

## In-Sync Replicas (ISR)

### ISR을 관리하는 곳

- Topic의 Leader Partition이 존재하는 Broker가 관리

### ISR을 판단하는 방법

- `replica.lag.max.messages`
  - Follower가 최대 몇 개까지의 복제가 늦어지는지 확인
  - 순간적으로 유입량이 늘어나는 경우 OSR(Out-of-Sync Replicas)로 판단해버릴 수 있는 문제가 있음
- `replica.lag.time.max.ms`
  - 해당 시간 내에 Follower가 Fetch 하지 않으면 ISR에서 제거

## Controller

- Kafka Cluster 내의 Broker 중 하나가 Controller가 됨
- Controller는 ZooKeeper를 통해 Broker Liveness를 모니터링
- Controller는 Leader와 Replica 정보를 Cluster 내의 다른 Broker들에게 전달
- Controller는 ZooKeeper에 Replicas 정보의 복사본을 유지한 다음 더 빠른 액세스를 위해 클러스터의 모든 Broker들에게 동일한 정보를 캐시함
- Controller가 Leader 장애시 Leader Election을 수행

## Consumer 관련 Position

![apache-kafka-consumer-positions](/assets/post/2024/what-is-apache-kafka/apache-kafka-consumer-positions.png)

- Last Committed Offset (Current Offset): Consumer가 최종 Commit한 Offset
- Current Position: Consumer가 읽어간 위치(처리 중, Commit 전)
- High Water Mark (Committed): ISR(Leader-Follower)간에 복제된 Offset
- Log End Offset: Producer가 메시지를 보내서 저장된, 로그의 맨 끝 Offset

## 참고

- Kafka 완전 정복 : 클러스터 구축부터 MSA 환경 활용까지 [(패스트캠퍼스)](https://fastcampus.co.kr/)
- https://www.confluent.io/what-is-apache-kafka/
- https://www.confluent.io/blog/kafka-fastest-messaging-system/

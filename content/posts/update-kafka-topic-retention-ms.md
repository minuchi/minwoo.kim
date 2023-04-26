---
author: minwoo.kim
categories:
  - kafka
date: 2023-04-25T08:14:52Z
tags:
  - kafka
title: 'Kafka topic retention.ms 변경하는 법'
cover:
  image: '/assets/img/kafka.jpeg'
  alt: 'Coding'
  relative: false
---

- `KAFKA_BOOTSTRAP_SERVER` : Kafka Bootstrap Server 주소
- `KAFKA_TOPIC` : `retention.ms` 를 변경할 Kafka Topic

구 버전의 경우 `--bootstrap-server` 가 아닌, `--zookeeper` 를 사용한다.  
해당 글은 Kafka version 3.4.0 기준으로 작성함.

## 변경 시도

```bash
kafka-topics.sh --bootstrap-server $KAFKA_BOOTSTRAP_SERVER --alter --topic $KAFKA_TOPIC --config retention.ms=43200000
```

Kafka topic을 위 명령어로 변경을 시도 했지만 아래와 같이 에러가 났다.  
당연히 `kafka-topics.sh` 명령어를 사용할 줄 알았지만 아니였음.

```plaintext
Option combination "[bootstrap-server],[config]" can't be used with option "[alter]" (the kafka-configs CLI supports altering topic configs with a --bootstrap-server option
```

## 해결 방법

```bash
kafka-configs.sh --bootstrap-server $KAFKA_BOOTSTRAP_SERVER --alter --entity-type topics --entity-name $KAFKA_TOPIC --add-config retention.ms=43200000
```

위 명령어를 실행하면 아래와 같이 성공했다는 메시지가 나온다.

```plaintext
Completed updating config for topic $KAFKA_TOPIC.
```

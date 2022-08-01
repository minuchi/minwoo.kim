---
author: minwoo.kim
categories:
  - k8s
  - RabbitMQ
date: 2022-08-01T03:30:00Z
tags:
  - k8s
  - rabbitmq
title: 'RabbitMQ Cluster 재시작 시 에러 (Kubernetes)'
cover:
  image: '/assets/img/rabbitmq.png'
  alt: 'RabbitMQ'
  relative: false
---

# RabbitMQ Cluster 재시작 시 에러

Kubernetes 환경에 올라가 있는 RabbitMQ가 재시작 되는 경우 아래와 같이 계속 재시도를 하다가 뜨지 않는 현상이 발생하는 경우가 있습니다.

```plaintext
2022-08-01 12:55:50.946 [info] <0.282.0> Running boot step pre_boot defined by app rabbit
2022-08-01 12:55:50.946 [info] <0.282.0> Running boot step rabbit_core_metrics defined by app rabbit
2022-08-01 12:55:50.947 [info] <0.282.0> Running boot step rabbit_alarm defined by app rabbit
2022-08-01 12:55:50.950 [info] <0.288.0> Memory high watermark set to 3152 MiB (3305309798 bytes) of 7880 MiB (8263274496 bytes) total
2022-08-01 12:55:50.955 [info] <0.290.0> Enabling free disk space monitoring
2022-08-01 12:55:50.955 [info] <0.290.0> Disk free limit set to 50MB
2022-08-01 12:55:50.958 [info] <0.282.0> Running boot step code_server_cache defined by app rabbit
2022-08-01 12:55:50.958 [info] <0.282.0> Running boot step file_handle_cache defined by app rabbit
2022-08-01 12:55:50.959 [info] <0.293.0> Limiting to approx 1048476 file handles (943626 sockets)
2022-08-01 12:55:50.959 [info] <0.294.0> FHC read buffering:  OFF
2022-08-01 12:55:50.959 [info] <0.294.0> FHC write buffering: ON
2022-08-01 12:55:50.959 [info] <0.282.0> Running boot step worker_pool defined by app rabbit
2022-08-01 12:55:50.959 [info] <0.283.0> Will use 2 processes for default worker pool
2022-08-01 12:55:50.959 [info] <0.283.0> Starting worker pool 'worker_pool' with 2 processes in it
2022-08-01 12:55:50.960 [info] <0.282.0> Running boot step database defined by app rabbit
2022-08-01 12:55:50.969 [info] <0.282.0> Waiting for Mnesia tables for 30000 ms, 9 retries left
2022-08-01 12:56:20.731 [info] <0.313.0> RabbitMQ hasn't finished starting yet. Waiting for startup to finish before stopping...
2022-08-01 12:56:20.970 [warning] <0.282.0> Error while waiting for Mnesia tables: {timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_route,rabbit_durable_exchange,rabbit_runtime_parameters,rabbit_durable_queue]}
2022-08-01 12:56:20.971 [info] <0.282.0> Waiting for Mnesia tables for 30000 ms, 8 retries left
2022-08-01 12:56:50.971 [warning] <0.282.0> Error while waiting for Mnesia tables: {timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_route,rabbit_durable_exchange,rabbit_runtime_parameters,rabbit_durable_queue]}

...생략...

2022-08-01 12:59:20.976 [info] <0.282.0> Waiting for Mnesia tables for 30000 ms, 2 retries left
2022-08-01 12:59:50.978 [warning] <0.282.0> Error while waiting for Mnesia tables: {timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_route,rabbit_durable_exchange,rabbit_runtime_parameters,rabbit_durable_queue]}
2022-08-01 12:59:50.978 [info] <0.282.0> Waiting for Mnesia tables for 30000 ms, 1 retries left
2022-08-01 13:00:20.978 [warning] <0.282.0> Error while waiting for Mnesia tables: {timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_route,rabbit_durable_exchange,rabbit_runtime_parameters,rabbit_durable_queue]}
2022-08-01 13:00:20.979 [info] <0.282.0> Waiting for Mnesia tables for 30000 ms, 0 retries left
2022-08-01 13:00:31.468 [info] <0.439.0> RabbitMQ hasn't finished starting yet. Waiting for startup to finish before stopping...
2022-08-01 13:00:50.980 [error] <0.281.0> CRASH REPORT Process <0.281.0> with 0 neighbours exited with reason: {{timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_route,rabbit_durable_exchange,rabbit_runtime_parameters,rabbit_durable_queue]},{rabbit,sta
rt,[normal,[]]}} in application_master:init/4 line 138
2022-08-01 13:00:50.980 [info] <0.44.0> Application rabbit exited with reason: {{timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_route,rabbit_durable_exchange,rabbit_runtime_parameters,rabbit_durable_queue]},{rabbit,start,[normal,[]]}}
{"Kernel pid terminated",application_controller,"{application_start_failure,rabbit,{{timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_route,rabbit_durable_exchange,rabbit_runtime_parameters,rabbit_durable_queue]},{rabbit,start,[normal,[]]}}}"}
Kernel pid terminated (application_controller) ({application_start_failure,rabbit,{{timeout_waiting_for_tables,[rabbit_user,rabbit_user_permission,rabbit_topic_permission,rabbit_vhost,rabbit_durable_r

Crash dump is being written to: /var/log/rabbitmq/erl_crash.dump...done
```

이런 상황의 경우 아래와 같이 진행하면 해결됩니다.

```bash
# 1. rabbitmq pod의 개수를 1개로 줄여줍니다.
kubectl scale --repliacs=1 statefulsets rabbitmq

# 2. rabbitmq cluster를 재시작 시켜줍니다.
kubectl exec rabbitmq-0 -- rabbitmqctl stop_app
kubectl exec rabbitmq-0 -- rabbitmqctl force_boot

# 3. 원하는 pod 개수만큼 rabbitmq를 scale out 시켜줍니다.
kubectl scale --repliacs=2 statefulsets rabbitmq-0
```
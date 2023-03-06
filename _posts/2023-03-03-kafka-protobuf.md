---
layout: post
title: Kafka调试protobuf格式的消息
categories: [java]
tags: [kafka]
---

> 

#### 1. 工具

使用[Kafka Protobuf Console](https://github.com/khorshuheng/kafka-protobuf-console)

#### 2. 生成FileDescriptorSets

使用protoc生成FileDescriptorSets文件，以opentelemetry-proto为例

```
$ protoc --descriptor_set_out otlp.fds --proto_path /proto --include_imports /proto/opentelemetry/proto/collector/metrics/v1/metrics_service.proto /proto/opentelemetry/proto/collector/trace/v1/trace_service.proto /proto/opentelemetry/proto/collector/logs/v1/logs_service.proto
```

#### 3. 测试

编写脚本test.sh

```
#!/bin/bash

DIR=`cd $(dirname $0); pwd`

BROKER=$1
TOPIC=$2
if [ "$3" = "trace" ]; then
        PROTO_NAME=opentelemetry.proto.collector.trace.v1.ExportTraceServiceRequest
elif [ "$3" = "metric" ]; then
        PROTO_NAME=opentelemetry.proto.collector.metrics.v1.ExportMetricsServiceRequest
elif [ "$3" = "log" ]; then
        PROTO_NAME=opentelemetry.proto.collector.logs.v1.ExportLogsServiceRequest
else
        echo "Unsupported data type [$3]!"
        exit 1
fi

./kafka-protobuf-console consume -b $1 -d $DIR/otlp.fds -t $2 -n $PROTO_NAME -v 3.2.1
```

```
$ ./test.sh 192.168.0.100:9092 otlp-spans trace
```
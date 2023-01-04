---
layout: post
title: Kafka异步消息
categories: [java, kafka]
tags: []
---

> kafka同步消费受制于分区数和消费线程数，在对数据一致性要求不高的场合，可以用异步消费方式提高消费性能

#### 1. 线程模型
```
+-------+    +--------------------+    +---------------+    +------------------+    +--------------+    +--------------+    +------------+
| Kafka |--->| Consumer Thread(n) |--->| Message Queue |--->| Handle Thread(n) |--->| Entity Queue |--->| Write Thread |--->| ClickHouse |
+-------+    +--------------------+    +---------------+    +------------------+    +--------------+    +--------------+    +------------+
```

#### 2. 代码实现

##### 2.1 Consumer Thread

```java
public void run() {
        try {
            while (true) {
                try {
                    ConsumerRecords<String, byte[]> records = consumer.poll(Duration.ofMillis(fetchMaxWait));
                    int recordSize = records.count();
                    if (recordSize <= 0) {
                        continue;
                    }
                    if (batchHandle) {
                        threadPoolExecutor.execute(() -> handleRecords(records));
                    } else {
                        records.forEach(t -> threadPoolExecutor.execute(() -> handleRecord(t.value())));
                    }
                } catch (WakeupException e) {
                    log.warn("ConsumerThread [{}] waked up");
                    break;
                } catch (KafkaException e) {
                    log.warn("Catch poll exception, ignored.", e);
                    continue;
                }

                try {
                    consumer.commitAsync();
                } catch (KafkaException e) {
                    log.warn("Catch commit exception, ignored.", e);
                }
            }
        } finally {
            consumer.close();
            log.info("Consumer thread [{}] closed");
        }
    }

    private void handleRecords(ConsumerRecords<String, byte[]> records) {
        List<T> result = new LinkedList<>();
        records.forEach(t -> {
            List<T> list = messageHandler.parse(t.value());
            if (CollectionUtils.isEmpty(list)) {
                return;
            }
            result.addAll(list);
        });
        queue.put(result);
    }

    private void handleRecord(byte[] message) {
        List<T> list = messageHandler.parse(message);
        if (CollectionUtils.isEmpty(list)) {
            return;
        }
        queue.put(list);
    }

    public void close() {
        log.info("Closing consumer thread [{}]...", this.getName());
        this.consumer.wakeup();
    }
```

##### 2.2 Message Queue和Handle Thread

这里用ThreadPool实现：

```java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(handleThreadNum, handleThreadNum, 0, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(handleThreadNum * 2),
                new CustomizableThreadFactory("Handler"),
                new ThreadPoolExecutor.CallerRunsPolicy());
```

##### 2.3 Entity Queue

使用LinkedBlockingQueue实现，可以批量写入实体元素，以减少高并发下线程间的锁等待时间
```java
LinkedBlockingQueue<List<MessageDTO>> queue = new LinkedBlockingQueue<>(capacity);
```

##### 2.4 Write Thread

泛型Thread

```java
    private final int maxBatchSize;
    private final long writeInterval;
    private final boolean writeReliable;
    private final LinkedBlockingQueue<List<T>> queue;
    private final List<List<T>> backlog = new LinkedList<>();

    public void run() {
        try {
            long ts = System.currentTimeMillis();
            int size = 0;
            while (true) {
                List<T> list = queue.poll(50, TimeUnit.MILLISECONDS);
                if (CollectionUtils.isEmpty(list)) {
                    if (isRunning) {
                        continue;
                    } else { // Close时，等待队列消费完再退出
                        break;
                    }
                }

                if (ps == null || ps.isClosed()) {
                    returnBacklog();
                    ps = statementPreparer.preparedStatement();
                }

                for (T t : list) {
                    statementPreparer.prepare(ps, t);
                    ps.addBatch();
                }

                writeBacklog(list);
                size += list.size();

                if (size >= maxBatchSize || (System.currentTimeMillis() - ts > writeInterval && size > 0)) {
                    try {
                        long l = System.currentTimeMillis();
                        ps.executeBatch();
                        long t = System.currentTimeMillis() - l;
                        log.info("Saved [{}] rows, cost [{}]ms", size, t);
                    } catch (SQLException e) {
                        log.warn("Save failed", e);
                        returnBacklog();
                    } finally {
                        size = 0;
                        ts = System.currentTimeMillis();
                    }
                }
            }
        } catch (Exception e) {
            log.warn("StoreThread [{}] error", this.getName(), e);
        } finally {
            synchronized (this) {
                this.notifyAll();
            }
        }
    }

    private void writeBacklog(List<T> list) {
        if (!this.writeReliable) {
            return;
        }
        backlog.add(list);
    }

    private void returnBacklog() {
        if (!this.writeReliable || CollectionUtils.isEmpty(backlog)) {
            return;
        }
        log.warn("Returning [{}] rows to queue...", backlog.size());
        for (List<T> li : backlog) {
            queue.put(li);
        }
        backlog.clear();
    }
```

实体写表逻辑
```java
public PreparedStatement preparedStatement() {
        try {
            if (conn == null || conn.isClosed()) {
                conn = dataSource.getConnection();
            }
            return conn.prepareStatement(insertSql);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void prepare(PreparedStatement ps, MessageDTO dto) throws SQLException {
        ps.setString(1, dto.getUsername());
        ps.setString(2, dto.getAge());
        ps.setString(3, dto.getSex());
    }

```
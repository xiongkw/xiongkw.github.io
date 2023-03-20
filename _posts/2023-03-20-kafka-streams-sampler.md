---
layout: post
title: SpEL实现线上动态采样
categories: [java, spring]
tags: []
---

> 大数据场景，往往需要对数据采样以排查问题，如何根据随机的规则来进行线上数据采样

#### 1. Sample

```java
@Getter
@Setter
public class Sample {
    private String expr;
    private long beginTs;
    private long endTs;
    private AtomicInteger total = new AtomicInteger(0);
    private AtomicInteger count = new AtomicInteger(0);
    private int limit;
    private List<Object> exemplars = new LinkedList<>();

    public Sample(int limit) {
        this.beginTs = System.currentTimeMillis();
        this.limit = limit;
    }

    public boolean record(Object t) {
        this.count.incrementAndGet();
        if (this.exemplars.size() < this.limit) {
            this.exemplars.add(t);
            return false;
        }else{
            return true;
        }
    }

    @Override
    public String toString() {
        return "Sample{" +
                "expr='" + expr + '\'' +
                ", beginTs=" + beginTs +
                ", endTs=" + endTs +
                ", total=" + total +
                ", count=" + count +
                ", limit=" + limit +
                ", exemplars=" + exemplars +
                '}';
    }
}
```

#### 2. Sampler

```java
@Component
@Slf4j
@Endpoint(id = "sampler")
public class Sampler {
    private static final int MAX_DELAY = 10 * 60 * 1000; // 10 min
    private static final int MAX_LIMIT = 10;
    private Sample sample;
    private final ExpressionParser parser = new SpelExpressionParser();
    private Expression expression;
    private volatile boolean isSampling;

    @ReadOperation
    public String getSample() {
        if (sample == null) {
            return "{}";
        }
        return this.sample.toString();
    }

    @WriteOperation
    public String sample(String expr, long delay, int limit) {
        if (StringUtils.isEmpty(expr)) {
            return "Empty expr";
        }
        delay = Math.min(delay, MAX_DELAY);
        limit = Math.min(limit, MAX_LIMIT);
        log.info("Sample begin, expr: [{}], delay: [{}], limit: [{}]", expr, delay, limit);
        this.expression = parser.parseExpression(expr);
        this.sample = new Sample(limit);
        this.sample.setExpr(expr);
        this.isSampling = true;

        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                Sampler.this.isSampling = false;
                Sample s = Sampler.this.sample;
                log.info("Sample end, expr: [{}], total: [{}], found [{}] records, [{}] exemplars", expr, s.getTotal(), s.getCount(), s.getExemplars().size());
            }

        }, delay);
        return String.format("Ok. {delay:%d,limit:%d}", delay, limit);
    }

    public void accept(Object o) {
        if (!isSampling) {
            return;
        }
        log.info("Sampling...");
        Sample s = Sampler.this.sample;
        EvaluationContext context = new StandardEvaluationContext(o);
        Object value = Sampler.this.expression.getValue(context);
        s.getTotal().incrementAndGet();
        if (value instanceof Boolean && (Boolean) value) {
            boolean completed = s.record(o);
            if (completed) {
                this.isSampling = false;
            }
        }
    }
}

```

#### 3. ETL

```java
messages.forEach(message -> {
	if (sampler != null) {
		try {
			sampler.accept(message);
		} catch (Exception e) {
			log.warn("Sample error: [{}]", e.getMessage());
		}
	}
	// ...
}
```

#### 4. 例子

对主机名包含gza7，并且指标名为jvm.no_heap.used的ResourceMetrics对象采样

```
$ curl -X POST 'http://127.0.0.1:8080/actuator/sampler' -H 'Content-Type: application/json' -d '{
    "expr":"resource.attributesList.?[key.equals('\''host.name'\'') and value.stringValue.contains('\''gza7'\'')].size() > 0 and scopeMetricsList.?[MetricsList.?[name.contains('\''jvm.non_heap.used'\'')].size()>0].size()>0",
    "delay":120000,
    "limit": 10
}'
```

获取采样结果：
```
$ curl http://127.0.0.1:8080/actuator/sampler
```

#### 5. 参考

* [Spring Expression Language (SpEL)](https://docs.spring.io/spring-framework/docs/6.0.x/reference/html/core.html#expressions)
* [spring-boot-actuator自定义Endpoint]({{ site.url}}/2023/02/02/springboot-actuator-custom-endpoint/)
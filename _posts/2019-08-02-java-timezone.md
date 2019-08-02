---
layout: post
title: 母校回忆
categories: [编程, java]
tags: [timezone]
---

> 有一个线上系统，根据日期查数据，查到的却总是早一天

#### 1. 源码

```java
String dateStr=DateFormatUtils.format(timeInMills, "yyyy.MM.dd");
List list = selectByDateStr(dateStr);
```

#### 2. 分析

通过日志发现这里的`dateStr`比预期值小一天，而同样的`timeInMills`在本地却是正常的，于是推测可能是时区的问题

查看源码`org.apache.commons.lang.time.DateFormatUtils#format(java.util.Date, java.lang.String, java.util.TimeZone, java.util.Locale)`

```java
public static String format(Date date, String pattern, TimeZone timeZone, Locale locale) {
    FastDateFormat df = FastDateFormat.getInstance(pattern, timeZone, locale);
    return df.format(date);
}

public static synchronized FastDateFormat getInstance(String pattern, TimeZone timeZone, Locale locale) {
    FastDateFormat emptyFormat = new FastDateFormat(pattern, timeZone, locale);
    FastDateFormat format = (FastDateFormat) cInstanceCache.get(emptyFormat);
    if (format == null) {
        format = emptyFormat;
        format.init();  // convert shell format into usable one
        cInstanceCache.put(format, format);  // this is OK!
    }
    return format;
}
```

`org.apache.commons.lang.time.FastDateFormat#FastDateFormat`

```java
protected FastDateFormat(String pattern, TimeZone timeZone, Locale locale) {
    super();
    if (pattern == null) {
        throw new IllegalArgumentException("The pattern must not be null");
    }
    mPattern = pattern;
    
    mTimeZoneForced = (timeZone != null);
    if (timeZone == null) {
        timeZone = TimeZone.getDefault();
    }
    mTimeZone = timeZone;
    
    mLocaleForced = (locale != null);
    if (locale == null) {
        locale = Locale.getDefault();
    }
    mLocale = locale;
}
```

```java
public static TimeZone getDefault() {
    return (TimeZone) getDefaultRef().clone();
}

static TimeZone getDefaultRef() {
    TimeZone defaultZone = defaultTimeZone;
    if (defaultZone == null) {
        // Need to initialize the default time zone.
        defaultZone = setDefaultZone();
        assert defaultZone != null;
    }
    // Don't clone here.
    return defaultZone;
}

private static synchronized TimeZone setDefaultZone() {
    TimeZone tz;
    // get the time zone ID from the system properties
    String zoneID = AccessController.doPrivileged(
            new GetPropertyAction("user.timezone"));

    // if the time zone ID is not set (yet), perform the
    // platform to Java time zone ID mapping.
    if (zoneID == null || zoneID.isEmpty()) {
        String javaHome = AccessController.doPrivileged(
                new GetPropertyAction("java.home"));
        try {
            zoneID = getSystemTimeZoneID(javaHome);
            if (zoneID == null) {
                zoneID = GMT_ID;
            }
        } catch (NullPointerException e) {
            zoneID = GMT_ID;
        }
    }
    // ...
}
```

> 发现可以通过`user.timezone`定义jvm默认时区

#### 3. 解决办法

1. 增加加`jvm`启动参数

```
java -Duser.timezone=GMT+08
```

2. 代码中设置时区

```java
String dateStr=DateFormatUtils.format(timeInMills, "yyyy.MM.dd", TimeZone.getTimeZone("GMT+08"));
```

3. 也可以设置默认时区

```java
TimeZone.setDefault(TimeZone.getTimeZone("GMT+08"));
```
---
layout: post
title: Java中gzip压缩和解压中文文本
categories: [编程, java]
tags: [java, gzip]
---

> java中用gzip压缩和解压文本，支持UTF-8编码

```java
public class GzipUtils {

    public static String gzip(String str, String charset) throws IOException {
        if (str == null || str.length() == 0) {
            return null;
        }
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        GZIPOutputStream gos = new GZIPOutputStream(bos);
        try {
            gos.write(str.getBytes(charset));
            gos.finish();
            gos.flush();
            return Base64.encodeBase64String(bos.toByteArray());
        } finally {
            IOUtils.closeQuietly(gos);
        }
    }

    public static String ungzip(String str, String charset) throws IOException {
        if (str == null || str.length() == 0) {
            return null;
        }
        ByteArrayInputStream in = new ByteArrayInputStream(Base64.decodeBase64(str));
        GZIPInputStream gis = new GZIPInputStream(in);
        try {
           return IOUtils.toString(gis, charset);
        } finally {
            IOUtils.closeQuietly(gis);
        }
    }

}
```
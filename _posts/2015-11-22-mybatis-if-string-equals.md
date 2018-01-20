---
layout: post
title: Mybatis动态sql中判断string相等的问题
categories: [编程, java]
tags: [mybatis, ognl]
---

> Mybatis动态sql中判断string相等，却报`NumberFormatException`异常

#### 1. 异常信息
```
### Error querying database.  Cause: java.lang.NumberFormatException: For input string: "r"
### Cause: java.lang.NumberFormatException: For input string: "r"] with root cause
java.lang.NumberFormatException: For input string: "r"
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:2043)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at org.apache.ibatis.ognl.OgnlOps.doubleValue(OgnlOps.java:242)
	at org.apache.ibatis.ognl.OgnlOps.compareWithConversion(OgnlOps.java:99)
	at org.apache.ibatis.ognl.OgnlOps.isEqual(OgnlOps.java:142)
	at org.apache.ibatis.ognl.OgnlOps.equal(OgnlOps.java:794)
	at org.apache.ibatis.ognl.ASTEq.getValueBody(ASTEq.java:52)
	at org.apache.ibatis.ognl.SimpleNode.evaluateGetValueBody(SimpleNode.java:212)
	at org.apache.ibatis.ognl.SimpleNode.getValue(SimpleNode.java:258)
	at org.apache.ibatis.ognl.Ognl.getValue(Ognl.java:494)
	at org.apache.ibatis.ognl.Ognl.getValue(Ognl.java:458)
	at org.apache.ibatis.scripting.xmltags.OgnlCache.getValue(OgnlCache.java:44)
	at org.apache.ibatis.scripting.xmltags.ExpressionEvaluator.evaluateBoolean(ExpressionEvaluator.java:32)
	at org.apache.ibatis.scripting.xmltags.IfSqlNode.apply(IfSqlNode.java:34)
	at org.apache.ibatis.scripting.xmltags.MixedSqlNode.apply(MixedSqlNode.java:33)
	at org.apache.ibatis.scripting.xmltags.DynamicSqlSource.getBoundSql(DynamicSqlSource.java:41)
	at org.apache.ibatis.mapping.MappedStatement.getBoundSql(MappedStatement.java:280)
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:80)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:120)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:113)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:497)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:386)
	at com.sun.proxy.$Proxy17.selectList(Unknown Source)
	at org.mybatis.spring.SqlSessionTemplate.selectList(SqlSessionTemplate.java:205)
	at org.apache.ibatis.binding.MapperMethod.executeForMany(MapperMethod.java:122)
	at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:64)
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:53)
	at com.sun.proxy.$Proxy38.queryPayback(Unknown Source)
```

#### 2. 代码
sqlmap：
```xml
<if test="status == 'p'">
...
</if>
```

参数：
```java
String status = "r";
```

#### 3. 调试跟踪

跟踪if判断到SqlNode.evaluateBoolean
```java
public boolean apply(DynamicContext context) {
    if (evaluator.evaluateBoolean(test, context.getBindings())) {
      contents.apply(context);
      return true;
    }
    return false;
}
```

接着到ASTEq.getValueBody
```java
protected Object getValueBody(OgnlContext context, Object source) throws OgnlException {
    // v1="r" v1.getClass()为String
    Object v1 = this._children[0].getValue(context, source);
    // v2='p' v2.getClass为Character
    Object v2 = this._children[1].getValue(context, source);
    // 所以v1和v2是不相同的
    return OgnlOps.equal(v1, v2)?Boolean.TRUE:Boolean.FALSE;
}
```
> 这里发现ognl把`<if test="status == 'p'">`中的`'p'`解析成了Character

进一步跟踪到OgnlOps.compareWithConversion
```java
public static int compareWithConversion(Object v1, Object v2) {
    int result;
    if(v1 == v2) {
        result = 0;
    } else {
        int t1 = getNumericType(v1);
        int t2 = getNumericType(v2);
        int type = getNumericType(t1, t2, true);
        switch(type) {
        case 6:
            result = bigIntValue(v1).compareTo(bigIntValue(v2));
            break;
        case 9:
            result = bigDecValue(v1).compareTo(bigDecValue(v2));
            break;
        case 10:
            if(t1 == 10 && t2 == 10) {
                if(v1 instanceof Comparable && v1.getClass().isAssignableFrom(v2.getClass())) {
                    result = ((Comparable)v1).compareTo(v2);
                    break;
                }

                throw new IllegalArgumentException("invalid comparison: " + v1.getClass().getName() + " and " + v2.getClass().getName());
            }
        case 7:
        case 8:
            double dv1 = doubleValue(v1);
            double dv2 = doubleValue(v2);
            return dv1 == dv2?0:(dv1 < dv2?-1:1);
        default:
            long lv1 = longValue(v1);
            long lv2 = longValue(v2);
            return lv1 == lv2?0:(lv1 < lv2?-1:1);
        }
    }

    return result;
}
```

> case 10 7 8：异常发生在doubleValue(v1)

#### 4. 改正

异常的原因是ognl把`<if test="status == 'p'">`中的`'p'`解析成了`Character`，改正方法：

* `<if test="status == 'p'">`改为`<if test='status == "p"'>`，这样`"p"`会解析为`String`
* 单个字符解析为`Character`，猜想字符串会解析正常，于是改为`<if test="status == 'pp'">`，调试发现解析为了`String`
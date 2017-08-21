---
layout: post
title: 使用正则表达式转换count-sql语句
categories: [编程, java]
tags: [sql, count, regex]
---


> dao底层实现中，通用count封装方式为`select count(1) from(selectSql)`,该方法的好处是简单，缺点是生成了临时表，对数据库性能有影响

一种优化的方法是替换sql中select部分，而不使用临时表，例如
```sql
#查询sql
select id, name from t_user;

#替换 id, name 为 count(1)
select count(1) from t_user;
```

该方法的要点在于准确识别sql中select...from的语句，当然可以使用工具包(例如jsqlparser等)来解析sql语法，这里使用java正则表达式来匹配，顺便复习下正则表达式

##### 1. 总体思路是使用正则表达式的分组，捕获`select...from`段，再替换成`count`，如下：
```java
Pattern p = Pattern.compile("(select\\s+.*\\s+from)");
String sql = "select id, name from t_user;";
Matcher matcher = p.matcher(sql);
if(matcher.find()){
    String r = matcher.replaceFirst("SELECT COUNT(1) FROM");
    System.out.println(r);//输出 SELECT COUNT(1) FROM t_user;
}
```
> 正则表达式中的`()`表示分组，`((A)(B(C)))`可分为如下四组
```
1 ((A)(B(C)))
2 (A) 
3 (B(C))
4 (C)
```

##### 2. 由于sql一般不区分大小写，还会分为多行，所以这里需要增加相应参数`Pattern.CASE_INSENSITIVE|Pattern.MULTILINE`
```java
Pattern p = Pattern.compile("(select\\s+.*\\s+from)", Pattern.CASE_INSENSITIVE|Pattern.MULTILINE);
```

##### 3. 由于sql中会包含大量换行和tab，而正则表达式中`.`是不匹配换行和tab的，所以正则表达式修改如下：
```java
(select\\s+[\\S\\s]*\\s+from)
```
> [\\S\\s]匹配所有空格字符和非空格字符

##### 4. 子查询的情况下，`from`语句后还有`select ... from`
```sql
select t.id, t.name 
from t_user t, 
  (select * from t_role) r
where t.id = r.user_id
```

贪婪模式下，`[\\S\\s]*`会匹配到第二个from之前，所以这里要改为非贪婪模式
```java
(select\\s+[\\S\\s]*?\\s+from)
```
> 正则表达式默认是贪婪模式，使用`量词+?`即可切换到非贪婪模式，例如`.*?`

##### 5. select语句中嵌套select的情况
```sql
select t.id, t.name, 
  (select id from t_xx) as tid,
  (select name from t_xx) as tname
from t_user t, 
  (select * from t_role) r
where t.id = r.user_id
```

这里要增加子分组
```java
(select\\s+[\\S\\s]*?(?:select\\s+[\\S\\s]*?from[\\S\\s]*?)*\\s+from)
```
> 使用子分组`(?:select\\s+[\\S\\s]*?from[\\S\\s]*?)*`匹配嵌套的`select`语句   
> `[\\S\\s]*?`表示非贪婪匹配，即匹配到`from`之前      
> `(?:)`表示匹配但不记录分组



> 更复杂的sql暂不考虑
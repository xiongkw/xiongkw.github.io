---
layout: post
title: Javascript中的一些日期计算方法
categories: [javascript]
tags: []
---

> 

#### 1. 格式化

```
function formatDate(d){
    const year = d.getFullYear();
    const mon = d.getMonth() + 1;
    const day = d.getDate();
    return year + "" + (mon < 10 ? ('0' + mon) : mon) + "" + (day < 10 ? ('0' + day) : day);
}
```

#### 2. 获取自然月份起始日期

```
function getDateRangeOfMonth(year, month){
    const beginDate=new Date(year,month-1,1);//开始日期，即1日
    const endDate=new Date(year,month,0);//date参数为0表示取前1天，即上个月最后一天
    const begin=formatDate(beginDate);
    const end=formatDate(endDate);
    console.log(begin+"-"+end);
}

getDateRangeOfMonth(2023,8);//20230801-20230831

```

#### 3. 获取前n天的日期

```
function getDateBefore(n) {
    let d = new Date();
    let year = d.getFullYear();
    let mon = d.getMonth() + 1;
    let day = d.getDate();
    if(day <= n) {
        if(mon > 1) {
            mon = mon - 1;
        } else {
            year = year - 1;
            mon = 12;
        }
    }
    d.setDate(d.getDate() - n);
    return formatDate(d);
}

getDateBefore(1);//获取昨天的日期
```
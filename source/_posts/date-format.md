---
title: java日期格式化总结
categories:
- java
date: 2018-04-01 14:15:00
tags:
- java
- date
---

**模式字母**:(a-z,A-z)在日期格式化中会被解释,单引号('')包裹的字符不解释，作为文本直接输出  
**其他字符**: 作为文本直接输出  
<!--more-->
模式字母:  

| 字母 | 日期或时间元素 | 表示 | 示例 |
|:-:|:---:| :---:|:---|
| G | Era 标志符 | Text | AD |
| y | 年 | Year | 1996; 96 |
| M | 年中的月份 | Month | July; Jul; 07 |
| w | 年中的周数 | Number | 27 |
| W | 月份中的周数 | Number | 2 |
| D | 年中的天数 | Number | 189 |
| d | 月份中的天数 | Number | 10 |
| F | 月份中的星期 | Number | 2 |
| E | 星期中的天数 | Text	 | Tuesday; Tue |
| a | Am/pm 标记 | Text | PM |
| H | 一天中的小时数（0-23) | Number | 0 |
| k | 一天中的小时数（1-24） | Number | 24 |
| K | am/pm 中的小时数（0-11） | Number | 0 |
| h | am/pm 中的小时数（1-12） | Number | 12 |
| m | 小时中的分钟数 | Number | 30 |
| s | 分钟中的秒数 | Number | 55 |
| S | 毫秒数 | Number | 978 |
| z | 时区 | General time zone | Pacific Standard Time;PST; GMT-08:00 |
| Z | 时区 | RFC 822 time zone | -0800 |
  
美国太平洋时区的本地时间 2001-07-04 12:08:56格式化：  

| 日期和时间模式 | 结果 |
| :--- | :---|
| "yyyy.MM.dd G 'at' HH:mm:ss z" | 2001.07.04 AD at 12:08:56 PDT |
| "EEE, MMM d, ''yy" | Wed, Jul 4, '01 |
| "h:mm a" | 12:08 PM |
| "hh 'o''clock' a, zzzz" | 12 o'clock PM, Pacific Daylight Time |
| "K:mm a, z" | 0:08 PM, PDT |
| "yyyyy.MMMMM.dd GGG hh:mm aaa" | 02001.July.04 AD 12:08 PM |
| "EEE, d MMM yyyy HH:mm:ss Z" | Wed, 4 Jul 2001 12:08:56 -0700 |
| "yyMMddHHmmssZ" | 010704120856-0700 |
| "yyyy-MM-dd'T'HH:mm:ss.SSSZ" | 2001-07-04T12:08:56.235-0700 |
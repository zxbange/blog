---
layout: post
title:  "MySQL JSON字段"
subtitle: "介绍MySQLJSON字段功能"
date:   2021-07-14 19:31:29 +0900
categories: MySQL
author:  "张鑫"
tags:
  - JSON字段
---

# JSON数据类型
JSON 分JSON对象和JSON数组
JSON对象
```json
{
 "Image": {
   "Width": 800,
   "Height": 600,
   "Title": "View from 15th Floor",
   "Thumbnail": {
     "Url": "http://www.example.com/image/481989943",
     "Height": 125,
     "Width": 100
   },
 "IDs": [116, 943, 234, 38793]
 }
}
```

JSON数组
```json
[
   {
     "precision": "zip",
     "Latitude": 37.7668,
     "Longitude": -122.3959,
     "Address": "",
     "City": "SAN FRANCISCO",
     "State": "CA",
     "Zip": "94107",
     "Country": "US"
   },
   {
     "precision": "zip",
     "Latitude": 37.371991,
     "Longitude": -122.026020,
     "Address": "",
     "City": "SUNNYVALE",
     "State": "CA",
     "Zip": "94085",
     "Country": "US"
   }
 ]
```

* 使用 JSON 数据类型，推荐用 MySQL 8.0.17 以上的版本，性能更好，同时也支持 Multi-Valued Indexes；
* JSON 数据类型的好处是无须预先定义列，数据本身就具有很好的描述性；
* 不要将有明显关系型的数据用 JSON 存储，如用户余额、用户姓名、用户身份证等，这些都是每个用户必须包含的数据；
* JSON 数据类型推荐使用在不经常更新的静态数据存储。
---
layout: post
title: "TOEFL 考位自动查询"
subtitle: "Auto Query of TOEFL Test Seats"
author: "Michael Wang"
header-img: "img/post-bg-js-version.jpg"
header-mask: 0.3
tags:
  - Web Crawler
  - Python
---

## 问题背景 Background
最近Tiffany女士想要查询一下托福考位，但是不同的场次需要一个个单独查询，非常麻烦。
然后我就想到那句名言：人生苦短，我用Python。何不写一个小爬虫去查询呢？    
Recently, Tiffany wants to register another TOEFL test, only to find hardly any open seat in mainland China from June to September 2019. So, why not write some programs to help querying any open seat?

## 必要安装包 Dependencies
1. Python 3.6 
2. Selenium Module : 自动化测试库  
   ``pip install selenium``
3. Firefox (67.0) (x64) : 浏览器
4. geckodriver.exe (0.24) : Firefox driver for Selenium

## 开工 Start
### 观察
事实证明，新版的[教育部托福官网](https://toefl.neea.edu.cn/)对自动查询还是做了一些限制的，例如频繁的查询会强行终止。  
![托福新版报名界面](/img/in-post/toefl-page.png)

不过好在我们需要的数据量不大，可以用``time.sleep()``来避免被服务器认定频繁查询。

### 获取方式
首先我们用Selenium调用Firefox浏览器，打开登录界面。

我们发现有验证码，而且验证码相当复杂，不能轻松识别。
因此，必须采用手动登陆的方式。在手动登录期间，可以让python程序休息一会：
```py
driver.get('https://toefl.neea.edu.cn/')
time.sleep(35)  # 35秒时间手动登录
```

之前做自动抢课程序的时候，我选择模拟点击的方式，让想要的数据加载在页面上，然后去读取html源代码来得到数据。**但是这次我想Fancy一点，用上JavaScript去请求数据。**

### 获取地址和日期
查看网站源代码可以发现，网站在加载时会调用`getJSON`方法来请求某些关键数据。  
例如，获取所有托福考场信息，就靠的是下面的这句js代码：
```js
$.getJSON("getTestCenterProvinceCity")'
```
我们把js代码嵌入到Python Selenium中，就可以把返回的结果读入Python变量了（类似`Dict`的格式）：
```py
citiesJSON = driver.execute_script('return $.getJSON("getTestCenterProvinceCity")')

[in]: citiesJSON[3]
[out]: {'cities': [{'cityNameCn': '福州', 'cityNameEn': 'FUZHOU'},
  {'cityNameCn': '厦门', 'cityNameEn': 'XIAMEN'}],
 'provinceNameCn': '福建',
 'provinceNameEn': 'FUJIAN'}
```

### 获取关键信息
利用上述`getJSON`方法，可以获取关键的考场和考位信息。
```js
$.getJSON("testSeat/queryTestSeats",{city: "SHANGHAI",testDay: "2019-09-22"});
```
如果不采用Selenium，直接使用Python的Request库进行请求，则会需要把cookie值一起添加到请求中去；而cookie值的破译相当麻烦。使用`getJSON`方法即方便又自然，不容易被服务器发现。
```py
for city in citiesList[23:]:
    for date in daysList[2:13]:
        js = 'return $.getJSON("testSeat/queryTestSeats",{city: "SHANGHAI",testDay: "2019-09-22"});'
        dataJSON = driver.execute_script(js)
```

![成功发送GET请求，收到回报JSON数据](/img/in-post/toefl-request.png)

### 数据处理
将获取的某个城市、某一天的考试信息聚合到`storage`这个`DataFrame`中，并且添加一列date作为日期标识：
```py
for preciseTime, dataDetail in dataJSON['testSeats'].items():
    df = pd.DataFrame(dataDetail)
    df['date'] = date
    storage = pd.concat([storage,df],ignore_index=True)
```
最终可以获得超过2000条的数据。


## 后记
虽然最后Tiffany女士还是决定买托福黄牛考位（第一次听说托福还有黄牛），但是这个相信小工具还是可以惠及更多的出国党。  

源代码如下

```py
# -*- coding: utf-8 -*-
"""
-----------------------
@Author    Michael Wang
@GitHub    Michany
@Website   michany.xyz
-----------------------
Created on Tue Jun  1 13:24:27 2019
"""

import time
from selenium import webdriver
import pandas as pd

#%%
firefox = r'C:\Users\70242\Downloads\Chrome Download\geckodriver.exe'
driver = webdriver.Firefox(executable_path = firefox)

#%%
driver.get('https://toefl.neea.edu.cn/')
time.sleep(35)  # 35秒时间手动登录

#%% 获取地址和日期
citiesJSON = driver.execute_script('return $.getJSON("/getTestCenterProvinceCity")')

citiesList = []
for i in range(len(citiesJSON)):
    cities = citiesJSON[i]['cities']
    for city in cities:
        citiesList.append(city['cityNameEn'])


daysJSON = driver.execute_script('return $.getJSON("testDays")')
daysList = list(daysJSON)

#%% 循环获取信息
storage = pd.DataFrame()

for city in citiesList:
    for date in daysList[2:13]:
        js = 'return $.getJSON("testSeat/queryTestSeats",{city: "%s",testDay: "%s"});' % (city, date)
        dataJSON = driver.execute_script(js)
        if dataJSON is None:
            print(f"------------------ {date} Error!")
            time.sleep(30)
            continue
        elif not dataJSON['status']:
            print(city, date, "NO data")
            continue
        else:
            print(city, date, "data fetched")
        for preciseTime, dataDetail in dataJSON['testSeats'].items():
            df = pd.DataFrame(dataDetail)
            df['date'] = date
            storage = pd.concat([storage,df],ignore_index=True)
    
        time.sleep(8.88)
        
```



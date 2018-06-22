---
layout: post
title: 'Python爬虫'
date: 2018-04-16
author: Feihang Han
tags: Python
---

脚本和插件往往能提升工作效率，减少重复性工作。

# 场景

在日常工作中，为了调试代码或者定位问题，我们经常要去申请预发环境的debug端口进行远程调试。

> 由于端口有效期只有1天，很容易失效；端口申请需要老板审批，效率很低；并且很有可能同事已经申请过了。

如果有一个地方可快速的查看某个应用的debug端口申请记录，并且知道具体的端口信息，那就很舒服了。

# 分析

为了拿到端口列表，我们先要基于应用名，查询到端口申请记录，然后基于申请记录，查询到具体端口信息。

在这个过程中，有一个问题需要解决：被爬的接口是需要cookie认证身份后才能使用，否则会提示无权限。

# 思路

通过selenium模拟chrome的打开，选择浏览器证书后会自动登陆，在这个过程中可以拿到cookie。一旦拿到cookie后，后续问题就引刃而解了。

```python
def get_cookie():
    driver = webdriver.Chrome()
    driver.get('https://ops.aone.alibaba-inc.com/phoenix/home.htm?appName=' + APP_NAME)
    cookies = driver.get_cookies()
    result = ''
    for cookie in cookies:
        result = result + cookie['name'] + '=' + cookie['value'] + '; '
    return result
```

# 效果

```bash
xx@xxMacBook-Pro:~$ debug_port chatting-shopping-team
chatting-shopping-team
3802407 -> 踏古 在 2018-06-07 19:10:46 申请了debug端口: 11.185.241.218:8000-100.66.12.78:9042。
3765570 -> 踏古 在 2018-06-05 21:14:35 申请了debug端口: 11.185.241.218:8000-100.66.12.78:9609。
3738951 -> 踏古 在 2018-06-04 13:58:46 申请了debug端口: 11.185.241.218:8000-100.66.12.78:9364。
```

# 代码

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import datetime
import json
import sys
import requests
from selenium import webdriver

# 使用脚本前，需要下载与chrome浏览器版本对应的chromedriver工具，并复制到macOS的/usr/local/bin/中
# 下载地址：https://sites.google.com/a/chromium.org/chromedriver/home

COOKIE = ''
APP_NAME = 'chatting'


def get_cookie():
    driver = webdriver.Chrome()
    driver.get('https://ops.aone.alibaba-inc.com/phoenix/home.htm?appName=' + APP_NAME)
    cookies = driver.get_cookies()
    result = ''
    for cookie in cookies:
        result = result + cookie['name'] + '=' + cookie['value'] + '; '
    return result


def get_change_orders():
    url = 'https://ops.aone.alibaba-inc.com/api/changeorder.json?pageIndex=1&pageSize=20&actionPage=ALL' \
          '&orderType=ONLINEDEBUG&appName=' + APP_NAME
    response = requests.get(url)
    ret = response.text
    json_ret = json.loads(ret)
    try:
        order_id_arr = []
        orders = json_ret['data']['orders']['data']
        for order in orders:
            if order['type'] == 'ONLINEDEBUG':
                order_id_arr.append(order['id'])
        return order_id_arr
    except Exception as e:
        print('get order list fail. ' + str(e))


def get_order_detail(order_detail_id):
    headers = {'Cookie': COOKIE}
    url = 'https://ops.aone.alibaba-inc.com/paas/onlinedebug/refresh.json?id=' + str(order_detail_id)
    response = requests.get(url, headers=headers)
    ret = response.text
    if '<html>' in ret:
        print("Cookie Invalid")
        return
    json_ret = json.loads(ret)
    try:
        if json_ret['data']['taskInfo']['status'] == 'SUCCESS':
            op_name = json_ret["data"]["taskInfo"]['stepList'][0]['operatorName']
            timestamp = json_ret["data"]["debugReq"]['gmtCreate']
            port = json_ret["data"]["debugReq"]['exeResult']
            time = datetime.datetime.fromtimestamp(timestamp / 1000)
            print('%s -> %s 在 %s 申请了debug端口: %s。' % (order_detail_id, op_name, time, port.replace('\n', ' ')))
        elif json_ret['data']['taskInfo']['status'] == 'READY':
            op_name = json_ret["data"]["taskInfo"]['stepList'][0]['operatorName']
            timestamp = json_ret["data"]["debugReq"]['gmtCreate']
            time = datetime.datetime.fromtimestamp(timestamp / 1000)
            print('%s -> %s 在 %s 申请了debug端口, 待审批。' % (order_detail_id, op_name, time))
    except Exception as e:
        print('get order detail fail. ' + str(e))


if __name__ == '__main__':
    args = sys.argv
    if len(args) > 1:
        APP_NAME = args[1]
    print(APP_NAME)
    COOKIE = get_cookie() // 获取cookie
    order_ids = get_change_orders() // 查询申请记录
    for order_id in order_ids[:3]: 
        get_order_detail(order_id) // 根据记录id查询端口信息

```




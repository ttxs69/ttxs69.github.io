---

title: Python-send-ip-to-wechat
date: 2019-07-31 14:29:43
categories: 
 - Python
---
# Python-send-ip-to-wechat

实现思路：
1、利用python获取ip
2、通过第三方平台向微信发送消息
采用api：
“[Pushbear](http://pushbear.ftqq.com/)”–基于微信模板的一对多消息送达服务

具体代码如下：

```python
import socket
import requests


def get_host_ip():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)        
        s.connect(('8.8.8.8', 80))        
        ip = s.getsockname()[0] 
    finally:        
        s.close() 
    return ip


def send_ip(ip): 
    try:        
        url = "https://pushbear.ftqq.com/sub"        
        data = {'sendkey': '这里填写你自己的key', 'text': '树莓派ip', 'desp': ip}        
        r = requests.post(url, data=data) 
        print(r.text) 
    except: 
        print("failed")
        send_ip(get_host_ip())
```
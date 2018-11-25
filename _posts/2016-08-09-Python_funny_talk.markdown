---
layout: post
title:  "微信开发：一个智能陪聊公众号"
categories: Python 
---

首先感慨微信生态之丰富，功能之强大！

起初是为了方便女友，开发了一个公众号，用于查询天气、快递等，之后逐步扩展功能，如：笑话、绕口令、星座运势和智能聊天等。步骤如下：

- [微信公众平台](open.weixin.qq.com)注册开发账号，注册订阅号。
- 获取一台带公网 IP 的云主机，本人使用的是 AWS，最低配置免费一年的那种。
- 某些开放平台提供了许多免费 API，如查询天气、笑话等，如[极速数据](http://www.jisuapi.com/my/)，[百度开放平台](http://apistore.baidu.com/astore/servicesearch?word=APIStore)。

效果如下：

![](http://wsfdl.oss-cn-qingdao.aliyuncs.com/weichat_xinwen.jpeg)

![](http://wsfdl.oss-cn-qingdao.aliyuncs.com/weichat_tianqi.jpeg)

![](http://wsfdl.oss-cn-qingdao.aliyuncs.com/weichat_tangshi.jpeg)

![](http://wsfdl.oss-cn-qingdao.aliyuncs.com/weichat_funny.jpeg)


代码如下：

server.py

~~~ python
# coding: utf-8

import random

import eventlet
from eventlet import wsgi
import request
import xml.etree.ElementTree as ET



class FunnyTalkServer(object):

    def application(self, env, start_response):
        input_msg = env['eventlet.input'].read()
        res = self.process_request(input_msg)
        start_response('200 OK', [('Content-Type', 'text/xml')])
        return [res]

    def serve_forever(self):
        wsgi.server(eventlet.listen(('', 80)), self.application)

    def process_request(self, msg):
        root = ET.fromstring(msg)
        children = root.getchildren()
        for child in children:
            if child.tag == 'ToUserName':
                to_user_name = child.text
            if child.tag == 'FromUserName':
                from_user_name = child.text
            if child.tag == 'CreateTime':
                child.text = str(int(child.text) + 1)
            if child.tag == 'MsgId':
                root.remove(child)
            if child.tag == 'Content':
                child.text = self.create_response(child.text)

        for child in children:
            if child.tag == 'ToUserName':
                child.text = from_user_name
            if child.tag == 'FromUserName':
                child.text = to_user_name

        return ET.tostring(root, encoding='utf8')

    def create_response(self, req_str):

        # 20% ratio for random response.
        if random.random() < 0.2:
            return self.random_funny_response()

        req_cmd = self.pre_format_req(req_str)
        if u'笑话' in req_cmd:
            req = request.RequestJoke()
        elif u'天气' in req_cmd:
            if len(req_cmd) == 1:
                city = u'上海'
            else:
                city = req_cmd[1]
            req = request.RequestWeather(city)
        elif u'绕口令' in req_cmd:
            if len(req_cmd) == 1:
                return u'请输入关键字，如：笑话 西瓜'
            else:
                keyword = req_cmd[1]
            req = request.RequestTongueTwister(keyword)
        elif u'快递' in req_cmd:
            if len(req_cmd) == 1:
                return u'请输入快递单，如：快递 121212121212'
            else:
                number = req_cmd[1]
            req = request.RequestExpress(number)
        else:
            req = request.RequestAuto(req_str)

        response = req.make_req()
        if response:
            return self.post_format_res(response)
        else:
            return u"服务器罢工中!"

    def pre_format_req(self, req_str):
        req_cmd = []
        req_list = req_str.split(' ')
        for req in req_list:
            if req not in ('', ' '):
                req_cmd.append(req)
        return req_cmd

    def random_funny_response(self):
        funny_responses = [
            u'宝宝别闹！',
            u'大爷在睡觉觉！',
            u'我知道你想要什么，可是我就是不给你，嘿嘿！',
            u'大爷，服务很辛苦的，赏点钱吧！',
            u'再试试！！',
            u'老板好久没有发钱了，拒绝服务！！！',
            u'认真学习！！！',
            u'早点休息！！！',
            u'来来来，给爷笑一个！！！']
        length = len(funny_responses)

        return funny_responses[random.randint(0, length - 1)]

    def post_format_res(self, res):
        res = res.replace(u'<br />', u'')
        res = res.replace(u'&rdquo;&ldquo', u'\n')
        res = res.replace(u'&ldquo;', u'\n')
        res = res.replace(u'&rdquo;', u'\n')
        res = res.replace(u'&hellip;', u'。')
        return res


if '__main__' == __name__:
    server = FunnyTalkServer()
    server.serve_forever()
~~~

request.py

~~~ python
# coding: utf-8

import json
import requests


TRY_AGAIN_LATER = "Please try again later!"
ANOTHER_KEYWORD = "Not found, please try another keyword!"
ANOTHER_CITY = "Not found, please try another city!"
ANOTHER_NUMBER = "Not found, please try express number!"
STATUS_OK = (200, 201, 202, 204)


class BaseRequest(object):

    def __init__(self):
        self.app_key = "appkey"

    def send_req(self, method, url, headers=None, data=None):
        try:
            conn = requests.request(method, url,
                                    headers=headers, data=data)
            response = json.loads(conn.text)
        except Exception:
            return None

        return response


class RequestJoke(BaseRequest):

    def __init__(self):
        super(RequestJoke, self).__init__()
        self.base_url = "http://api.jisuapi.com/xiaohua/text?"
        self.url = self.base_url + "pagenum=1&pagesize=1&sort=rand"
        self.url += "&appkey=" + self.app_key

    def make_req(self):
        res = self.send_req('GET', self.url)

        if res and res.get('status') == '0':
            result = res['result']['list'][0]['content']
            return result
        else:
            return TRY_AGAIN_LATER

class RequestTongueTwister(BaseRequest):

    def __init__(self, keyword):
        super(RequestTongueTwister, self).__init__()
        self.base_url = "http://api.jisuapi.com/rkl/search?"
        self.url = self.base_url + "pagenum=1&pagesize=1"
        self.url += "&appkey=" + self.app_key
        self.url += "&keyword=" + keyword

    def make_req(self):
        res = self.send_req('GET', self.url)

        if res:
            if res.get('status') == '0':
                title = res['result']['list'][0]['title']
                content = res['result']['list'][0]['content']
                result = title + '   ' + content
                return result
            elif res.get('status') == '202':
                return ANOTHER_KEYWORD
        else:
            return TRY_AGAIN_LATER


class RequestWeather(BaseRequest):

    def __init__(self, city):
        super(RequestWeather, self).__init__()
        self.base_url = "http://api.jisuapi.com/weather/query?"
        self.url = self.base_url + "&appkey=" + self.app_key
        self.url += "&city=" + city

    def make_req(self):
        res = self.send_req('GET', self.url)

        if res:
            if res.get('status') == '0':
                return self.format_weather_info(res['result'])
            elif res.get('status') in ('202', '203'):
                return ANOTHER_CITY
        else:
            return TRY_AGAIN_LATER

    def format_weather_info(self, info):
        result = info['city'] + '    ' + info['weather']
        result += u'    温度: ' + info['templow'] + u'-' + info['temphigh']
        result += u'    ' + info['winddirect'] + u' ' + info['windpower']
        result += u'    PM2.5: ' + info['aqi']['ipm2_5']
        result += u'    温馨提醒宝宝: ' + info['index'][1]['detail']
        return result


class RequestExpress(BaseRequest):

    def __init__(self, number):
        super(RequestExpress, self).__init__()
        self.base_url = "http://api.jisuapi.com/express/query?type=auto"
        self.url = self.base_url + "&appkey=" + self.app_key
        self.url += "&number=" + number

    def make_req(self):
        res = self.send_req('GET', self.url)

        if res:
            if res.get('status') == '0':
                result = ''
                contents = res['result']['list']
                for content in contents:
                    result += content['time'] + ' '
                    result += content['status'] + '   '
                return result
            elif res.get('status') in ('202', '203'):
                return ANOTHER_NUMBER
        else:
            return TRY_AGAIN_LATER


class RequestAuto(BaseRequest):

    def __init__(self, question):
        super(RequestAuto, self).__init__()
        self.base_url = "http://api.jisuapi.com/iqa/query?"
        self.url = self.base_url + "&appkey=" + self.app_key
        self.url += "&question=" + question

    def make_req(self):
        res = self.send_req('GET', self.url)

        if res and res.get('status') == '0':
            result = res['result']['content']
            return result

        return None
~~~

运行：

~~~ bash
$ sudo nohup python server.py >/dev/null 2>&1 &
~~~

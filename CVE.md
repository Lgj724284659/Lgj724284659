#!/usr/bin/python3
# -*- coding:utf-8 -*-
# @Author : yhy

# 每3分钟检测一次github是否有新的cve漏洞提交记录，若有则通过server酱和钉钉机器人推送（二者配置一个即可）
# 建议使用screen命令运行在自己的linux vps后台上，就可以愉快的接收各种cve了

# https://my.oschina.net/u/4581868/blog/4380482
# https://github.com/kiang70/Github-Monitor

import requests, re, time
import dingtalkchatbot.chatbot as cb
import datetime


def getNews():
    try:
        # 抓取本年的
        year = datetime.datetime.now().year
        api = "https://api.github.com/search/repositories?q=CVE-{}&sort=updated".format(year)
        req = requests.get(api).text
        cve_total_count = re.findall('"total_count":*.{1,10}"incomplete_results"',req)[0][14:17]
        cve_description = re.findall('"description":*.{1,200}"fork"',req)[0].replace("\",\"fork\"",'').replace("\"description\":\"",'')
        cve_url = re.findall('"svn_url":*.{1,200}"homepage"',req)[0].replace("\",\"homepage\"",'').replace("\"svn_url\":\"",'')

        return cve_total_count, cve_description, cve_url

    except Exception as e:
        print (e, "github链接不通")

# 钉钉
def dingding(text, msg):
    # 将此处换为钉钉机器人的api
    webhook = 'xxxxx'
    ding = cb.DingtalkChatbot(webhook)
    ding.send_text(msg = '{}\r\n{}'.format(text, msg), is_at_all=False)

# server酱  http://sc.ftqq.com/?c=code
def server(text, msg):
    # 将 xxxx 换成自己的server SCKEY
    uri = 'https://sc.ftqq.com/xxxx.send?text={}&desp={}'.format(text, msg)
    requests.get(uri)


# 通过检查name 和 description 中是否存在test字样，排除test
def regular(req):
    cve_name = re.findall('"name":*.{1,200}"full_name"', req)[0].replace("\"name\":\"",'').replace("\",\"full_name\"",'')
    cve_description = re.findall('"description":*.{1,200}"fork"', req)[0].replace("\",\"fork\"", '').replace(
        "\"description\":\"", '')

    if cve_name.lower().find('test') == -1 and cve_description.lower().find('test') == -1:
        return True
    return False

def sendNews():
    while True:
        try:
            print("cve 监控中 ...")
            # 抓取本年的
            year = datetime.datetime.now().year
            api = "https://api.github.com/search/repositories?q=CVE-{}&sort=updated".format(year)
            # 请求API
            req = requests.get(api).text
            # 正则获取
            total_count = re.findall('"total_count":*.{1,10}"incomplete_results"', req)[0][14:17]

            # 监控时间间隔3分钟
            time.sleep(180)
            # 推送正文内容
            msg = str(getNews())
            # 推送标题
            text = r'有新的CVE送达！'
            regular(req)
            # 检查name 和 description 中是否存在test字样 和 是否更新
            if regular(req) and total_count != getNews()[0]:
                # 二选一即可，没配置的 注释或者删掉
                server(text, msg)
                dingding(text, msg)
                print(msg)
            else:
                pass
        except Exception as e:
            raise e


if __name__ == '__main__':
    sendNews()

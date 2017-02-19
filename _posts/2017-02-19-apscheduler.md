---
date: 2017-02-19
layout: post
title: apscheduler定时任务器的使用（tornado场景）
thread: 2017-02-19-apscheduler.md
categories: python
tags: apscheduler
---

</br>
1.七种scheduler
</br>
> BlockingScheduler: 当应用程序中只有调度器时使用

> BackgroundScheduler: 不使用任何以下框架：asyncio, gevent, Tornado, Twisted, Qt, 并且需要在你的应用程序后台运行调度程序

> AsyncIOScheduler: 应用程序使用asyncio模块时使用

> GeventScheduler: 应用程序使用gevent模块时使用程序

> TornadoScheduler: Tornado应用程序使用

> TwistedScheduler: Twisted应用程序使用

> QtScheduler: Qt应用程序使用
</br>
</br>

2.Tornado中使用的demo
</br>
> sched = TornadoScheduler() 可以作为全局变量，在项目任何引用的地方操作调度器

> sched.start() 是调度器真正开始执行的入口

> sched.start() 之前 sched.add_job缓存的任务会开始调度，之后 sched.add_job的任务立即调度（达到执行时间的时候执行任务） 

> scheduler.shutdown(wait=True) 默认情况下调度器会等待所有正在运行的作业完成后，关闭所有的调度器和作业存储。如果你不想等待，可以将wait选项设置为False。

> scheduler.add_job() 第二个参数是trigger，它管理着作业的调度方式。它可以为date, interval或者cron。对于不同的trigger，对应的参数也相同。\

> trigger:
> </br>
> 1) interval 间隔调度
> </br>
> 循环执行，间隔固定的时间
> <pre>
> \# Schedule job_function to be called every two hours
> sched.add_job(job_function, 'interval', hours=2)
> </pre>
> </br>
> 2) date 定时调度
> </br>
> 只会执行一次，指定时间执行后就结束
> <pre>
> \# The job will be executed on November 6th, 2009
> sched.add_job(my_job, 'date', run_date=date(2009, 11, 6), args=['text'])
> 
> \# The job will be executed on November 6th, 2009 at 16:30:05
> sched.add_job(my_job, 'date', run_date=datetime(2009, 11, 6, 16, 30, 5), args=['text'])
> </pre>
> </br>
> 3) cron 定时调度
> </br>
> 闹钟似执行，设定计划时间表执行
> <pre>
> \# Schedules job_function to be run on the third Friday
> \# of June, July, August, November and December at 00:00, 01:00, 02:00 and 03:00
> sched.add_job(job_function, 'cron', month='6-8,11-12', day='3rd fri', hour='0-3')
> 
> \# Runs from Monday to Friday at 5:30 (am) until 2014-05-30 00:00:00
> sched.add_job(job_function, 'cron', day_of_week='mon-fri', hour=5, minute=30, end_date='2014-05-30')
> </pre>
> </br>

<pre>
# -*- coding:utf8 -*-

import tornado.httpserver
import tornado.ioloop
import tornado.options
import tornado.web
from tornado.options import define, options
define("port", default=8888, help="run on the given port", type=int)
from apscheduler.schedulers.tornado import TornadoScheduler
sched = TornadoScheduler()

def my_job():
    print 1

def my_job2():
    print 2
    print sched.get_jobs()

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, guys")


class PauseJobHandler(tornado.web.RequestHandler):
    def get(self):
        sched.pause_job('1')
        self.write("pause")

class ResumeJobHandler(tornado.web.RequestHandler):
    def get(self):
        sched.resume_job('1')
        self.write("resume")

class AddJobHandler(tornado.web.RequestHandler):
    def get(self):
        sched.add_job(my_job, 'interval', seconds=1, id='1')
        sched.add_job(my_job2, 'interval', seconds=1, id='2')
        self.write("add job")

class RemoveJobHandler(tornado.web.RequestHandler):
    def get(self):
        sched.remove_job('2')
        self.write("remove job: %s" % 2)

class RemoveAllJobsHandler(tornado.web.RequestHandler):
    def get(self):
        sched.remove_all_jobs()
        self.write("remove all jobs")


class StartHandler(tornado.web.RequestHandler):
    def get(self):
        sched.start()
        result = sched.running
        self.write("start scheduler: %s" % result)

class ShutdownSchedHandler(tornado.web.RequestHandler):
    def get(self):
        sched.shutdown()
        self.write("shutdown scheduler")

class PauseSchedHandler(tornado.web.RequestHandler):
    def get(self):
        sched.pause()
        self.write("pause scheduler")

class ResumeSchedHandler(tornado.web.RequestHandler):
    def get(self):
        sched.resume()
        self.write("resume scheduler")


def main():
    tornado.options.parse_command_line()
    application = tornado.web.Application([
        # home
        (r"/", MainHandler),
        # scheduler
        (r"/start_sched", StartHandler),
        (r"/shutdown_sched", ShutdownSchedHandler),
        (r"/pause_sched", PauseSchedHandler),
        (r"/resume_sched", ResumeSchedHandler),
        # job
        (r"/add_job", AddJobHandler),
        (r"/pause_job", PauseJobHandler),
        (r"/resume_job", ResumeJobHandler),
        (r"/remove_job", RemoveJobHandler),
        (r"/remove_all_jobs", RemoveAllJobsHandler),
    ])
    http_server = tornado.httpserver.HTTPServer(application)
    http_server.listen(options.port)
    tornado.ioloop.IOLoop.instance().start()

if __name__ == "__main__":
    main()

</pre>


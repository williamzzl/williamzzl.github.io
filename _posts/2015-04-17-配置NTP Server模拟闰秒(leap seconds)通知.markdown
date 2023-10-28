---
layout: post
title:  "配置NTP Server模拟闰秒(leap seconds)通知"
date:   2015-04-17 10:30:25
categories: Linux
tags: Linux NTP
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

# 配置NTP Server模拟闰秒(leap seconds)通知

###背景
由于UTC时间和GPS时间的不完全一致，每隔一定时间系统需要增加一秒做调整。最近的一次将发生在2015-06-30 23:59:59(GMT)。如果使用date查看，届时会如下出现两个59秒(或多一个60秒):

    [root@linux ~]# while true ; do date; sleep 1 ; done
    Tue Jun 30 23:59:58 GMT 2015
    Tue Jun 30 23:59:59 GMT 2015
    Tue Jun 30 23:59:59 GMT 2015
    Wed Jul  1 00:00:00 GMT 2015
    Wed Jul  1 00:00:01 GMT 2015


###需求
为了测试系统对leap seconds的行为，需要配置NTP server来模拟GPS Server对闰秒时间做出通知。

###配置NTP Server
参考[ntp org]，对于4.2.6以上的NTP Server版本，放置文件[leap-seconds.3629404800]在NTP Server(如`/etc/ntp/crypto/`)，并且将路径配置在/etc/ntp.conf中：

    [root@linux ~]# less /etc/ntp.conf
    leapfile  /etc/ntp/crypto/leap-seconds.3629404800
    
>对于4.2.6以前版本，需要[开启autokey]功能，并配置相应的leapsecond信息。但比较复杂，所以建议先尝试升级NTP版本（只需要升级Server端版本）。

将系统时间调整到leapsecond时间发生时间附近(对于GMT时间，执行以下命令)：

    [root@linux ~]# date --set="20150630 23:50:00"

等待client完成对Server的同步后，可以看到Client的时间在23:59:59之后会增加一秒（59秒或60秒）。并且Server端的log会显示leap event通知：

    [root@linux ~]# tail -f /var/log/ntp
    30 Jun 23:50:04 ntpd[18017]: 0.0.0.0 c01e 0e TAI 36 leap 201507010000 expire 201512280000
    30 Jun 23:50:04 ntpd[18017]: 0.0.0.0 c016 06 restart
    30 Jun 23:50:04 ntpd[18017]: 0.0.0.0 c012 02 freq_set kernel 0.000 PPM
    30 Jun 23:50:05 ntpd[18017]: 0.0.0.0 c515 05 clock_sync
    30 Jun 23:50:05 ntpd[18017]: 0.0.0.0 0519 09 leap_armed
    30 Jun 23:59:59 ntpd[18017]: 0.0.0.0 051b 0b leap_event

[ntp org]:http://support.ntp.org/bin/view/Support/ConfiguringNTP#Section_6.14.
[leap-seconds.3629404800]:ftp://time.nist.gov/pub/leap-seconds.3629404800
[开启autokey]:http://support.ntp.org/bin/view/Support/ConfiguringAutokeyFourTwoFour

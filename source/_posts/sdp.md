---
title: 会话描述协议
date: 2017-08-03 18:35:49
categories: 网络协议
tags:
- 网络协议
- 翻译
---

**会话描述协议 (SDP)** 是一种描述流媒体通信参数的格式。最初的规范是 [IETF](https://en.wikipedia.org/wiki/IETF) 以  [IETF 建议标准](https://en.wikipedia.org/wiki/Internet_Standard) 的形式于 1998 年四月发布的，后来于2006 年七月发布了一个修订版规范，为 IETF 建议标准  [RFC 4566](https://tools.ietf.org/html/rfc4566) 。

SDP旨在描述用于会话通知、会话邀请和参数协商目的的多媒体通信会话。SDP 自身不传送任何媒体，但它被用于端点之间的媒体类型，格式和所有相关属性的协商。属性和参数的集合经常被称作 *会话规范*。

<!--more-->

SDP 旨在可扩展以支持新的媒体类型和格式。SDP 最初是[会话通知协议](https://en.wikipedia.org/wiki/Session_Announcement_Protocol) (SAP) 的一个组件，但通过和 [Real-time Transport Protocol](https://en.wikipedia.org/wiki/Real-time_Transport_Protocol) (RTP)，[Real-time Streaming Protocol](https://en.wikipedia.org/wiki/Real-time_Streaming_Protocol) (RTSP)，[Session Initiation Protocol](https://en.wikipedia.org/wiki/Session_Initiation_Protocol) (SIP) 结合发现了其它用途，甚至作为一个独立的格式来描述多播会话。

# 会话描述
会话通过一系列字段描述，每个一行。每个字段的格式如下：
```
<character>=<value>
```
其中 `<character>` 是单个大小写敏感的字符，而 `<value>` 是格式依赖于属性类型的结构化文本。典型的值是一个 UTF-8 编码的字符串。`=` 的两边不能紧挨着空格。

一个 SDP 消息内有三个主要的段，详细描述了 *会话*，*时序* 和 *媒体*。每个消息可以包含多个 *时序* 和 *媒体* 描述。关联的语法结构内名字是唯一的，比如，在 *会话*、*时序* 和 *媒体* 内。

可选的值通过 `=*` 指定，且每个字段必须以下面展示的顺序出现。

**Session description**
```
    v=  (protocol version number, currently only 0)
    o=  (originator and session identifier : username, id, version number, network address)
    s=  (session name : mandatory with at least one UTF-8-encoded character)
    i=* (session title or short information)
    u=* (URI of description)
    e=* (zero or more email address with optional name of contacts)
    p=* (zero or more phone number with optional name of contacts)
    c=* (connection information—not required if included in all media)
    b=* (zero or more bandwidth information lines)
    One or more Time descriptions ("t=" and "r=" lines; see below)
    z=* (time zone adjustments)
    k=* (encryption key)
    a=* (zero or more session attribute lines)
    Zero or more Media descriptions (each one starting by an "m=" line; see below)
```

**Time description** (mandatory)
```
    t=  (time the session is active)
    r=* (zero or more repeat times)
```

**Media description** (if present)
```
    m=  (media name and transport address)
    i=* (media title or information field)
    c=* (connection information — optional if included at session level)
    b=* (zero or more bandwidth information lines)
    k=* (encryption key)
    a=* (zero or more media attribute lines — overriding the Session attribute lines)
```

下面是  [RFC 4566](https://tools.ietf.org/html/rfc4566) 的一个简单会话描述。这个会话由用户 'jdoe' 发起，位于 IPv4 地址 10.47.16.5 处。它的名字为 "SDP Seminar"，还包含了扩展的会话信息 ("A Seminar on the session description protocol") ，一个用于获取额外信息的链接，及负责聚会的联系人 Jane Doe 的 E-Mail 地址。该会话被指定为使用 NTP 时间戳持续两个小时，其连接地址（它表示客户端必须连接的地址 - 或者提供一个多播地址，就像在这里 - 订阅的地址）指定为具有 TTL 值为 127 的 IPv4 224.2.17.12。这个会话描述的接收者被指导只接收媒体。提供了两个媒体描述，都使用 RTP Audio Video Profile。第一个是端口 49170 上的音频流，使用 RTP/AVP 载荷类型 0（由 [RFC 3551](http://tools.ietf.org/html/rfc3551) 定义为 PCMU），第二个是端口 51372 上的视频流，使用 RTP/AVP 载荷类型 99 （定义为 "dynamic"）。最后，包含了一个属性，它将 RTP/AVP 载荷类型 99 映射为 90kHz 时钟频率的格式 h263-1998。分别用于音频和视频流的 RTCP 端口分别为 49171 和 51373。
```
    v=0
    o=jdoe 2890844526 2890842807 IN IP4 10.47.16.5
    s=SDP Seminar
    i=A Seminar on the session description protocol
    u=http://www.example.com/seminars/sdp.pdf
    e=j.doe@example.com (Jane Doe)
    c=IN IP4 224.2.17.12/127
    t=2873397496 2873404696
    a=recvonly
    m=audio 49170 RTP/AVP 0
    m=video 51372 RTP/AVP 99
    a=rtpmap:99 h263-1998/90000
```

SDP 规范不包含任何传输协议; 它纯粹是会话描述的格式。旨在根据需要使用不同的传输协议，包括 [SAP](https://en.wikipedia.org/wiki/Session_Announcement_Protocol)，[SIP](https://en.wikipedia.org/wiki/Session_Initiation_Protocol)，和 RTSP。SDP 甚至可以通过 email 或作为 HTTP 载荷传输。

# 属性
SDP 使用属性扩展核心协议。属性可以出现在 Session 或 Media 段内，并作为会话级别或媒体级别进行范围限定。新的属性偶尔会通过 IANA 注册被添加到标准中来。

属性有两种形式：
 * 属性形式：`a=flag` 表达媒体或会话的一个简单的布尔属性。
 * 值形式：`a=attribute:value` 提供了一个命名参数。

其中两个属性被特别定义：
 * `a=charset:encoding`
 * `a=sdplang:code`

第一个被用在会话和媒体段，用于指定另一种字符编码（在 IANA 注册表中注册）而不是默认的强烈建议的那个（[UTF-8](https://en.wikipedia.org/wiki/UTF-8)），UTF-8被用于标准的协议键中，键的值包含一个用于显示给用户的文本。第二个用于指定编写所用的是哪种语言（可以在协议中携带多种语言的文本，并由用户代理根据用户偏好自动选择。在这些情况中，协议中的每个文本字段不由协议本身符号化地解释，将被解释为透明字符串，但是交付给当前 Media 段中最后一次出现的 `charset` 和 `sdplang`，或者是 Section 段中它们最后的值，表示的用户或应用程序）。

注意，前三个强制参数（`v=`，`s=` 和 `o=`），尽管它们看起来包含了可显示的文本，但不是用来显示给用户或翻译的。它们的值中出现的字段，在协议中被认为是透明的字符串，它们被用作标识符，就像 URL 中的路径或文件系统中的文件名：SDP 标准表示它们必须都是非空的，且应该以 UTF-8 编码。

上面的例子中也展示了一些其它的属性（在相同的 RFC 中被描述为标准 SDP 规范的一部分），作为会话级属性（比如属性形式的属性 `a=recvonly`），它们也应用于描述媒体，除非它们覆盖了那些值，或作为媒体级属性（比如示例中视频媒体的值形式的属性 `a=rtpmap:99 h263-1998/90000` ）。

# 时间格式和重复
 [网络时间协议](https://en.wikipedia.org/wiki/Network_Time_Protocol) (NTP) 格式表示的绝对时间 （自 1900 年所经过的秒数）如果停止时间为 0，则会话是 “无限的”。如果开始时间也是 0，则会话被认为是“永久的”。无限的和永久的会话不鼓励，但也不禁止。时间间隔可由 [网络时间协议](https://en.wikipedia.org/wiki/Network_Time_Protocol) 时间或类型时间表示：值和时间单元 (日 ('d')，时 ('h')，分 ('m') 和 秒 ('s')) 序列。

这样从 UTC 2010 年 8 月 1 日上午 10 点开始的一个小时的会议，且在一周后有一个单独的相同时间的重复时间，可以被表示为：

```
        t=1280656800 1281265200
        r=604800 3600 0
```

或使用类型时间：
```
        t=1280656800 1281265200
        r=7d 1h 0
```

当指定重复次数时，可能需要调整每个重复的开始时间，以便在开始时间和停止时间之间的整个时间段内，在特定时区的相同本地时间发生（同样以 NTP 格式的绝对 UTC [时区](https://en.wikipedia.org/wiki/Timezone) 描述）。

不是指定这个时区，并支持一个时区数据库来了解何时何地需要夏令时调整，重复时间假设都在相同的时区内定义，当夏令时偏移（以秒或使用类型时间表示）需要应用于每个夏令时调整之后的重复的开始时间或结束时间时 SDP 支持 NTP 绝对时间表示。所有的这些偏移是相对于开始时间的，它们是不累积的。NTP 通过 `z=` 字段支持它，其表示一系列对，其中的第一个项是当夏令时调整发生时的 NTP 绝对时间，第二个项表示应用相对于由 `r=` 字段计算的绝对时间的值的偏移。

比如，如果夏令时调整将把 31 October 2010 at 03am UTC 减 1（比如，开始时间的 Sunday 1 August 2010 at 10am UTC 之后，60 天减去 7 小时），这将是仅有的应用于将在 1 August 2010 到  28 November 2010 at 10am UTC（每周在同一时间重复的 1 小时的会话的停止时间，发生在88天后） 发生的调度周期内的夏令时调整，这可以被描述为：
```
        t=1280656800 1290938400
        r=7d 1h 0
        z=1288494000 -1h
```

If the weekly 1-hour session was repeated every Sunday for full one year, i.e. from Sunday 1 August 2010 03am UTC to Sunday 26 June 2011 04am UTC (stop time of the last repeat, i.e. 360 days plus 1 hour later, or 31107600 seconds later), so that it would include the transition back to Summer time on Sunday 27 March 2011 at 02am (1 hour is added again to local time, so that the second daylight transition would occur 209 days after the first start time):
```
        t=1280656800 1290938400
        r=7d 1h 0
        z=1288494000 -1h 1269655200 0
```
As SDP announcements for repeated sessions should not be made to cover very long periods exceeding a few years, the number of daylight adjustments to include in the z= parameter should remain small.
Note also that sessions may be repeated irregularly over a week but scheduled the same way for all weeks in the period, by adding more tuples in the r parameter. For example, to schedule the same event also on Saturday (at the same time of the day) you would use :
```
        t=1280656800 1290938400
        r=7d 1h 0 6d
        z=1288494000 -1h 1269655200 0
```
The SDP protocol does not support repeating sessions monthly and yearly schedules with such simple repeat times, because they are irregularly spaced in time; instead, additional t/r tuples may be supplied for each month or year.

# 参考资料
[H264(NAL简介与I帧判断)](http://blog.csdn.net/jefry_xdz/article/details/8461343)

[H.264 RTP PAYLOAD 格式](http://www.cppblog.com/czanyou/archive/2008/11/26/67940.html)

[用实例分析H264 RTP payload](http://blog.csdn.net/zblue78/article/details/5948538)

[sdp文件详细总结](http://blog.csdn.net/tongjing524/article/details/49635065)

[ORTP移植到Hi3518e，h.264封包rtp发送](http://www.aiuxian.com/article/p-1981030.html)

[H264编码 封装成MP4格式 视频流 RTP封包](http://blog.csdn.net/crazyman2010/article/details/8596229)

[RTP 协议](http://www.cnblogs.com/huaping-audio/archive/2009/04/22/1441285.html)

[RTP/RTCP/RTSP/SIP/SDP](http://www.cnblogs.com/whyandinside/archive/2009/08/30/1556572.html)

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done。

[原文](https://en.wikipedia.org/wiki/Session_Description_Protocol)

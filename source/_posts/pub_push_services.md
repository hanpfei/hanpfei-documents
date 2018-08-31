---
title: 第三方推送服务
date: 2018-08-31 19:37:49
categories: Android开发
tags:
- Android开发
---

## 通知栏消息和透传消息

第三方推送服务（小米、华为和魅族）都支持两种类型的推送消息，分别是通知栏消息和透传消息，这两种类型的推送消息，在用户端的行为、消息的格式和到达率等许多方面存在一定的差异。这几个第三方推送服务关于透传消息和通知栏消息的说明如下。
<!--more-->
[华为透传消息和通知栏消息说明](https://developer.huawei.com/consumer/cn/service/hms/catalog/huaweipush_agent.html?page=hmssdk_huaweipush_devguide_server_agent)：

 * 通知栏消息：所谓通知栏消息是指消息通过Push平台发送到Push客户端的时候使用华为默认的消息呈现和点击动作（点击后是需要应用处理的）。
 * 透传消息：所谓透传消息是指消息通过Push平台发送到Push客户端的时候会透传给对应的App，由App自己控制消息呈现方式和点击动作。

[魅族透传消息和通知栏消息说明](http://open.res.flyme.cn/fileserver/upload/file/201612/728a49f530c64c5a832d7ba1de69e356.pdf)：

 * 通知栏消息：通知栏通知类型，由 SDK 展示及统计展示数、点击数。
 * 透传消息：此消息 SDK 不解析，直接透传给应用处理，SDK 不负责后续的统计。

为优化flyme系统整体功耗，魅族推送平台 6 月 16 号起限制透传消息推送的使用，不排除关闭透传推送类型。（[说明文档地址](http://open.res.flyme.cn/fileserver/upload/file/201803/e174a5709f134f64aae3fb168aec8ea3.pdf)）

[小米透传消息和通知栏消息说明](https://dev.mi.com/doc/?p=544)：

小米推送目前支持两种消息传递方式：透传方式和通知栏方式。透传消息到达手机端后，SDK会将消息通过广播方式传给AndroidManifest中注册的PushMessageReceiver的子类的onReceivePassThroughMessage；对于通知栏消息，SDK会根据消息中设置的信息弹出通知栏通知，通知消息到达时会到达PushMessageReceiver子类的onNotificationMessageArrived方法,用户点击之后再传给您的PushMessageReceiver的子类的onNotificationMessageClicked方法；对于应用在前台时不弹出通知的通知消息，SDK会将消息通过广播方式传给AndroidManifest中注册的PushMessageReceiver的子类的onNotificationMessageArrived方法(在MIUI上，如果没有收到onNotificationMessageArrived回调，是因为使用的MIUI版本还不支持该特性，需要升级到MIUI7之后。非MIUI手机都可以收到这个回调)。

[小米透传消息和通知栏消息说明二](https://dev.mi.com/doc/?p=7674)：

首先解释一下透传和通知栏方式的原理，透传是指当小米推送服务客户端SDK接收到消息之后，直接把消息通过回调方法发送给应用，不做任何处理；而通知栏方式，则在设备接收到消息之后，首先由小米推送服务SDK弹出标准安卓通知栏通知，在用户点击通知栏之后，激活应用。 在非MIUI系统中，由于维护小米推送服务长连接的service是寄生在App的运行空间当中的，因此透传和通知栏方式在送达率上并没有任何区别，都需要应用驻留在后台。即，如果一台设备通知栏消息能够接收到并弹出，那么其透传消息也同样能接收到。 在MIUI系统中，由于长连接是由MIUI系统服务建立并维护的，因此在接收消息的时候并不需要应用驻留后台。如果采用通知栏方式接收消息，由于通知栏也是MIUI系统服务弹出的，就可以做到不需要用户后台驻留或者可以自启动消息就能送达。而如果采用透传消息，由于需要直接执行应用的代码，因此即使消息已经到了系统服务，如果应用没有驻留后台或者能自启动，消息依然不能送达，需等下次用户手动点击激活应用后，才能接收到消息。 综上，在MIUI系统中，通知栏消息的送达率会远高于透传方式；在非MIUI系统中，通知栏和透传方式的送达率是一样的。

总结：

| 消息类型 | 展示样式 |消息数据格式 | 到达率 |
|--------------|-------------|-------------|-------------|
| 通知栏消息 | 由第三方推送服务提供固定的消息展示样式 | 数据格式基本固定，但可携带一些完制的数据 | 较好，不会比透传消息差 |
| 透传消息 | 完全由 App 控制 | 数据格式自由可定制 | 不太好，魅族甚至宣称未来可能不再支持这种消息类型 |

## 收费情况

各个主要设备厂商没有将推送服务当作一项主营的需要盈利的业务在做，因而基本上不收费，这些推送服务的具体收费情况如下：

| 推送服务 | 收费情况 | 官方说明（信息出处） |
|--------------|-------------|-------------|
| 小米推送 | 小米推送的基础服务目前是免费的。 | [推送服务常见问题](https://dev.mi.com/doc/p=1608/index.html) |
| 魅族推送 | Flyme推送平台的基础推送功能是免费的，但是某些定制功能会考虑收费，在功能和体验上会有不同。 | [魅族推送平台常见问题](http://open-wiki.flyme.cn/index.php?title=%E9%AD%85%E6%97%8F%E6%8E%A8%E9%80%81%E5%B9%B3%E5%8F%B0%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98) |
| 华为推送 | 不收费 | 没找到官方的正式声明，收费情况通过华为的相关人员确认 |

## 小米消息推送

小米消息推送[官方文档地址](https://dev.mi.com/doc/p=6421/index.html)，其中包含了整个推送系统的说明，各端的接入说明，FAQ 等内容。

### 推送消息结构

由于透传消息的到达率、兼容性等不是很好，因而后面主要分析通知栏消息。

小米推送通知栏消息格式如下图：

![小米推送通知栏消息格式](https://www.wolfcstech.com/images/1315506-db7a737277fa84c6.png)

### 消息推送服务端操作

![小米消息推送服务端操作](https://www.wolfcstech.com/images/1315506-6256bf45b1ad515e.png)

### 推送消息发送回执

开发者通过调用 `Message.Builder` 类的 `extra(String key, String value)` 来指定消息的送达和点击回执规则。

| KEY  | VALUE​含义  |
|--------|-------------------|
| callback | String，必填字段。第三方接收回执的http接口，最大长度128个字节。 |
| callback.param | String，可选字段。第三方自定义回执参数，最大长度64个字节。 |
| callback.type | int，可选字段。第三方所需要的回执类型。1，送达回执；2，点击回执；3，送达和点击回执，默认值为3。 |

小米推送服务器每隔 1s 将已送达或已点击的消息 ID 和对应设备的 regid 或 alias通过调用第三方 http 接口传给开发者。（每次调用后，小米推送服务器会清空这些数据，下次传给开发者将是新一拨数据。）消息的送达回执只支持向regId或alias发送的消息。

服务器 POST 一个 JSON 数据到 callback 参数对应的 URL。

JSON 对应的 http 参数名为 `data`，JSON数据格式如下：
```
{
      "msgId1":{"param":"param","type": 1, "targets":"alias1,alias2,alias3", "jobkey": "123"},
      "msgId2":{"param":"param","type": 2, "targets":"alias1,alias2,alias3", "jobkey": "456"}
}
```

格式说明：
外层 key 代表相应的消息 msgId，value 是一个 `JSONObject`，包含了下面的参数值。

 * param： 开发者上传的自定义参数值。
 * type：callback类型。
 * targets：一批alias或者regId列表，之间是用逗号分割。
 * jobkey：发送消息时设置的jobkey值。

设置回调相关信息示例如下：
```
        Message.Builder messageBuilder = new Message.Builder()
                .passThrough(Message.PASS_THROUGH_NOTIFICATION)
                .title(title)
                .description(description)
                .payload(paramJsonObject.toJSONString())
                .restrictedPackageName(pusherConfigDto.getPackageName())
                .notifyType(message.getNotifyType())
                .extra("flow_control", "4000") // 设置平滑推送, 推送速度4000每秒(qps=4000)
                .extra("callback", pusherConfigDto.getCallbackUrl())
                .extra("callback.param", JSON.toJSONString(biTag))
                .extra("callback.type", "3");
```

从回执的请求中拿到回执数据的方法如下：
```
    @RequestMapping("/callback/xiaomi/v{ver}")
    @ResponseBody
    public MobReturnVo callbackXiaomi(HttpServletRequest request) {
        String callback = request.getParameter("data");
        if (StringUtils.isNotEmpty(callback)) {
            LOG.info("callback:" + callback);
            pushLogic.processXiaomiCallbackLog(callback);
        }
        return new MobReturnVo.OKVo();
    }
```

### 通知栏推送消息中的额外应用数据

通知栏消息中的标题等字段用于在客户端的通知栏中展示给用户，对于小米推送而言，标题 `title` 和描述 `description` 将会展示给用户，通知类型等用于控制推送消息的行为等。除此之外，推送消息中还可以携带一些数据，不由第三方推送服务使用，而是完全由APP 处理。对于小米推送而言，这部分数据由载荷携带，即通过 `Message.Builder` 的 `payload()` 设置的数据。

像下面这样：
```
        JSONObject paramJsonObject = new JSONObject();
        paramJsonObject.put("payload", message.getContent());
        paramJsonObject.put("msgId", messageUUID);
        Map<String, String> extra = message.getExtra();
        if (MapUtils.isNotEmpty(extra)) {
            paramJsonObject.putAll(extra);
        }

        BiTag biTag = new BiTag();
        biTag.setProductKey(pusherConfigDto.getProductKey());
        biTag.setMsgId(messageUUID);

        Message.Builder messageBuilder = new Message.Builder()
                .passThrough(Message.PASS_THROUGH_NOTIFICATION)
                .title(title)
                .description(description)
                .payload(paramJsonObject.toJSONString())
                .restrictedPackageName(pusherConfigDto.getPackageName())
                .notifyType(message.getNotifyType())
                .extra("flow_control", "4000") // 设置平滑推送, 推送速度4000每秒(qps=4000)
                .extra("callback", pusherConfigDto.getCallbackUrl())
                .extra("callback.param", JSON.toJSONString(biTag))
                .extra("callback.type", "1");
```

在 Android 客户端，这部分数据在实现的 `com.xiaomi.mipush.sdk.PushMessageReceiver` 子类的 `onNotificationMessageClicked()` 方法中，由 `MiPushMessage` 的 `getContent()` 方法获取，像下面这样：
```
    @Override
    public void onNotificationMessageClicked(Context context, MiPushMessage message) {
        if (message == null) {
            return;
        }
        NTLog.d(TAG, message.toString());
        MessageInstance.getInstance().getPushMessageLogic().clickPushMessage(context, message);

        mMessage = message.getContent();
    }
```

## 魅族消息推送

魅族消息推送[官方文档地址](http://open-wiki.flyme.cn/doc-wiki/index)。

### 魅族推送消息组成及行为控制选项

![魅族推送消息组成及行为控制选项](https://www.wolfcstech.com/images/1315506-71ce7553e6232a91.png)

### 消息推送服务端操作
![魅族消息推送服务端操作](https://www.wolfcstech.com/images/1315506-8deb2be2d516c118.png)

关于魅族消息推送系统需要注意：

 * 支持回执接口的接口为 ***pushId通知栏消息推送(pushMessage)***，也就是说，任务推送不支持回执接口。

### 推送消息发送回执

关于消息推送回调的详细说明，可以参考文档 [Flyme推送JAVA版本SDK接口文档](http://open.res.flyme.cn/fileserver/upload/file/201803/e174a5709f134f64aae3fb168aec8ea3.pdf) 的 **高级功能** -> **消息送达与回执** 的部分。

开发者首先需要登录魅族推送平台【配置管理】->【回执管理】注册回执地址，然后通过调用 `VarnishedMessage.Builder` 的 `extra()` 设置相关的回执配置，包括回执的 URL、回执参数，和回执事件类型等。类似于下面这样：
```
        VarnishedMessage.Builder messageBuilder = new VarnishedMessage.Builder()
                .appId(Long.valueOf(pusherConfigDto.getAppId()))
                .title(message.getTitle())
                .content(message.getAlert())
                .clickType(0)
                .offLine(true)
                .validTime(24)
                .fixSpeed(true)
                .fixSpeedRate(500)
                .suspend(true)
                .clearNoticeBar(true)
                .vibrate((message.getNotifyType() & EdsMessage.NOTIFY_TYPE_VIBRATE) != 0)
                .lights((message.getNotifyType() & EdsMessage.NOTIFY_TYPE_LIGHTS) != 0)
                .sound((message.getNotifyType() & EdsMessage.NOTIFY_TYPE_SOUND) != 0)
                .parameters(paramJsonObject)
                .extra(ExtraParam.CALLBACK.getKey(), pusherConfigDto.getCallbackUrl())
                .extra(ExtraParam.CALLBACK_PARAM.getKey(), JSON.toJSONString(biTag))
                .extra(ExtraParam.CALLBACK_TYPE.getKey(), CallBackType.RECEIVE_CLICK.getKey());
```

回执参数(ExtraParam) 说明如下：

| 枚举 | 描述 |
|-------|---------|
| CALLBACK | 回执接口(第三方接收回执的Http接口, 最大长度128字节) |
| CALLBACK_PARAM | 回执参数(第三方自定义回执参数, 最大长度64字节) |
| CALLBACK_TYPE | 回执类型((1-送达回执, 2-点击回执, 3-送达与点击回执), 默认3) |

回执类型(CallBackType) 说明如下：

| 枚举 | 描述 |
|-------|---------|
| RECEIVE | 送达回执 |
| CLICK | 点击回执 |
| RECEIVE_CLICK | 送达与点击回执 |

魅族推送服务器每隔 1s 将已送达或已点击的消息 ID 和对应设备的 pushId 或alias 通过调用开发者 http 接口传给开发者(每次调用后, 魅族推送服务器会清空这些数据,下次传给业务方将是新一拨数据)。

消息的送达回执只支持向pushId或alias发送的消息。

单个应用注册不同回执地址累计上限不能超过100个。

回执响应内容为：

| Key | value |
|-------|---------|
| cb | 回执明细内容 如下所述(Json数据) |
| access_token | 回执接口访问令牌(推送平台设置回执地址令牌) |

回执明细格式说明: 外层 key 代表相应的消息 id 和回执类型 (msgId-type)，value 是一个
JSONObject，包含了下面的参数值：

 * param：业务上传的自定义参数值
 * type：callback类型
 * targets：一批alias或者pushId集合

抓包示例：
```
Hypertext Transfer Protocol
POST /callbackUrl/callback HTTP/1.1\r\n
[Expert Info (Chat/Sequence): POST /callbackUrl/callback HTTP/1.1\r\n]
Request Method: POST
Request URI: /callbackUrl/callback
Request Version: HTTP/1.1
Content-Length: 32\r\n
[Content length: 32]
Content-Type: application/x-www-form-urlencoded; charset=UTF-8\r\n
Host: xxx.xxx.xxx\r\n
Connection: Keep-Alive\r\n
30/39User-Agent: Apache-HttpClient/4.3.4 (java 1.5)\r\n
Accept-Encoding: gzip,deflate\r\n
\r\n
[Full request URI: http://xxx.xxx.xxx/callbackUrl/callback]
[HTTP request 1/1]
[Response in frame: 7253]
File Data: 32 bytes
HTML Form URL Encoded: application/x-www-form-urlencoded
Form item: "access_token" = "token"
   Key: access_token
   Value: token
Form item: "cb" = "json value"
   Key: cb
   Value: json value
```

由回调请求拿到回调数据的方法如下：
```
    @RequestMapping("/callback/meizu/v{ver}")
    @ResponseBody
    public String callbackMeizu(HttpServletRequest request) {
        String callbackData = request.getParameter("cb");
        String accessToken = request.getParameter("access_token");
        System.out.println("callbackData:" + callbackData);
        System.out.println("accessToken:" + accessToken);
        return "ok";
    }
```

由回调请求拿到的回调数据类似下面这样：
```
callbackData:{"NS20180830094758844_0_11529504_1_3-1":{"param":"{\"msgId\":\"c69f1867-0509-4343-bff0-04ba5beb6725\"}","type":1,"targets":["N6A6977416a7a77536b59675d61507c7b036074466a77"]}}
accessToken:2324515cf21b4ea2b6b916637b1a7c3b
callbackData:{"NS20180830094902957_0_11529500_1_3-2":{"param":"{\"msgId\":\"4a9ef86f-66db-4819-b98b-537a4dca64ce\"}","type":2,"targets":["N6A6977416a7a77536b59675d61507c7b036074466a77"]}}
accessToken:2324515cf21b4ea2b6b916637b1a7c3b
```

### 通知栏推送消息中的额外应用数据

对于魅族消息推送而言，用于显示的是标题 `title` 和内容 `content`，推送服务不处理，消息中完全由应用程序自行解析的额外数据，在构造 `VarnishedMessage` 时，通过 `VarnishedMessage.Builder` 的 `parameters()` 方法设置，如下面这样：
```
        JSONObject paramJsonObject = new JSONObject();
        paramJsonObject.put("payload", message.getContent());
        paramJsonObject.put("msgId", messageUUID);

        Map<String, String> extra = message.getExtra();
        if(MapUtils.isNotEmpty(extra)) {
            paramJsonObject.putAll(extra);
        }

        VarnishedMessage.Builder messageBuilder = new VarnishedMessage.Builder()
                .appId(Long.valueOf(pusherConfigDto.getAppId()))
                .title(message.getTitle())
                .content(message.getAlert())
                .clickType(0)
                .offLine(true)
                .validTime(24)
                .fixSpeed(true)
                .fixSpeedRate(500)
                .suspend(true)
                .clearNoticeBar(true)
                .vibrate((message.getNotifyType() & EdsMessage.NOTIFY_TYPE_VIBRATE) != 0)
                .lights((message.getNotifyType() & EdsMessage.NOTIFY_TYPE_LIGHTS) != 0)
                .sound((message.getNotifyType() & EdsMessage.NOTIFY_TYPE_SOUND) != 0)
                .parameters(paramJsonObject)
                .extra(ExtraParam.CALLBACK.getKey(), pusherConfigDto.getCallbackUrl())
                .extra(ExtraParam.CALLBACK_PARAM.getKey(), JSON.toJSONString(biTag))
                .extra(ExtraParam.CALLBACK_TYPE.getKey(), CallBackType.RECEIVE_CLICK.getKey());
```

`parameters()` 方法接受一个 `JSONObject` 的参数。

在 Android 端，这些数据，在实现的`com.meizu.cloud.pushsdk.MzPushMessageReceiver` 子类的回调中，通过 `MzPushMessage` 的  `getSelfDefineContentString()` 方法获得，如下面这样：
```
    @Override
    public void onNotificationClicked(Context context, MzPushMessage mzPushMessage) {
        DebugLogger.i(TAG, "onNotificationClicked title "+ mzPushMessage.getTitle() + "content "
                + mzPushMessage.getContent() + " selfDefineContentString " + mzPushMessage.getSelfDefineContentString()+" notifyId "+mzPushMessage.getNotifyId());

        if(!TextUtils.isEmpty(mzPushMessage.getSelfDefineContentString())){
            print(context," 点击自定义消息为："+mzPushMessage.getSelfDefineContentString());
        }
    }
```

## 华为消息推送

华为消息推送[官方文档地址](https://developer.huawei.com/consumer/cn/service/hms/catalog/huaweipush_agent.html?page=hmssdk_huaweipush_api_reference_agent_s2)。

### 推送消息结构

![华为推送消息载荷组成](https://www.wolfcstech.com/images/1315506-73a51aa013b55921.png)

### 消息推送服务端操作

华为推送只提供了两个 REST 接口，分别是获得访问令牌和推送消息。

### 推送消息发送回执

(1). 回执的安全 目前华为Push服务器和接收回执消息的 APP 服务器之间走的是HTTPS 协议，华为 Push 服务器会校验 APP 服务器提供证书的合法性，APP 服务器务必使用正式商用的HTTPS证书。

(2). 回执消息的开通 a). 开通Push服务后，发送邮件到 ([developer@huawei.com](mailto:developer@huawei.com))申请开通回执消息，邮件中需要提供应用的 AppId、接收回执消息的URL地址以及每天发送消息的总量。 b). 申请加入华为 Push 回执 QQ 群，便于后续联调功能以及沟通交流问题，QQ群号：224090622。

(3). 回执消息的API 回执消息发生在系统检测到客户端有应答返回，并且有 CP 订阅了该应答结果的场景。华为Push平台和CP接收回执消息服务器使用HTTPS协议。

华为推送消息的发送回执搞起来比较麻烦，详见 [服务端开发指南](https://developer.huawei.com/consumer/cn/service/hms/catalog/huaweipush_agent.html?page=hmssdk_huaweipush_devguide_server_agent) 的 ***3.3 消息回执***。

### 通知栏推送消息中的额外应用数据
对于华为消息推送而言，用于显示的是标题 `title` 和内容 `content`，推送服务不处理，消息中完全由应用程序自行解析的额外数据，由消息中的 `hps` -> `ext` -> `customize` 字段携带。

在 Android 客户端，这些数据，在实现的`com.huawei.hms.support.api.push.PushReceiver` 子类的回调方法 `onEvent()` 中获得，如下面这样：

```
    public void onEvent(Context context, Event event, Bundle extras) {
        Intent intent = new Intent();
        intent.setAction(ACTION_UPDATEUI);

        int notifyId = 0;
        if (Event.NOTIFICATION_OPENED.equals(event) || Event.NOTIFICATION_CLICK_BTN.equals(event)) {
            notifyId = extras.getInt(BOUND_KEY.pushNotifyId, 0);
            if (0 != notifyId) {
                NotificationManager manager = (NotificationManager) context
                        .getSystemService(Context.NOTIFICATION_SERVICE);
                manager.cancel(notifyId);
            }
        }

        String message = extras.getString(BOUND_KEY.pushMsgKey);
        intent.putExtra("log", "Received event,notifyId:" + notifyId + " msg:" + message);
        callBack(intent);
        super.onEvent(context, event, extras);
    }
```

参考文档
[Android 端外推送到底有多烦？](https://juejin.im/entry/57b3fae71532bc0063e30c7d)
[推送服务常见问题](https://dev.mi.com/doc/p=1608/index.html)
[Android 第三方 Push 推送方案使用调查](https://github.com/android-cn/topics/issues/4)
[android推送服务，目前哪家相对较好？](https://www.zhihu.com/question/23507243)

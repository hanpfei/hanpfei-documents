---
title: mediasoup-client 和 libmediasoupclient 指南
date: 2022-02-20 21:05:49
categories: 音视频开发
tags:
- 音视频开发
---

mediasoup 是一个多用于多端视频会议系统的 SFU，它主要用来做各个端点之间的媒体数据等的转发。mediasoup 本身主要围绕媒体数据转发来构建。

这里先通过 mediasoup 的架构，简单看一下 mediasoup 这个 SFU 建立的主要概念和抽象，如 Router、Transport、Producer、Consumer、DataProducer 和 DataConsumer 等，以及它们在 mediasoup 的媒体数据转发系统中的角色和作用：

![1640916356184.jpg](https://upload-images.jianshu.io/upload_images/1315506-f5c9568d35ecf87c.jpg)

如上图，描述了 mediasoup 转发服务的主要架构。在 mediasoup 媒体数据转发系统的服务端，Router 是核心，它完成媒体数据的转发。Producer/DataProducer 抽象数据提供者，Consumer/DataConsumer 抽象数据消费者，Transport 抽象 WebRTC 数据传输。
<!--more-->
mediasoup 客户端库，包括用于 JavaScript 的 mediasoup-client，和用于 C++ 的 libmediasoupclient，建立了一些抽象用来支持服务端的媒体数据转发。客户端的部分抽象的名称可能与它对应的服务端中的抽象名称相同，但含义是不一样的。比如 Producer，在服务器端，这个角色拿到媒体数据给到 Router，但在客户端，所有的数据并不会经过这个组件，它只是维护了服务端对应角色的信息而已。客户端库 mediasoup-client 和 libmediasoupclient 中所有这些抽象的含义可以参考 [API 文档](https://mediasoup.org/documentation/v3/mediasoup-client/api/)，下文也会对这些抽象做更详细的说明。

有了客户端库和 mediasoup 服务，还不足以建立完整的多方视频会议场景，这还需要信令协议的协助。对于 mediasoup，需要信令协议在适当的时候协调 mediasoup SFU 服务器完成媒体转发所需要的服务端对象的创建，如 router、producer 和 consumer 等，资源的分配，和链路的打通，并协调客户端与服务端建立连接，发送接收数据，及数据收发控制，协调客户端和服务器之间交换媒体相关参数等。

mediasoup 本身不提供任何信令协议来帮助客户端和服务器进行通信。信令的传递取决于应用程序，它们可以使用 WebSocket、HTTP 或其它通信方式进行通信，并在客户端和服务器之间交换 mediasoup 相关的参数、请求/响应和通知。在大多数情况下，服务端可能需要主动向客户端递送消息或事件通知，则客户端和服务器的这种通信必须是双向的，因此通常需要全双工的通道。但是，应用程序可以服用相同的通道进行非 mediasoup 相关的消息交换 ( 例如身份验证过程、聊天消息、文件传输和任何应用程序希望实现的内容)。

 > 前面说 mediasoup 本身不提供任何信令协议，其实不太准确。在 mediasoup v2 的时候，还是有信令协议的，具体内容如 [mediasoup protocol](https://mediasoup.org/documentation/v2/mediasoup-protocol/) 和 [MEDIASOUP_PROTOCOL.md](https://github.com/versatica/mediasoup-client/blob/v2/MEDIASOUP_PROTOCOL.md) 的说明。但在最新的 v3 版中，已经没有这部分了。信令协议需要应用系统自己实现。

 > 信令协议实现的示例可以参考 [mediasoup-demo](https://github.com/versatica/mediasoup-demo) 的 server 的 `mediasoup-demo/server/server.js`。

在客户端，可以认为 Device 是整个应用程序的中心，它协调和控制整个操作过程。Device 表示连接到 mediasoup 以发送/接收媒体的端点。它是客户端应用程序的入口点。

我们假设我们的 JavaScript 或 C++ 客户端应用程序初始化了一个 mediasoup-client [Device](https://mediasoup.org/documentation/v3/mediasoup-client/api/#Device) 或一个 libmediasoupclient [Device](https://mediasoup.org/documentation/v3/libmediasoupclient/api/#Device) 对象，连接一个 mediasoup [Router](https://mediasoup.org/documentation/v3/mediasoup/api/#Router) （已经在服务器中创建）并基于 WebRTC 发送和接收媒体数据。

这里 mediasoup [Router](https://mediasoup.org/documentation/v3/mediasoup/api/#Router) 创建的服务端应用接口，如 [mediasoup-demo](https://github.com/versatica/mediasoup-demo) 的 server 的 `mediasoup-demo/server/server.js` 所实现的 websocket 接口：
```
async function runProtooWebSocketServer()
{
	logger.info('running protoo WebSocketServer...');

	// Create the protoo WebSocket server.
	protooWebSocketServer = new protoo.WebSocketServer(httpsServer,
		{
			maxReceivedFrameSize     : 960000, // 960 KBytes.
			maxReceivedMessageSize   : 960000,
			fragmentOutgoingMessages : true,
			fragmentationThreshold   : 960000
		});

	// Handle connections from clients.
	protooWebSocketServer.on('connectionrequest', (info, accept, reject) =>
	{
		// The client indicates the roomId and peerId in the URL query.
		const u = url.parse(info.request.url, true);
		const roomId = u.query['roomId'];
		const peerId = u.query['peerId'];

		if (!roomId || !peerId)
		{
			reject(400, 'Connection request without roomId and/or peerId');

			return;
		}

		logger.info(
			'protoo connection request [roomId:%s, peerId:%s, address:%s, origin:%s]',
			roomId, peerId, info.socket.remoteAddress, info.origin);

		// Serialize this code into the queue to avoid that two peers connecting at
		// the same time with the same roomId create two separate rooms with same
		// roomId.
		queue.push(async () =>
		{
			const room = await getOrCreateRoom({ roomId });

			// Accept the protoo WebSocket connection.
			const protooWebSocketTransport = accept();

			room.handleProtooConnection({ peerId, protooWebSocketTransport });
		})
			.catch((error) =>
			{
				logger.error('room creation or room joining failed:%o', error);

				reject(error);
			});
	});
}
```

mediasoup-client (客户端 JavaScript 库) 和 libmediasoupclient (基于 libwebrtc 的 C++ 库) 都生成适用于 mediasoup 的 [RTP 参数](https://mediasoup.org/documentation/v3/mediasoup/rtp-parameters-and-capabilities/)，这简化了客户端应用程序的开发。

## 信令和 Peers

应用程序可以使用 WebSocket，并将每个经过认证的 WebSocket 连接与一个 “peer” 关联。

注意 mediasoup 中本身并没有 “peers”。然而，应用程序可能希望定义 “peers”，这可以标识并关联一个特定的用户账号，WebSocket 连接，metadata，及一系列 mediasoup transports，producers，consumers，data producers 和 data consumers。

## 设备加载

客户端应用程序通过给 device 提供服务端 mediasoup router 的 RTP capabilities 加载它的 mediasoup device。参考 [device.load()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#device-load)。

这里的服务端 mediasoup router 的 RTP capabilities 需要通过信令协议从 mediasoup 服务器端获取。如对于 [mediasoup-demo](https://github.com/versatica/mediasoup-demo) 的 server 应用，向服务器端发送 GET 请求，服务器端返回 JSON 格式的 RTP capabilities 的响应。HTTP url path 为 `/rooms/:roomId`，如对于房间名为 `broadcaster` 的房间，为 `/rooms/broadcaster`：
```
  auto r = cpr::GetAsync(cpr::Url{baseUrl}, cpr::VerifySsl{verifySsl}).get();

  if (r.status_code != 200) {
    std::cerr << "[ERROR] unable to retrieve room info"
              << " [status code:" << r.status_code << ", body:\"" << r.text
              << "\"]" << std::endl;

    return 1;
  } else {
    std::cout << "[INFO] found room " << envRoomId << std::endl;
  }
  auto response = nlohmann::json::parse(r.text);
```

响应为一个常常的 JSON 字符串。

## 创建 Transports

mediasoup-client 和 libmediasoupclient 都需要将 WebRTC 传输的发送和接收分开。通常客户端应用程序会提前创建这些 transports，甚至在想要发送或接收媒体数据之前。

对于发送媒体数据：
 * WebRTC transport 必须首先在 mediasoup router 中创建： [router.createWebRtcTransport()](https://mediasoup.org/documentation/v3/mediasoup/api/#router-createWebRtcTransport)。
 * 然后重复地在客户端应用程序中创建：[device.createSendTransport()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#device-createSendTransport)。
 * 客户端应用程序必须订阅本地 transport 中的 “connect” 和 “produce” 事件。

对于 [mediasoup-demo](https://github.com/versatica/mediasoup-demo) 的 server 的 `mediasoup-demo/server/server.js`，在创建 transport 之前，还需要先在服务器中创建 Broadcaster，POST 请求为：
```
	json body =
	{
		{ "id",          this->id          },
		{ "displayName", "broadcaster"     },
		{ "device",
			{
				{ "name",    "libmediasoupclient"       },
				{ "version", mediasoupclient::Version() }
			}
		},
		{ "rtpCapabilities", this->device.GetRtpCapabilities() }
	};
  /* clang-format on */

  auto url = baseUrl + "/broadcasters";
  auto r = cpr::PostAsync(cpr::Url{url}, cpr::Body{body.dump()},
                          cpr::Header{{"Content-Type", "application/json"}},
                          cpr::VerifySsl{verifySsl})
               .get();
```

其中 id 为本地生成的一个随机字符串。

响应为：
```
{
    "peers":[
        {
            "id":"ej8ogujz",
            "displayName":"Elgyem",
            "device":{
                "flag":"safari",
                "name":"Safari",
                "version":"14.1"
            },
            "producers":[
                {
                    "id":"87230aeb-027e-4204-99eb-080cd4972bb0",
                    "kind":"audio"
                },
                {
                    "id":"66c62c26-7101-43b2-b82c-cdf537b8d9ed",
                    "kind":"video"
                }
            ]
        }
    ]
}
```

响应中主要包含了相同房间内，其它 peer 的信息。

在 mediasoup router 中创建 WebRTC transport 通过如下 HTTP 请求完成：
```
  json sctpCapabilities = this->device.GetSctpCapabilities();
  /* clang-format off */
	json body =
	{
		{ "type",    "webrtc" },
		{ "rtcpMux", true     },
		{ "sctpCapabilities", sctpCapabilities }
	};
  /* clang-format on */

  auto url = baseUrl + "/broadcasters/" + id + "/transports";
  auto r = cpr::PostAsync(cpr::Url{url}, cpr::Body{body.dump()},
                          cpr::Header{{"Content-Type", "application/json"}},
                          cpr::VerifySsl{verifySsl})
               .get();
```

这个请求的响应为：
```
{
    "id":"6eae5aae-3ae9-4545-a146-466b28e05da7",
    "iceParameters":{
        "iceLite":true,
        "password":"g08jh0b528i0fshqld1cmdgijhzhstuz",
        "usernameFragment":"v77q4zq05bhni7c1"
    },
    "iceCandidates":[
        {
            "foundation":"udpcandidate",
            "ip":"192.168.217.129",
            "port":40065,
            "priority":1076302079,
            "protocol":"udp",
            "type":"host"
        }
    ],
    "dtlsParameters":{
        "fingerprints":[
            {
                "algorithm":"sha-1",
                "value":"5F:2D:8A:74:CD:95:65:3C:4B:10:27:1A:01:BA:CE:F7:0B:23:B9:AE"
            },
            {
                "algorithm":"sha-224",
                "value":"9C:19:4F:40:43:A9:AE:DD:01:00:7A:98:0C:5D:26:99:BD:9E:FB:A0:4F:EA:FB:0C:39:D2:2B:BD"
            },
            {
                "algorithm":"sha-256",
                "value":"D8:FD:D9:5B:9C:37:2A:4C:F7:99:D4:35:F2:90:7C:9E:D8:1A:74:10:B3:33:B4:71:B7:22:8F:C5:A5:59:FF:BD"
            },
            {
                "algorithm":"sha-384",
                "value":"B9:2B:D5:6C:60:0F:B0:A0:E3:6E:57:7D:02:91:52:AE:75:D7:3F:E1:34:83:45:39:DA:53:93:09:ED:53:6C:A9:01:1E:20:16:06:C3:48:40:07:9B:A5:6C:B3:E1:81:A9"
            },
            {
                "algorithm":"sha-512",
                "value":"46:F6:77:11:ED:ED:80:EA:97:EA:36:FF:CD:4B:E1:C0:36:09:ED:F4:E0:B8:56:F0:8D:FB:9C:12:AF:A3:86:05:82:C0:F8:B9:CA:E6:7D:62:5C:72:5F:10:23:F5:66:27:04:A5:BA:F4:63:D9:F5:42:D6:22:0C:86:51:43:1D:B4"
            }
        ],
        "role":"auto"
    },
    "sctpParameters":{
        "MIS":1024,
        "OS":1024,
        "isDataChannel":true,
        "maxMessageSize":262144,
        "port":5000,
        "sctpBufferedAmount":0,
        "sendBufferSize":262144
    }
}
```

对于接收媒体数据：
 * WebRTC transport 必须首先在 mediasoup router 中创建： [router.createWebRtcTransport()](https://mediasoup.org/documentation/v3/mediasoup/api/#router-createWebRtcTransport)。
 * 然后重复地在客户端应用程序中创建：[device.createRecvTransport()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#device-createRecvTransport)。
 * 客户端应用程序必须订阅本地 transport 中的 “connect” 和 “produce” 事件。

如果在这些 transports 中需要使用 SCTP (即 WebRTC 中的 DataChannel)，必须在其中启用 **enableSctp** (使用适当的 [numSctpStreams](https://mediasoup.org/documentation/v3/mediasoup/sctp-parameters/#NumSctpStreams)) 和其他 SCTP 相关设置。

## 生产媒体数据

一旦创建了 send transport，客户端应用程序就可以在其上生成多个音频和视频 tracks。

 * 应用程序获得一个 [track](https://www.w3.org/TR/mediacapture-streams/#mediastreamtrack) (例如，通过使用 **navigator.mediaDevices.getUserMedia()** API)。
 * 它在本地 send transport 中调用 [transport.produce()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-produce)。
   * 如果这是对 **transport.produce()** 的第一次调用，则 transport 将发出 “connect” 事件。
   * transport 将发出“produce” 事件，因此应用程序将把事件参数传递给服务器，并在服务器端创建一个 [Producer](https://mediasoup.org/documentation/v3/mediasoup/api/#Producer) 实例。
 * 最后，**transport.produce()** 将在客户端使用 [Producer](https://mediasoup.org/documentation/v3/mediasoup-client/api/#Producer) 实例进行解析。

这里的把事件参数传递给服务器，对应于 [mediasoup-demo](https://github.com/versatica/mediasoup-demo) 的 server 的 `mediasoup-demo/server/server.js` 的连接 send transport 请求：
```
  /* clang-format off */
	json body =
	{
		{ "dtlsParameters", dtlsParameters }
	};
  /* clang-format on */

  auto url = baseUrl + "/broadcasters/" + this->id + "/transports/" +
             sendTransport->GetId() + "/connect";
  std::cout << "Connect send transport url: " << url << std::endl;
  auto r = cpr::PostAsync(cpr::Url{url}, cpr::Body{body.dump()},
                          cpr::Header{{"Content-Type", "application/json"}},
                          cpr::VerifySsl{verifySsl})
               .get();
```

在本地 send transport 中调用 [transport.produce()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-produce) 时发出请求：
```
#0  Broadcaster::OnConnectSendTransport (this=0x3d440000c280, dtlsParameters=...) at ~/mediasoup-broadcaster-demo/src/Broadcaster.cpp:58
#1  0x0000555555655e86 in Broadcaster::OnConnect (this=0x7fffffffdbd0, transport=0x3d4400031180, dtlsParameters=...)
    at ~/mediasoup-broadcaster-demo/src/Broadcaster.cpp:44
#2  0x00005555576d7d91 in mediasoupclient::Transport::OnConnect (this=0x3d4400031180, dtlsParameters=...)
    at ~/mediasoup-broadcaster-demo/build/_deps/mediasoupclient-src/src/Transport.cpp:106
#3  0x00005555576bab97 in mediasoupclient::Handler::SetupTransport (this=0x3d44000bd280, localDtlsRole="server", localSdpObject=...)
    at ~/mediasoup-broadcaster-demo/build/_deps/mediasoupclient-src/src/Handler.cpp:145
#4  0x00005555576bb6b0 in mediasoupclient::SendHandler::Send (this=0x3d44000bd280, track=0x3d4400085fc0, encodings=0x7fffffffcff0, codecOptions=0x7fffffffd1e0, 
    codec=0x0) at ~/mediasoup-broadcaster-demo/build/_deps/mediasoupclient-src/src/Handler.cpp:232
#5  0x00005555576d8f70 in mediasoupclient::SendTransport::Produce (this=0x3d4400031180, producerListener=0x7fffffffdbe0, track=0x3d4400085fc0, encodings=0x0, 
    codecOptions=0x7fffffffd1e0, codec=0x0, appData=...)
    at ~/mediasoup-broadcaster-demo/build/_deps/mediasoupclient-src/src/Transport.cpp:220
#6  0x000055555565ae46 in Broadcaster::CreateSendTransport (this=0x7fffffffdbd0, enableAudio=true, useSimulcast=true)
    at ~/mediasoup-broadcaster-demo/src/Broadcaster.cpp:420
#7  0x000055555565901e in Broadcaster::Start (this=0x7fffffffdbd0, baseUrl="https://192.168.217.129:4443/rooms/broadcaster", enableAudio=true, useSimulcast=true, 
    routerRtpCapabilities=..., verifySsl=false) at ~/mediasoup-broadcaster-demo/src/Broadcaster.cpp:296
#8  0x000055555569f309 in main () at ~/mediasoup-broadcaster-demo/src/main.cpp:103
```

这个请求没有响应。

此外，还会向服务端发送两个请求，分别在 mediasoup 服务器中为音频和视频创建 Producer：
```
#0  Broadcaster::OnProduce (this=0x7fffffffcc80, kind="", rtpParameters=...) at ~/mediasoup-broadcaster-demo/src/Broadcaster.cpp:147
#1  0x00005555576d9019 in mediasoupclient::SendTransport::Produce (this=0x3d4400031180, producerListener=0x7fffffffdbe0, track=0x3d440009d690, 
    encodings=0x7fffffffd200, codecOptions=0x0, codec=0x0, appData=...)
    at ~/mediasoup-broadcaster-demo/build/_deps/mediasoupclient-src/src/Transport.cpp:229
#2  0x000055555565b0ad in Broadcaster::CreateSendTransport (this=0x7fffffffdbd0, enableAudio=true, useSimulcast=true)
    at ~/mediasoup-broadcaster-demo/src/Broadcaster.cpp:438
#3  0x000055555565901e in Broadcaster::Start (this=0x7fffffffdbd0, baseUrl="https://192.168.217.129:4443/rooms/broadcaster", enableAudio=true, useSimulcast=true, 
    routerRtpCapabilities=..., verifySsl=false) at ~/mediasoup-broadcaster-demo/src/Broadcaster.cpp:296
#4  0x000055555569f309 in main () at ~/mediasoup-broadcaster-demo/src/main.cpp:103
```

请求格式如下：
```
	json body =
	{
		{ "kind",          kind          },
		{ "rtpParameters", rtpParameters }
	};
  /* clang-format on */

  auto url = baseUrl + "/broadcasters/" + id + "/transports/" +
             sendTransport->GetId() + "/producers";
  std::cout << "Produce url: " << url << std::endl;
  auto r = cpr::PostAsync(cpr::Url{url}, cpr::Body{body.dump()},
                          cpr::Header{{"Content-Type", "application/json"}},
                          cpr::VerifySsl{verifySsl})
               .get();
```

响应格式如下：
```
{
    "id":"8624d454-9519-436b-8da9-56755c1bd2b6"
}
```

返回一个 id。

## 消费媒体数据

一旦创建了 receive transport，客户端应用程序就可以使用它上的多个音频和视频 tracks。但是顺序是相反的 (这里消费者必须首先在服务器中创建)。

 * 客户端应用程序向服务器发送它的 [device.rtpCapabilities](https://mediasoup.org/documentation/v3/mediasoup-client/api/#device-rtpCapabilities) (它可能已经提前完成了)。
 * 服务器应用程序应该检查远端设备是否可以使用特定的生产者 (也就是说，它是否支持生产者媒体编解码器)。它可以通过使用  [router.canConsume()](https://mediasoup.org/documentation/v3/mediasoup/api/#router-canConsume) 方法来实现。
 * 然后服务器应用程序在客户端为接收媒体数据而创建的 WebRTC transport 中调用 [transport.consume()](https://mediasoup.org/documentation/v3/mediasoup/api/#transport-consume) ，从而生成一个服务器端的 [Consumer](https://mediasoup.org/documentation/v3/mediasoup-client/api/#Consumer)。
   * 正如 [transport.consume()](https://mediasoup.org/documentation/v3/mediasoup/api/#transport-consume) 文档中所解释的，强烈建议使用 **paused: true** 创建服务器端 consumer，并在远程端点中创建 consumer 后恢复它。
 * 服务器应用程序将 consumer 信息和参数传输到远程客户端应用程序，远程客户端应用程序在本地 receive transport 中调用 [transport.consume()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-consume)。
   * 如果这是对 **transport.consume()** 的第一次调用，transport 将发出 [“connect”](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-on-connect) 事件。
 * 最后，在客户端将以一个 [Consumer](https://mediasoup.org/documentation/v3/mediasoup-client/api/#Consumer) 实例解析 **transport.consume()**。

## 生产数据 (DataChannels)

一旦创建了 send transport，客户端应用程序就可以在其上生成多个 [DataChannels](https://www.w3.org/TR/webrtc/#rtcdatachannel)。

 * 应用程序在本地 send transport 中调用 [transport.produceData()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-producedata)。
   * 如果这是对 **transport.produceData()** 的第一次调用，则 transport 将发出 [“connect”](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-on-connect) 事件。
   * transport 将发出[“producedata”](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-on-producedata) 事件，因此应用程序将把事件参数传递给服务器，并在服务器端创建一个 [DataProducer](https://mediasoup.org/documentation/v3/mediasoup/api/#DataProducer) 实例。
 * 最后，**transport.produceData()** 将在客户端使用 [DataProducer](https://mediasoup.org/documentation/v3/mediasoup-client/api/#DataProducer) 实例进行解析。


## 消费数据 (DataChannels)

一旦创建了 receive transport，客户端应用程序就可以使用它上的多个 [DataChannels](https://www.w3.org/TR/webrtc/#rtcdatachannel) 了。但是顺序是相反的 (这里消费者必须首先在服务器中创建)。

 * 服务器应用程序在客户端为接收数据而创建的 WebRTC transport 中调用  [transport.consumeData()](https://mediasoup.org/documentation/v3/mediasoup/api/#transport-consumedata)，从而生成一个服务器端的 [DataConsumer](https://mediasoup.org/documentation/v3/mediasoup-client/api/#DataConsumer)。
 * 服务器应用程序将 consumer 信息和参数传输到客户端应用程序，客户端应用程序在本地 receive transport 中调用 [transport.consumeData()](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-consumedata)。
   * 如果这是对 **transport.consumeData()** 的第一次调用，transport 将发出 [“connect”](https://mediasoup.org/documentation/v3/mediasoup-client/api/#transport-on-connect) 事件。
 * 最后，在客户端将以一个 [DataConsumer](https://mediasoup.org/documentation/v3/mediasoup-client/api/#Consumer) 实例解析 **transport.consumeData()**。

## 通信行为和事件

作为核心原则，调用 mediasoup 实例中的方法不会在该实例中生成直接事件。总之，这意味着在路 router、transport、producer、consumer、data producer 或 data consumer 上调用 **close()** 不会触发任何事件。

当一个 transport、producer、consumer、data producer 或 data consumer 在客户端或服务器端被关闭时 (例如通过在它上调用 **close()**)，应用程序应该向另一端发出它的关闭信号，另一端也应该在相应的实体上调用 **close()**。另外，服务器端应用程序应该监听以下关闭事件并通知客户端：

*   Transport [“routerclose”](https://mediasoup.org/documentation/v3/mediasoup/api/#transport-on-routerclose)。客户端应该在对应的本地 transport 中调用 `close()`。
*   Producer [“transportclose”](https://mediasoup.org/documentation/v3/mediasoup/api/#producer-on-transportclose)。客户端应该在对应的本地 producer 中调用 `close()`。
*   Consumer [“transportclose”](https://mediasoup.org/documentation/v3/mediasoup/api/#consumer-on-transportclose)。客户端应该在对应的本地 consumer 中调用 `close()`。
*   Consumer [“producerclose”](https://mediasoup.org/documentation/v3/mediasoup/api/#consumer-on-producerclose)。客户端应该在对应的本地 consumer 中调用 `close()`。
*   DataProducer [“transportclose”](https://mediasoup.org/documentation/v3/mediasoup/api/#dataProducer-on-transportclose)。客户端应该在对应的本地 data producer 中调用 `close()`。
*   DataConsumer [“transportclose”](https://mediasoup.org/documentation/v3/mediasoup/api/#dataConsumer-on-transportclose)。客户端应该在对应的本地 data consumer 中调用 `close()`。
*   DataConsumer [“dataproducerclose”](https://mediasoup.org/documentation/v3/mediasoup/api/#dataConsumer-on-dataproducerclose)。客户端应该在对应的本地 data consumer 中调用 `close()`。

在客户端或服务器端暂停 RTP 生产者或消费者时也会发生同样的情况。行为必须向对方发出信号。另外，服务器端应用程序应该监听以下事件并通知客户端：

*   Consumer [“producerpause”](https://mediasoup.org/documentation/v3/mediasoup/api/#consumer-on-producerpause)。客户端应该在对应的本地 transport 中调用 `pause()`。
*   Consumer [“producerresume”](https://mediasoup.org/documentation/v3/mediasoup/api/#consumer-on-producerresume)。客户端应该在对应的本地 transport 中调用 `resume()`(除非 consumer 本身也被故意暂停)。

当使用 simulcast 或 SVC 时，应用程序可能会对客户端和服务器端消费者之间的首选层和有效层感兴趣。

*   服务器端应用程序通过 [consumer.setPreferredLayers()](https://mediasoup.org/documentation/v3/mediasoup/api/#consumer-setPreferredLayers) 设置 consumer 首选层。
*   服务器端 consumer 订阅 [“layerschange”](https://mediasoup.org/documentation/v3/mediasoup/api/#consumer-on-layerschange) 事件，并通知客户端应用程序正在传输的有效层。

参考文档：
[Communication Between Client and Server](https://mediasoup.org/documentation/v3/communication-between-client-and-server/)
[Mediao Soup Demo协议分析](https://www.jianshu.com/p/36078f46a001)
[mediasoup protocol](https://mediasoup.org/documentation/v2/mediasoup-protocol/)

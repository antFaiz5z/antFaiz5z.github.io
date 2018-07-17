---
title: Library paho.mqtt.c API
comments: true
toc: true
date: 2018-07-13 16:35:34
categories: C/C++
tags: C/C++
---

# 前言

MQTT客户端分为同步客户端和异步客户端.　一般流程如下：

``` sh
  1.创建一个客户端对象；
  2.设置连接MQTT服务器的选项；
  3.如果多线程（异步模式）操作被使用则设置回调函数；
  4.订阅客户端需要接收的任意话题；
  5.重复以下操作直到结束：
    a.发布客户端需要的任意信息；
    b.处理所有接收到的信息；
  6.断开客户端连接；
  7.释放客户端使用的所有内存。
```

# 重要变量

## typedef void* MQTTClient

>A handle representing an MQTT client. A valid client handle is available following a successful call to MQTTClient_create().

使用MQTTClient_create()创建新连接前需预先定义一个MQTTClient，初始为空指针。

## typedef int MQTTClient_deliveryToken & <br> typedef int MQTTClient_token

>A value representing an MQTT message. A delivery token is returned to the client application when a message is published. The token can then be used to check that the message was successfully delivered to its destination.

一般用于异步客户端中，token用于记录当前客户端已发送message计数，deliveryToken用于记录确认已发送至服务器的message计数，二者相等即可安全断开客户端与服务器的连接。

# 重要结构体

## MQTTClient_connectOptions

>MQTTClient_connectOptions defines several settings that control the way the client connects to an MQTT server.
>Note: Default values are not defined for members of MQTTClient_connectOptions so it is good practice to specify all settings. If the MQTTClient_connectOptions structure is defined as an automatic variable, all members are set to random values and thus must be set by the client application. If the MQTTClient_connectOptions structure is defined as a static variable, initialization (in compliant compilers) sets all values to 0 (NULL for pointers). A keepAliveInterval setting of 0 prevents correct operation of the client and so you must at least set a value for keepAliveInterval.

客户端连接选项。

| 类型  | 变量名         | 描述 |
| -----| ---------     | ---- |
|char  | struct_id [4] | 必须为"MQTC"|
|int |struct_version|0: 无SSL选项和serverURIs;<br>1,2,3,4,5,6略|
|int |keepAliveInterval|最大通信间隔时间（秒）|
|int |cleansession|布尔值，是否清除session|
|int |reliable|布尔值。<br>1:上一条message发送完毕才发送下一条；<br> 0:最多允许10条message同时发送|
|MQTTClient_willOptions * |will|遗嘱选项|
|onst char * |username|用户名（MQTTv3.1.1提供）|
|const char * |password|密码（MQTTv3.1.1提供）|
|int |connectTimeout|连接超时|
|int |retryInterval|重试间隔|
|MQTTClient_SSLOptions *| ssl|SSL选项|
|int |serverURIcount|可选的serverURIs数组入口数量，默认0|
|char *const * |serverURIs|可选的serverURIs数组，形如：protocol://host:port|
|int |MQTTVersion|MQTT版本，默认先使用v3.1.1|
|struct {<br>const char *   serverURI <br>int MQTTVersion <br> int   sessionPresent<br>} |returned|连接返回值|
|struct {<br>int   len <br>const void *   data<br>} |binarypwd|二进制密码|

```c
#define MQTTClient_connectOptions_initializer { {'M', 'Q', 'T', 'C'}, 6, 60, 1, 1, NULL, NULL, NULL, 30, 20, NULL, 0, NULL, MQTTVERSION_DEFAULT, {NULL, 0, 0}, {0, NULL}, -1, 0}
```

## MQTTClient_willOptions

>MQTTClient_willOptions defines the MQTT "Last Will and Testament" (LWT) settings for the client. In the event that a client unexpectedly loses its connection to the server, the server publishes the LWT message to the LWT topic on behalf of the client. This allows other clients (subscribed to the LWT topic) to be made aware that the client has disconnected. To enable the LWT function for a specific client, a valid pointer to an MQTTClient_willOptions structure is passed in the MQTTClient_connectOptions structure used in the MQTTClient_connect() call that connects the client to the server. The pointer to MQTTClient_willOptions can be set to NULL if the LWT function is not required.

客户端遗嘱选项。当客户端意外丢失连接，服务器会替此客户端发送LWT主题的LWT消息，订阅了LWT主题的其他客户端可以此知道此客户端已断开连接。

| 类型  | 变量名         | 描述 |
| -----| ---------     | ---- |
|char |struct_id [4]|必须为"MQTW"|
|int |struct_version|布尔值。0:无二进制payload选项|
|const char * |topicName|LWT主题|
|const char * |message|LWT消息 |
|int |retained|是否保留LWT消息|
|int |qos|quality of service|
|struct {<br>int   len<br>const void *   data<br>}<br> |payload|二进制LWT payload|

```c
#define MQTTClient_willOptions_initializer { {'M', 'Q', 'T', 'W'}, 1, NULL, NULL, 0, 0, {0, NULL} }
```

## MQTTClient_SSLOptions

>MQTTClient_sslProperties defines the settings to establish an SSL/TLS connection using the OpenSSL library. It covers the following scenarios:
>Server authentication: The client needs the digital certificate of the server. It is included in a store containting trusted material (also known as "trust store").
>Mutual authentication: Both client and server are authenticated during the SSL handshake. In addition to the digital certificate of the server in a trust store, the client will need its own digital certificate and the private key used to sign its digital certificate stored in a "key store".
>Anonymous connection: Both client and server do not get authenticated and no credentials are needed to establish an SSL connection. Note that this scenario is not fully secure since it is subject to man-in-the-middle attacks.

客户端SSL选项。支持服务器认证（需提供服务器证书）、双向认证（需提供服务器证书及客户端自身私钥生成的证书）、匿名连接。

| 类型  | 变量名         | 描述 |
| -----| ---------     | ---- |
|char |struct_id [4]|必须为"MQTS"|
|int  |struct_version|必须为0|
|const char * |trustStore|PEM格式服务器证书|
|const char * |keyStore|PEM格式包括客户端的公共证书链及私钥|
|const char * |privateKey|PEM格式客户端私钥|
|const char * |privateKeyPassword|客户端私钥密码，若加密的话|
|const char * |enabledCipherSuites|密码套件列表|
|int |enableServerCertAuth|布尔值。是否启用服务器证书认证|
|int |sslVersion|SSL版本|
|int |verify|是否执行连接后检查，包括证书是否匹配主机名|
|const char *| CApath|包含PEM格式的CA证书的文件夹|

```c
#define MQTTClient_SSLOptions_initializer { {'M', 'Q', 'T', 'S'}, 2, NULL, NULL, NULL, NULL, NULL, 1, MQTT_SSL_VERSION_DEFAULT, 0, NULL }
```

## MQTTClient_message

>A structure representing the payload and attributes of an MQTT message. The message topic is not part of this structure.

客户端发送的消息（不含消息主题）。

| 类型  | 变量名         | 描述 |
| -----| ---------     | ---- |
|char  | struct_id [4] | 必须为"MQTM"  |
|int   | struct_version| 必须为0 |
|int   | payloadlen    | payload字节数  |
|void *| payload       | payload  |
|int   | qos           | quality of service  |
|int   | retained      | 1:MQTT服务器将会保留消息副本，并传送值此主题的新订阅者，新订阅者将会知晓这是MQTT保留的旧消息;<br>0:MQTT服务器不会保留此消息，订阅者认为此为普通消息  |
|int   | dup           | 布尔值。是否是重复消息  |
|int   | msgid         | message id  |
|MQTTProperties|properties|MQTT内容 |

```c
#define MQTTClient_message_initializer { {'M', 'Q', 'T', 'M'}, 1, 0, NULL, 0, 0, 0, 0, MQTTProperties_initializer }
```

# 重要函数

## MQTTClient_create

```c
int MQTTClient_create(MQTTClient* handle, const char* serverURI, const char* clientId, int persistence_type, void* persistence_context)
```

>This function creates an MQTT client ready for connection to the specified server and using the specified persistent storage (see MQTTClient_persistence). See also MQTTClient_destroy().

建立新连接。使用参数创建一个client，并将其赋值给之前声明的client

## MQTTClient_setCallbacks

```c
int MQTTClient_setCallbacks(MQTTClient handle, void* context, MQTTClient_connectionLost* cl, MQTTClient_messageArrived* ma, MQTTClient_deliveryComplete* dc)
```

>This function sets the callback functions for a specific client.If your client application doesn't use a particular callback, set the　relevant parameter to NULL. Calling MQTTClient_setCallbacks() puts the　client into multi-threaded mode. Any necessary message acknowledgements and　status communications are handled in the background without any intervention　from the client application. See @ref async for more information.

设置回调函数。

## MQTTClient_connect

```c
int MQTTClient_connect(MQTTClient handle, MQTTClient_connectOptions* options)
```

>This function attempts to connect a previously-created client (see MQTTClient_create()) to an MQTT server using the specified options. If you want to enable asynchronous message and status notifications, you must call MQTTClient_setCallbacks() prior to MQTTClient_connect().

## MQTTClient_subscribe

```c
int MQTTClient_subscribe(MQTTClient handle, const char* topic, int qos)
```

>Subscribe is synchronous.  QoS list parameter is changed on return to granted QoSs.Returns return code, MQTTCLIENT_SUCCESS == success, non-zero some sort of error (TBD)
>This function attempts to subscribe a client to a single topic, which may contain wildcards (see @ref wildcard). This call also specifies the @ref qos requested for the subscription (see also MQTTClient_subscribeMany()).

订阅某个主题。

取消订阅函数如下：

```c
int MQTTClient_unsubscribe(MQTTClient handle, const char* topic)
```

## MQTTClient_subscribeMany

```c
int MQTTClient_subscribeMany(MQTTClient handle, int count, char* const* topic, int* qos)
```

>This function attempts to subscribe a client to a list of topics, which may contain wildcards (see @ref wildcard). This call also specifies the @ref qos requested for each topic (see also MQTTClient_subscribe()).

同时订阅多个主题。

取消订阅函数如下:

```c
int MQTTClient_unsubscribeMany(MQTTClient handle, int count, char* const* topic)
```

## MQTTClient_publish

```c
int MQTTClient_publish(MQTTClient handle, const char* topicName, int payloadlen, void* payload, int qos, int retained, MQTTClient_deliveryToken* dt)
```

>This function attempts to publish a message to a given topic (see also MQTTClient_publishMessage()). An ::MQTTClient_deliveryToken is issued when this function returns successfully. If the client application needs to test for succesful delivery of QoS1 and QoS2 messages, this can be done either asynchronously or synchronously (see @ref async, ::MQTTClient_waitForCompletion and MQTTClient_deliveryComplete()).

## MQTTClient_publishMessage

```c
int MQTTClient_publishMessage(MQTTClient handle, const char* topicName, MQTTClient_message* msg, MQTTClient_deliveryToken* dt)
```

>This function attempts to publish a message to a given topic (see also MQTTClient_publish()). An ::MQTTClient_deliveryToken is issued when his function returns successfully. If the client application needs to test for succesful delivery of QoS1 and QoS2 messages, this can be done either asynchronously or synchronously (see @ref async, ::MQTTClient_waitForCompletion and MQTTClient_deliveryComplete()).

## MQTTClient_waitForCompletion

```c
int MQTTClient_waitForCompletion(MQTTClient handle, MQTTClient_deliveryToken dt, unsigned long timeout)
```

>This function is called by the client application to synchronize execution of the main thread with completed publication of a message. When called, MQTTClient_waitForCompletion() blocks execution until the message has been successful delivered or the specified timeout has expired. See @ref async.
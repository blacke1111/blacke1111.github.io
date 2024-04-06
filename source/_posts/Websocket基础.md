---
title: Websocket基础
date: 2023-09-05 20:50:10
categories: Websocket
---

# websocket介绍

 	WebSocket协议是基于TCP的一种新的网络协议。它实现了浏览器与服务器全双工(full-duplex)通信——允许服务器主动发送信息给客户端

 **websocket使用场景分享**:

            * 弹幕，
            * 网页聊天系统
            * 实时监控
            * 股票行情推送

# 框架介绍：

## Socketjs：

​	1、是一个浏览器JavaScript库，提供了一个类似WebSocket的对象。

​	 2、提供了一个连贯的跨浏览器的JavaScriptAPI，在浏览器和Web服务器之间创建了一个低延迟，全双工，跨域的通信通道

​	3、在底层SockJS首先尝试使用本地WebSocket。如果失败了，它可以使用各种浏览器特定的传输协议，并通过类似WebSocket的抽象方式呈现它们

​	4、SockJS旨在适用于所有现代浏览器和不支持WebSocket协议的环境。

​	学习资料：git地址：https://github.com/sockjs/sockjs-client

## stompjs：

​                      1、STOMP Simple (or Streaming) Text Orientated Messaging Protocol

​                      它定义了可互操作的连线格式，以便任何可用的STOMP客户端都可以与任何STOMP消息代理进行通信，以在语言和平台之间提供简单而广泛的消息互操作性（归纳一句话：是一个简单的面向文本的消息传递协)

​		学习资料:https://stomp-js.github.io/stomp-websocket/codo/class/Client.html#connect-dynamic





```java

@Configuration
@EnableWebSocket
public class WebSocketConfig  implements WebSocketMessageBrokerConfigurer {
    /**
     * 注册端点，发布或者订阅消息的时候需要连接端点
     * @param registry
     */
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        //该方法相当于一个基站
        registry.addEndpoint("/endpoint-websocket").setAllowedOrigins("*").withSockJS();
    }

    /**
     * 配置消息代理中介
     * enableSimpleBroker 服务端推送给客户端的路径前缀 （服务端推送数据给客户端是一对多    服务端@sendTO必须指定完整url）
     * setApplicationDestinationPrefixes 客户端推送给服务端的前缀  (客戶端推送消息給服务端是多对一  客户端请求url前缀必须指定，并且要在ApplicationDestinationPrefixes中配置)
     * @param registry
     */
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic","/chat");
        registry.setApplicationDestinationPrefixes("/app");
    }
}

```


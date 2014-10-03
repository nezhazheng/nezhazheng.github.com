---
layout: post
title:  "XMPP、Ejabberd与Smack快速入门"
---

> 需求：实现一个类微信的即时通信客户端（平台不限）。

> 马上可以想到的方案是客户端轮询服务器端获取聊天信息，这个方案的弊端是显而易见的，无论是性能和扩展性都非常差。所以咱们也就有了XMPP，也就有了这篇文章。


###XMPP
简单说[XMPP](xmpp.org)是由IETF制定的一个**可扩展**的即时消息通信协议。XMPP基本有两部分组成，一部分是它的核心协议，还有一部分是扩展协议。

其中核心协议主要由[RFC-6120](http://tools.ietf.org/search/rfc6120)、[RFC-6121](http://tools.ietf.org/search/rfc6121)、[RFC-6122](http://tools.ietf.org/search/rfc6122)组成。

还有一堆的扩展协议，具体可以参见XMPP官网扩展协议[列表部分](http://xmpp.org/xmpp-protocols/xmpp-extensions/)。一般会使用到得扩展协议包括[XEP-0160 XEP-0013离线消息处理](http://xmpp.org/extensions/xep-0160.html)、[xep-0045多用户聊天](http://xmpp.org/extensions/xep-0045.html)、[XEP-0096文件传输](http://xmpp.org/extensions/xep-0096.html)

XMPP具有的一些基本性质如下：

* 基于XML

* 标准

* 可扩展

###Ejabberd
[Ejabberd](http://www.process-one.net/en/ejabberd/)(Erlang)可以说是最棒的XMPP Server了吧，另外Openfire(Java)也是一个比较不错的XMPP Server实现。个人虽然是Java开发者，但还是更倾向于使用Erlang开发的Ejabberd，原因是Erlang自身具备的容错性和强大的无痛水平扩展能力确实比Java要高明很多。同时Ejabberd的安装和使用都非常爽快，属于是开箱即用类型的，而且文档也很齐全。

#####安装与启动

安装采用的是傻瓜式的安装，非源码安装。

1、Ejabberd的开发语言是Erlang，首先安装Erlang。

`apt-get install erlang`

2、安装Ejabberd

`apt-get install ejabberd`

3、启动一个ejabberd实例

`ejabberdctl start`

#####配置和管理

当你启动一个ejabberd实例后，事实上你就可以直接用了，当然如果你需要深入的使用一些特性，还是需要做一些简单的配置的。

ejabberd默认的配置文件在/etc/ejabberd/ejabberd.cfg。

有一些配置在配置文件中被注释了的，Erlang是用百分号%作为注释语法，如果要使用这个配置删除%号就可以了。

另外erlang提供了一个web管理界面，首先先通过下面的这条命令注册一个XMPP管理账号。

`ejabberdctl register user host password`

如果使用默认的端口配置的话就可以用这个链接http://server:5280/admin/登录到web管理控制台了。

###Smack

[Smack](http://www.igniterealtime.org/projects/smack/)是Java语言写的XMPP Library，个人观察下来貌似也是Java下最好得XMPP客户端了。下面演示一下实际IM场景中最基本的几个需求用Smack怎么来实现。

##### 链接与登录

localhost 也就是Ejabberd的服务的域名

{% highlight java %}
ConnectionConfiguration config = new ConnectionConfiguration("192.168.16.199", 5222, "localhost");
connection = new XMPPConnection(config);
connection.connect();
connection.login("nezhazheng", "123456", "test");
{% endhighlight %}

登录后可以在web管理界面中看到用户在线得状态，前提是与Ejabberd的连接不能断，比如你就不能紧接着运行`disconnect()`了，这个和数据库之类的连接比起来是一个很大的差别，原因是XMPP协议本身就是长连接的。

可以通过`setSendPresence(boolean sendPresence)`来控制隐身登录。

##### 聊天
发送
{% highlight java %}
ChatManager chatmanager = connection.getChatManager();
Chat newChat = chatmanager.createChat("whoyouwannatalk@localhost", new MessageListener() {
    public void processMessage(Chat chat, Message message) {
        System.out.println("Received message: " + message);
    }
});

try {
    newChat.sendMessage("hello world!");
}
catch (XMPPException e) {
    System.out.println("Error Delivering block");
}
{% endhighlight %}
接收
{% highlight java %}
connection.getChatManager().addChatListener(new ChatManagerListener() {
    @Override
    public void chatCreated(Chat chat, boolean createdLocally) {
        chat.addMessageListener(new MessageListener() {
            @Override
            public void processMessage(Chat chat, Message message) {
                System.out.println(message.getBody());
            }
        });
    }
});
{% endhighlight %}

##### 离线消息处理
下面的这种处理方式可以用于在线状态登录时离线消息的接收，如果需要在离线状态也可以接收到离线消息时，可以用`OfflineMessageManager`来控制，但是需要注意，比较坑爹的是Ejabberd的Community Server版本并不支持XEP-0013Flexible Offline Message Retrieval 这个协议，所以`OfflineMessageManager`也就用不了啦。

好在Smack在很多协议API中都有关于服务端是否支持此协议的API。比如下面这种

`public boolean supportsFlexibleRetrieval()`

{% highlight java %}
ConnectionConfiguration config = new ConnectionConfiguration("192.168.16.199", 5222, "localhost");
config.setSendPresence(true);
Connection conn = new XMPPConnection(config);
conn.connect();
conn.addPacketListener(new PacketListener() {
    @Override
    public void processPacket(Packet packet) {
        DelayInformation delay = (DelayInformation)packet.getExtension("x", "jabber:x:delay");
        if(delay == null) {
            return;
        }
        if(packet instanceof Message) {
            System.out.println(((Message)packet).getBody());
        }
    }
}, null);
conn.login("mahaiyue", "123456", "test");
{% endhighlight %}

##### 发送文件
发送
{% highlight java %}
FileTransferManager fileTransferManager = new FileTransferManager(connection);

// Create the outgoing file transfer
OutgoingFileTransfer transfer = fileTransferManager.createOutgoingFileTransfer("mahaiyue@localhost/test");
  
// Send the file
transfer.sendFile(new File(TestMessaging.class.getResource("origin").toURI()), "You won't believe this!");
Thread.sleep(5000);
System.out.println(transfer.getException());  
System.out.println(transfer.getProgress());  
System.out.println(transfer.isDone());  
{% endhighlight %}
接收
{% highlight java %}
manager.addFileTransferListener(new FileTransferListener() {
      public void fileTransferRequest(FileTransferRequest request) {
            // Check to see if the request should be accepted
            if(shouldAccept(request)) {
                  // Accept it
                  IncomingFileTransfer transfer = request.accept();
                  try {
                    transfer.recieveFile(new File("received.txt"));
                }
                catch (XMPPException e) {
                    e.printStackTrace();
                }
            } else {
                  // Reject it
                  request.reject();
            }
      }

    private boolean shouldAccept(FileTransferRequest request) {
        System.out.println(request.getMimeType());
        return true;
    }
});
{% endhighlight %}

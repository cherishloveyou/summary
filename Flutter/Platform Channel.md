## **Flutter通信机制——Platform Channel工作原理**

Flutter定义了三种不同类型的Channel，它们分别是

- BasicMessageChannel：用于传递字符串和半结构化的信息。

- MethodChannel：用于传递方法调用（method invocation）。

- EventChannel: 用于数据流（event streams）的通信。

  

三种Channel之间互相独立，各有用途，但它们在设计上却非常相近。每种Channel均有三个重要成员变量：

- name: String类型，代表Channel的名字，也是其唯一[标识符](https://zhida.zhihu.com/search?content_id=207041319&content_type=Article&match_order=1&q=标识符&zhida_source=entity)。
- messager：BinaryMessenger类型，代表消息信使，是消息的发送与接收的工具。
- codec: MessageCodec类型或MethodCodec类型，代表消息的[编解码器](https://zhida.zhihu.com/search?content_id=207041319&content_type=Article&match_order=1&q=编解码器&zhida_source=entity)。





### **使用方法**

### **BasicMessageChannel**

### **Android端：**

```text
BasicMessageChannel mBasicMessageChannel = new BasicMessageChannel(getFlutterView(), "basic_channel", StringCodec.INSTANCE);
mBasicMessageChannel.setMessageHandler(new BasicMessageChannel.MessageHandler() {
    //接受消息
    @Override
    public void onMessage(Object o, BasicMessageChannel.Reply reply) {
        Log.e("basic_channel", "接收到来自flutter的消息:"+o.toString());
        reply.reply("回馈消息");
    }
});
//发送消息
mBasicMessageChannel.send("向flutter发送消息");
//发送消息并接受flutter的回馈
mBasicMessageChannel.send("向flutter发送消息", new BasicMessageChannel.Reply() {
            @Override
            public void reply(Object o) {
                
            }
});
```

### **Flutter端：**

```text
const basicMessageChannel = const BasicMessageChannel('basic_channel', StringCodec());
//接受并回复消息
basicMessageChannel.setMessageHandler(
      (String message) => Future<String>(() {
            setState(() {
              this.message = message;
            });
            return "回复native消息";
      }),
);
//发送消息
basicMessageChannel.send("来自flutter的message");
//flutter并没有发送并接受回复消息的`send(T message, BasicMessageChannel.Reply<T> callback)`方法
```

### **MethodChannel**

Android端：

```text
MethodChannel mMethodChannel = new MethodChannel(getFlutterView(), "method_channel");
mMethodChannel.setMethodCallHandler(new MethodChannel.MethodCallHandler() {
    //响应flutter端的调用
    @Override
    public void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
        if (methodCall.method.equals("noticeNative")) {
            todo()
            result.success("接受成功");
        }
    }
});
//原生调用flutter
mMethodChannel.invokeMethod("noticeFlutter", "argument", new MethodChannel.Result() {
    @Override
    public void success(Object o) {
        //回调成功
    }
    @Override
    public void error(String s,String s1, Object o) {
        //回调失败
    }
    @Override
    public void notImplemented() {

    }
});
```

Flutter端：

```text
const methodChannel = const MethodChannel('method_channel');
Future<Null> getMessageFromNative() async {
    //flutter调原生方法
    try {
      //回调成功
      final String result = await methodChannel.invokeMethod('noticeNative');
      setState(() {
        method = result;
      });
    } on PlatformException catch (e) {
      //回调失败
    }
  }
methodChannel.setMethodCallHandler(
      (MethodCall methodCall) => Future<String>(() {
            //响应原生的调用
          if(methodCall.method == "noticeFlutter"){
            setState(() {
              
            });
          }
      }),
); 
```

### **EventChannel**

Android端：

```text
EventChannel eventChannel = new EventChannel(getFlutterView(),"event_channel");
eventChannel.setStreamHandler(new EventChannel.StreamHandler() {
    @Override
    public void onListen(Object o, EventChannel.EventSink eventSink) {
        eventSink.success("成功");
        //eventSink.error("失败","失败","失败");
    }
    @Override
    public void onCancel(Object o) {
        //取消监听时调用
    }
});
```

Flutter端：

```text
const eventChannel = const EventChannel('event_channel');
eventChannel.receiveBroadcastStream().listen(_onEvent,onError:_onError);
void _onEvent(Object event) {
    //返回的内容
}
void _onError(Object error) {
    //返回的回调
}
```

其中：Object args是传递的参数，EventChannel.EventSink eventSink是Native回调Dart时的会[回调函数](https://zhida.zhihu.com/search?content_id=207041319&content_type=Article&match_order=1&q=回调函数&zhida_source=entity)，eventSink提供success、error与endOfStream三个回调方法分别对应事件的不同状态

### **源码初探**

### **Platform Channel基本结构**

首先了解一下这三种Channel的代码：

### **BasicMessageChannel**

```text
class BasicMessageChannel<T> {
  const BasicMessageChannel(this.name, this.codec);
  final String name;
  final MessageCodec<T> codec;
  Future<T> send(T message) async {
    return codec.decodeMessage(await BinaryMessages.send(name, codec.encodeMessage(message)));
  }
  void setMessageHandler(Future<T> handler(T message)) {
    if (handler == null) {
      BinaryMessages.setMessageHandler(name, null);
    } else {
      BinaryMessages.setMessageHandler(name, (ByteData message) async {
        return codec.encodeMessage(await handler(codec.decodeMessage(message)));
      });
    }
  }
  void setMockMessageHandler(Future<T> handler(T message)) {
    if (handler == null) {
      BinaryMessages.setMockMessageHandler(name, null);
    } else {
      BinaryMessages.setMockMessageHandler(name, (ByteData message) async {
        return codec.encodeMessage(await handler(codec.decodeMessage(message)));
      });
    }
  }
}
```

### **MethodChannel**

```text
class MethodChannel {
  const MethodChannel(this.name, [this.codec = const StandardMethodCodec()]);
  final String name;
  final MethodCodec codec;
  void setMethodCallHandler(Future<dynamic> handler(MethodCall call)) {
    BinaryMessages.setMessageHandler(
      name,
      handler == null ? null : (ByteData message) => _handleAsMethodCall(message, handler),
    );
  }
  void setMockMethodCallHandler(Future<dynamic> handler(MethodCall call)) {
    BinaryMessages.setMockMessageHandler(
      name,
      handler == null ? null : (ByteData message) => _handleAsMethodCall(message, handler),
    );
  }
  Future<ByteData> _handleAsMethodCall(ByteData message, Future<dynamic> handler(MethodCall call)) async {
    final MethodCall call = codec.decodeMethodCall(message);
    try {
      return codec.encodeSuccessEnvelope(await handler(call));
    } on PlatformException catch (e) {
      returun ...
    } on MissingPluginException {
      return null;
    } catch (e) {
      return ...
    }
  }
  Future<T> invokeMethod<T>(String method, [dynamic arguments]) async {
    assert(method != null);
    final ByteData result = await BinaryMessages.send(
      name,
      codec.encodeMethodCall(MethodCall(method, arguments)),
    );
    if (result == null) {
      throw MissingPluginException('No implementation found for method $method on channel $name');
    }
    final T typedResult = codec.decodeEnvelope(result);
    return typedResult;
  }
}
```

### **EventChannel**

```text
class EventChannel {
  const EventChannel(this.name, [this.codec = const StandardMethodCodec()]);
  final String name;
  final MethodCodec codec;
  Stream<dynamic> receiveBroadcastStream([dynamic arguments]) {
    final MethodChannel methodChannel = MethodChannel(name, codec);
    StreamController<dynamic> controller;
    controller = StreamController<dynamic>.broadcast(onListen: () async {
      BinaryMessages.setMessageHandler(name, (ByteData reply) async {
        ...
      });
      try {
        await methodChannel.invokeMethod<void>('listen', arguments);
      } catch (exception, stack) {
        ...
      }
    }, onCancel: () async {
      BinaryMessages.setMessageHandler(name, null);
      try {
        await methodChannel.invokeMethod<void>('cancel', arguments);
      } catch (exception, stack) {
        ...
      }
    });
    return controller.stream;
  }
}
```

这三种Channel都有两个成员变量：

- name:表示Channel名字,用于区分不同Platform Channel的唯一标志，每个Channel使用唯一的name作为其唯一标志
- codec: 表示消息的编解码器,Flutter采用了二进制[字节流](https://zhida.zhihu.com/search?content_id=207041319&content_type=Article&match_order=1&q=字节流&zhida_source=entity)作为数据传输协议:发送方需要把[数据编码](https://zhida.zhihu.com/search?content_id=207041319&content_type=Article&match_order=1&q=数据编码&zhida_source=entity)成二进制数据,接受方再把数据解码成原始数据.而负责编解码操作的就是Codec。 每个Channel中都使用到了`BinaryMessages`，它起到了信使的作用，负责将信息进行跨平台的搬运，是消息发送和接受的工具。

### **setMessageHandler**

在创建好`BasicMessageChannel`后，让其接受来自另一平台的消息，`BinaryMessenger`调用它的`setMessageHandler`方法为其设置一个消息处理器,配合`BinaryMessenger`完成消息的处理以及回复；

### **send**

在创建好`BasicMessageChannel`后，可以调用它的send方法向另一个平台传递数据。

### **setMethodCallHandler**

设置用于在此`MethodChannel`上接收方法调用的回调

### **receiveBroadcastStream**

设置广播流以接收此`EventChannel`上的事件

### **Handler**

Flutter使用Handler处理Codec解码后的消息。三种Platform Channel相对应,Flutter中也定义了三种Handler:

- MessageHandler: 用于处理字符串或者半结构化消息,定义在BasicMessageChannel中.
- MethodCallHandler: 用于处理方法调用,定义在MethodChannel中.
- StreamHandler: 用于事件流通信,定义在EventChannel中

使用Platform Channel时，需要为其注册一个对应BinaryMessageHandler为其设置对应的Handler。二进制数据会被BinaryMessageHanler进行处理,首先使用Codec进行解码操作,然后再分发给具体Handler进行处理。

## **文末**

在Flutter与Native混合开发的模式下，Platform Channel的应用场景非常多，理解Platform Channel的工作原理，有助于我们在从事这方面开发时能做到得心应手。

跟完`MethodChannel`的源码，会发现整个通信机制还挺简单的，先去不去理解Codec的话，等于就是将dart的变量，传到dart Native，然后交到java Native， 再传到java。然后相反的路径，再从java到dart。

然后再去看`BasicMessageChannel`就是没有`MethodCall`这个结构的，其他的也是走的`BinaryMessages.send`方法。然后在Android端，没有=`IncomingMethodCallHandler`这个类，直接就是`BinaryMessageHandler`。所以了解了`MethodChannel`，`BasicMessageChannel`原理自然就懂了。

同样的`EventChannel`则是基于`MethodChannel`来实现的，只是两端的handler会有一些特殊的处理方式，这个倒是与通信没有多大关系了，不过设计的也很简单，比较有意思。

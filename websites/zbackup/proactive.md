##### Proactive具体实现设计原理

> 实现异步操作的处理器

- 定义一系列异步操作的API

```java
// 客户端调用类
abstract class AsyncStream {
    
    public AsyncStream(CompletionHandler handler, Handler handler, Proactor proactor){
        //...
    }
    
    int asyncRead(ByteBuffer readByte, long start, AsyncReadResult result);
    
    int asyncWrite(ByteBuffer writeByte, long start, AsyncWriteResult result);
}


// 定义异步回调,内核执行回调操作并返回异步结果
interface AsyncResult {
    
    long bytesTransferred();
    
    long error();
    
    void callback();
}

// 处理读取
class AsyncReadResult implements AsyncResult {
     
}

// 处理写出
class AsyncWriteResult implements AsyncResult {
    
}
```


> Proactor实现

Proactor实现机制需要包含以下几方面:
- 如何定义一个多路复用器将结果通知到completion handler的策略

```text
在Proactor模式中，当调用异步操作时，关联的Proactor使用异步完成令牌模式将状态与操作关联起来。
此状态通常涉及对完成处理程序的引用和客户端提供的异步完成令牌(ACT)。
然后，将包含此状态的对象作为动作传递给异步操作处理器，该处理器在异步完成令牌模式中扮演服务的角色
```

- 实现结果转发机制
- 回调类: 对外暴露回调函数接口,负责将异步操作完成的结果的传递
- 

#### Logcabin 介绍

Logcabin 是 Raft 作者 (https://ongardie.net/) 为了介绍它的论文 Raft 实现的一个简单分布式存储，是最早的一版本 Raft 工业实现了。

#### Rpc 模块介绍

Rpc 是远程过程调用的缩写，是分布式系统中最基础的组建，用来实现节点之间的通信。下面是 logcanbin 中 RPC 模块的类结构：

![rpc class](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/rpcCore.svg)


[查看大图](rpcCore.svg)

上述类图还是很复杂的，为了快速入口理解这个框架，我们可以先从它的使用开始看，作者在 ServerTest.cc 文件中给了如何使用的示例，让我们来分析一下下面这段摘录的代码：

```
class RPCServerTest : public ::testing::Test {
    RPCServerTest()
        : eventLoop()
        , eventLoopThread(&Event::Loop::runForever, &eventLoop)
        , address("127.0.0.1", DEFAULT_PORT)
        , server(eventLoop, MAX_MESSAGE_LENGTH)
        , session()
        , service1(std::make_shared<ServiceMock>())
        , service2(std::make_shared<ServiceMock>())
        , service3(std::make_shared<ServiceMock>())
        , request()
        , reply()
    {
        address.refresh(Address::TimePoint::max());
        EXPECT_EQ("", server.bind(address));
        session = ClientSession::makeSession(
                        eventLoop,
                        address,
                        MAX_MESSAGE_LENGTH,
                        RPC::ClientSession::TimePoint::max(),
                        Core::Config());
        request.set_field_a(3);
        request.set_field_b(4);
        reply.set_field_a(5);
        reply.set_field_b(6);
    }
    void deinit() {
        eventLoop.exit();
        if (eventLoopThread.joinable())
            eventLoopThread.join();
    }
    void childDeathInit() {
        assert(!eventLoopThread.joinable());
        eventLoopThread = std::thread(&Event::Loop::runForever, &eventLoop);
    }
    ~RPCServerTest() {
        deinit();
    }
    Event::Loop eventLoop; // 事件循环 
    std::thread eventLoopThread; // 事件循环线程
    Address address; // 地址
    Server server; // 服务端对象
    std::shared_ptr<ClientSession> session; // 客户端到服务端的会话
    std::shared_ptr<ServiceMock> service1; // 
    std::shared_ptr<ServiceMock> service2;
    std::shared_ptr<ServiceMock> service3;
    ProtoBuf::TestMessage request;
    ProtoBuf::TestMessage reply;
};
```

RPCServerTest 是我们测试的基类，主要定义了时间循环，运行事件循环的线程


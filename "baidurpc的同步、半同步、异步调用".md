---
title: "baidurpc的同步、半同步、异步调用"
author: 一张狗
lastmod: 2019-07-06 07:45:45
date: 2018-09-20 20:28:48
tags: []
---



## 一、ParallelChannel组合访问

对多个下游调用的抽象，它们的组合仍是同一种结构，用户可以便用统一接口完成同步、异步、取消等操作。

int AddChannel(baidu::rpc::ChannelBase* sub_channel, ChannelOwnership ownership, CallMapper* call_mapper, ResponseMerger* response_merger);

用法：把下游的channel作为sub_channel  加入到ParallelChannel中;

CallMapper：用于把对ParallelChannel的调用转化为对sub channel的调用，需要在这里调用具体下游；

ResponseMerger：response_merger把sub channel的response合并入总的response，如果你需要更复杂的行为，则需实现ResponseMerger。

例子：
```cpp
class UseFieldAsSubRequest : 
public CallMapper { 
    public: 
        SubCall Map(int channel_index/*starting from 0*/, 
                    const google::protobuf::MethodDescriptor* method, 
                    const google::protobuf::Message* request,
                    google::protobuf::Message* response) { 
            if (channel_index >= request->sub_request_size()) { 
            // sub_request不够，说明外面准备数据的地方和pchan中sub channel的个数不符. 
            // 返回Bad()让该次访问立刻失败 
            return SubCall::Bad(); 
        } 
        // 取出对应的sub request，增加一个sub response，最后的flag为0告诉pchan什么都不用删 
        return SubCall(sub_method, request->(channel_index), response->add_sub_response</span>(), 0); 
    } 
};
```

sub_method就是代表可能和method不同，但一般是相同的，因为大部分情况用这个是为了”parallel地“访问同一种method。如果不同的话，一般是访问google::protobuf::ServiceDescriptor::method(int index)获得对应的method，例子：
```
channel.AddChannel(&subchans[i], 
                    rpc::DOESNT_OWN_CHANNEL, 
                    new GetReqAndAddRes, 
                    new MergeNothing)
class GetReqAndAddRes : 
public brpc::CallMapper { 
    brpc::SubCall Map( int channel_index, 
                        const google::protobuf::MethodDescriptor* method,
                        const google::protobuf::Message* req_base, 
                        google::protobuf::Message* res_base) { 
        const test::ComboRequest* req = dynamic_cast<const test::ComboRequest*>(req_base); 
        test::ComboResponse* res = dynamic_cast<test::ComboResponse*>(res_base); 
        if (method->name() != "ComboEcho" || res == NULL || req == NULL || req->requests_size() <= channel_index) { 
            return brpc::SubCall::Bad(); 
        }
        return brpc::SubCall(<::test::EchoService::descriptor()->method(0), 
                            &req->requests(channel_index), 
                            res->add_responses(), 0); 
        } 
};
```
test

PS：http方式是支持ParallelChannel的，但是只能支持不同Server发送相同的请求。这是由于http只能使用CallMethod设置request和response为NULL的方式调用，没法在CallMapper和ResponseMerger中对不同的Server构造不同的请求并Merge应答。如果需要向不同的http server发送不同的请求，最简单的方式就是使用半同步的方式来实现ParallelChannel，也可以通过CountDownEvent和bthread_start_backgroud实现，或者是定制CallMethod的Done来实现。


## 二、同步调用
```
MyRequest request; 
MyResponse response; 
baidu::rpc::Controller cntl; 
XXX_Stub stub(&channel); 
request.set_foo(...); 
cntl.set_timeout_ms(...); 
stub.some_method(&cntl, &request, &response, NULL); 
if (cntl.Failed()) { 
    // RPC出错了 
} else { 
// RPC成功了，response里有我们想要的回复数据。 
}
```

## 三、异步回调

异步访问指的是给CallMethod传递一个额外的回调对象done，CallMethod会在发出request后就结束了，而不是在RPC结束后。当server端返回response或发生错误（包括超时）时，done->Run()会被调用。对RPC的后续处理应该写在done->Run()里，而不是CallMethod后。

http：`channel.CallMethod(NULL, &cntl, NULL, NULL, NULL/*done*/);`

HTTP和protobuf无关，所以除了Controller和done，CallMethod的其他参数均为NULL。如果要异步操作，最后一个参数传入done。

例子：
```
static void OnRPCDone(MyResponse* response, 
                    baidu::rpc::Controller* cntl) { 
// unique_ptr会帮助我们在return时自动删掉response/cntl，防止忘记。gcc 3.4下的unique_ptr是public/common提供的模拟版本。 
    std::unique_ptr<MyResponse> response_guard(response);
    std::unique_ptr<baidu::rpc::Controller> cntl_guard(cntl); 
    if (cntl->Failed()) { 
        // RPC出错了. response里的值是未定义的，勿用。 
    } else { 
        // RPC成功了，response里有我们想要的数据。开始RPC的后续处理。 
    } // NewCallback产生的Closure会在Run结束后删除自己，不用我们做。 
} 
MyResponse* response = new MyResponse; 
baidu::rpc::Controller* cntl = new baidu::rpc::Controller; 
MyService_Stub stub(&channel); 
MyRequest request; // you don't have to new request, even in an asynchronous 
call. request.set_foo(...); 
cntl->set_timeout_ms(...); 
stub.some_method(cntl, &request, response, google::protobuf::NewCallback(OnRPCDone, response, cntl)); 
```
例

异步访问之后可通过CountdownEvent等所有的请求返回后合并：

CountdownEvent which provides a synchronization primitive that a threads would be block until the counter reaches 0

也可以通过Join等待结束：
```
const baidu::rpc::CallId cid1 = controller1->call_id(); 
const baidu::rpc::CallId cid2 = controller2->call_id(); 
... 
stub.method1(controller1, request1, response1, done1); 
stub.method2(controller2, request2, response2, done2); 
... 
baidu::rpc::Join(cid1); baidu::rpc::Join(cid2);
```

**发起RPC前**调用Controller.call_id()获得一个id，发起RPC调用后Join那个id。

Join()的行为是等到**RPC结束且调用了done后**，一些Join的性质如下：

- 如果对应的RPC已经结束，Join将立刻返回。
- 多个线程可以Join同一个id，RPC结束时都会醒来。
- 同步RPC也可以在另一个线程中被Join，但一般不会这么做。


## 四、半同步

Join可用来实现“半同步”操作：即等待多个异步操作返回。由于调用处的代码会等到多个RPC都结束后再醒来，所以controller和response都可以放栈上。
```
baidu::rpc::Controller cntl1; 
baidu::rpc::Controller cntl2; 
MyResponse response1; 
MyResponse response2; 
... 
stub1.method1(&cntl1, &request1, &response1, baidu::rpc::DoNothing()); 
stub2.method2(&cntl2, &request2, &response2, baidu::rpc::DoNothing()); 
... 
baidu::rpc::Join(cntl1.call_id()); 
baidu::rpc::Join(cntl2.call_id());
```

baidu::rpc::DoNothing()可获得一个什么都不干的done，专门用于半同步访问。它的生命周期由框架管理，用户不用关心。

注意在上面的代码中，我们在RPC结束后又访问了controller.call_id()，这是没有问题的，因为DoNothing中并不会像上面的on_rpc_done中那样删除Controller。

***其实异步和半同步的区别就是半同步的回调那里是baidu::rpc::DoNothing，因为最后Join那里是等所有request都返回然后再唤醒的。***



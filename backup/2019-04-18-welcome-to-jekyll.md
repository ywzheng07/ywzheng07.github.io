---
title: "gRPC的C++应用详解 "
date: 2020-05-09
categories:
  - blog
tags:
  - gRPC
  - C++	
---

本文主要针对希望对gRPC入门的读者，读者需要有最最基本的C++语法知识，但完全不需要是C++专家，多数的代码都以直白的语言进行了解释。 

## RPC 简介

RPC (Remote Procedure Call)即远端程序呼叫，指由一台电脑的程序呼叫另一个地址空间(Address Space)的程序，在这里另一个地址空间可以是本机上的另一个进程，也可以是另一台电脑的进程。因此，RPC也属于一种进程间通讯模式(Interprocess Communication)。

通过RPC，客户端可以如同调用本进程内的程序一样调用远程服务器上的程序。

#### 为什么要用RPC框架

如果没有RPC框架，试想客户端需要调用服务器上如下的一个简单程序：

``` c++
int sum_int(int a, int b){
    return a+b
};
```

整个调用过程需要：

1. 客户端与服务器建立通讯；
2. 建立通讯后，客户端将需要调用的函数sum_int及函数的参数a和b编码打包之后传输给服务器（即Marshalling）；
3. 服务器收到字符串解码(Unmarshalling)。这里需要假设客户端与服务器提前关于编码已有协议；
4. 调用合适的函数，计算出结果；
5. 服务器将结果编码及序列化之后传输给客户端；
6. 客户端收到结果后进行解析，得到最终结果；

以上的各个过程如果全都没有标准化流程，将是一个浩大的工作量，许多的时间将被花费在通讯，编码和错误处理(Error handling)等细节问题上。

因此，历史上出现了许多标准化的RPC框架，将通讯，编码等细节问题全部自动化，客户端和服务器的程序员只需要考虑具体调用的程序的实现。其中，gRPC就是Google推出的一个简洁高效的RPC框架。

#### RPC框架概览

RPC框架的一个重要功能，就是提供标准化的接口界面。这个语言被称为IDL(Interface description language，可以参考[维基百科](https://en.wikipedia.org/wiki/Interface_description_language)) ，它并不针对特定语言(language agnostic)，是一种纯粹用来描述接口的语言，由此使得通讯本身不受所使用的语言影响，并使得不同语言编写的程序可以交流。接口界面的定义中，一般包含参数和程序两种，RPC框架会根据定义文件，生成对应的对象和框架，供程序员调用。框架的概览如下：

* 服务器端：服务器端会生成所有在界面接口中定义好的参数及函数，同时会搭建好所有与通讯、编码/解码相关的代码。程序员只需要具体实现前面被定义的函数即可。在C++中，这可以简单地通过函数重写(override)来实现。
* 客户端：客户端也会生成界面接口定义好的参数，同时会生成一个“桩”(stub)，即客户端与服务器通讯的接口。客户端就可以通过该接口调用服务器的函数。

同样，如果以上概念性的讲解不是很明白，不用担心，后面我们会用gRPC来举例说明，我个人觉得在例子中会更好理解， 建议标注出不理解的地方，然后看完例子后再回来看一下这段话，会更容易理解。

## gRPC 简介

gRPC就是目前最为广泛使用的RPC框架之一，接下来，我们继续沿用上述的整数相加函数作为例子，详细描述gRPC的使用方法。使用gRPC在C++实现客户端调用服务器这个函数可以分为以下几个步骤：

1. 用IDL定义接口界面，包括传输的参数和函数类型。
2. 编译定义好的IDL
3. 在服务器端实现函数
4. 客户端调用函数。
5. 编译和运行

#### 定义Protobuf的接口界面

gRPC的IDL叫做Protocol Buffers(Protobuf)，它允许我们定义两种类型的接口，一个是消息(message)，一个是服务(service)。

客户端每次调用函数的时候，会向服务器发送一个信息，在Protobuf中被定义为消息(message)。注意，客户端每次调用函数时只能发送**一条**message，但可以把所有参数放到message里面。在我们的例子中，我们有两个需要相加的整数`a`和`b` ，需要将它们放到同一条message里。出于演示的目的，我们加入一条文本信息来表明客户端的姓名，则可以定义该message为：

```protobuf
syntax = "proto3"; //申明用proto3的语法，否则会默认使用proto2。
message Numbers{
	int64 a = 1;
    int64 b = 2;
    string name = 3;
}
```

上面的代码中，message是类型，Numbers是我们给予这个message的名字，因为这个例子比较简单，`a`和`b`只是两个整数而已，因此我们将名字命名为Numbers，即这是客户端发给服务器的请求中所必须包含的参数。现实中，请按照一般的命名规则进行适当的命名。

值得注意的是，message里的每一个条目包含三部分内容，即类型，名字，编号，这篇[官方文档](https://developers.google.com/protocol-buffers/docs/proto3)可以找到支持的数据类型的名单，名字可以自定义，而编号应该在1-536,870,911之间，是在二进制编码中用来唯一识别不同条目的，在每个message内部，编码应该唯一。出于编码效率的考虑，可以尽量使用较小的数字。

通过这个简单的结构，无论客户端需要发给服务器多少内容，均可以打包在message里面。同样，服务器的回复虽然只包含一个整型，但也应该被打包在这样的一个message里：

``` protobuf
message Result{
	int64 res = 1;
}
```

所以，针对每一个函数，我们应该包含一个request message和一个response message，在外面这个例子中，客户需要发送的request即Numbers类型，而服务器返回的response即Result型。注意如果有多个函数的情况下，可以重复使用这些message，下面用例子来说明。

除了定义message之外，我们还需要在Protobuf里面声明函数。在整个服务类下面，可以用rpc关键词定义不同的类函数(method)。我们这里定义了两个类函数，整数加法(intSum)和整数乘法(intMult)。该函数需要获取一个Numbers型(之前已经定义)作为函数的输入参数，同时，该函数将会返回一个Result型。正如前文所述，已经定义的message可以用在不同的函数之中。

```protobuf
syntax = "proto3"; //申明用proto3的语法，否则会默认使用proto2。
package simplemath; //即该文件中定义的所有类将会在simplemath这个作用域内
service SimpleMath{ //声明整个服务器函数，将会被编译为一个类
	//声明加法函数的接口
	rpc intSum(Numbers) returns (Result) {};
	//声明乘法函数的接口
	rpc intMult(Numbers) returns (Result) {};
}
```

完整的proto文件[见此](https://github.com/ywzheng07/gRPCSimpleMath/blob/master/simple_math.proto)。

针对protobuf支持的数据类型等信息，可以参考[官方文档](https://developers.google.com/protocol-buffers/docs/proto3)。

#### 编译Protobuf

在[安装](https://github.com/grpc/grpc/blob/v1.28.1/src/cpp/README.md#make)好gRPC的基础上，我们这里的`simple_math.proto`的编译可以通过以下命令进行编译，为了方便读者，我们也在Makefile里提供了编译proto的命令，可以直接通过`make proto`进行编译。

``` bash
protoc --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` simple_math.proto
protoc --cpp_out=. simple_math.proto
```

这个命令会在同目录下，根据`simple_math.proto`这一文件生成对应的C++头文件和源文件，一共有四个文件，具体如下：

- `simple_math.grpc.pb.h`该文件是服务类(service class)的头文件，在我们这里，定义了SimpleMath这个服务(service)的类(class)。
- `simple_math.grpc.pb.cc`该文件是服务类(service class)的源文件，包括了各种序列化，通讯等代码，同时也对我们在protobuf里面声明的两个类函数进行了定义，但具体实现需要我们来写。
- `simple_math.pb.h`该文件包含了消息类(message class)的头文件，在我们这里，包括Numbers和Result.
- `simiple_math.pb.cc` 该文件是对消息类(message class)的具体实现。

这些生成的文件**不要修改**，可以打开看一看具体的实现，不过这些代码比较复杂，而且很多地方调用了grpc库里的函数，看起来会比较吃力。如果只是想用这个框架，我们只需要知道接口就可以了。C++中常用的接口可以参考这篇[官方文档](https://developers.google.com/protocol-buffers/docs/cpptutorial)。我们这里总结一下最常用的几个类函数，并在后续的实际使用中介绍。

对于message中定义的每一个字段(field)，会有`“set_”,"has_", "clear_"`这几个类函数可以调用。正如命名所体现的，其功能分别为，“设置该字段的值”，“该字段是否已设置值”以及"清除该字段的值“。假设定义nums为一个Numbers型，我们可以利用`nums.set_name()`来设定nums中的name的值，用`nums.has_name()`来判定nums中name这个字段是否已经被设置，然后利用`nums.clear_name()`来清除nums中已经设定的name值。

另外值得注意的是，在消息类里面，以message中定义的字段均为私有，如果需要调用，需要通过同名的类函数调用，在以上的例子里，如果想调用Numbers的name，应该用`nums.name()`。

#### 在服务器端实现函数

当protofbuf被编译后，我们就可以服务器和客户端的实现文件中调用编译好的消息类(message class)和服务类(service class)，这一点只需要包含生成的protobuf头文件即可，在我们的例子中加入`#include "simple_math.grpc.pb.h"`即可。一般而言服务类的实现头文件中已经包含了消息类，所以不需要重复包含。

实现服务类函数可以通过继承已经写好的服务类，并重写(override)对应的类函数来实现，这里官方推荐的命名方式为类+Impl。注意在服务器端和客户端中用到的的许多标识符或者类型是在grpc或者simplemath的命名空间内，具体可以参见[源代码]()。

在服务器端，我们将先定义：

``` c++
class SimpleMathImpl : public SimpleMath::Service{
    ...
}
```

然后在这个子类中对rpc服务进行具体实现。在这里我们给出整数相加这个函数的实现，读者可以实现一下整数相乘作为练习。

```c++
class SimpleMathImpl : public SimpleMath::Service{
	Status intSum(ServerContext* context,const Numbers* nums, Result* result) override{
		std::cout<<"Processing sum request from client"<<nums->name()<<std::endl;
		int res = nums->a() + nums->b();
		//回复的内容装在Result中，返回值是处理状态.
		result->set_res(res);
        //这个简单例子中我们没有错误处理，因此总是返回OK。
		//但是注意如果传输出现问题，客户端依然可能受到非OK的返回值。
		return Status::OK;
	}
}
```

本函数比较简单，注意这个类函数已经在头文件中被定义好了，一定要遵守传参规则，即接受三个参数，类型分别为服务器端配置类(ServerContext)，请求类（在我们的例子中是一个Numbers类，这是我们自己在protobuf中规定的），返回类（在我们的例子中是一个Result类，也是我们自己在protobuf中规定的）。

实现的过程唯一需要注意的是函数返回的是处理状态，根据处理情况可以有多个状态类型可以返回。例如在文件服务器中，可能出现文件"没有找到"("Not Found")的错误信息，在这种情况下可以返回：

```c++
return Status(StatusCode::Not_Found, "File Not Found") 
```

gRPC还提供了许多类型的返回状态，可以参考这篇[官方API](https://developers.google.com/maps-booking/reference/grpc-api-v2/status_codes)，或者在写代码的时候利用自动补全，选择合适的返回类型。

在返回值上值得注意的是，客户端收到的状态并不永远等于服务器的类函数返回的状态，实际的返回值还要经过通讯过程的错误处理。一个简单的例子是在超时(timeout)的情况下，服务器的这个类函数可能返回了OK，但通讯已经超时，因此客户端收到的状态为Timeout，而非OK。 有关超时的具体介绍，可以参见这篇[官方文档](https://grpc.io/blog/deadlines/)。

写好客户端的服务类之后，再通过gRPC提供的ServerBuilder建立服务器。新建builder之后，首先向builder提供服务器的地址/端口信息，及加密证书。这里我们用本地地址即47852这个端口。同时选择无需加密或认证，直接用gRPC提供的未加密认证，有关其他的认证方法综述可以看[官方文档](https://grpc.io/docs/guides/auth/)。

添加以上信息后，将我们写好的服务类注册到builder，然后再用BuildAndStart即可以得到服务器并启动。代码如下：

```c++
int main(int argc, char** argv){
	std::string addr = "127.0.0.1:47852";
	SimpleMathImpl simple_math_service;
	ServerBuilder builder;
	builder.AddListeningPort(addr,grpc::InsecureServerCredentials());
	builder.RegisterService(&simple_math_service);
	std::unique_ptr<Server> simple_math_server = builder.BuildAndStart();
	std::cout << "Simple Math Server listening on" << addr << std::endl;
	simple_math_server->Wait();
	//服务器将一直等待连接，不会执行到return
	return 0;
}
```



#### 客户端调用函数

客户端需要通过“桩”(stub)来调用函数，对桩的理解可以认为是一个接口，即在客户端这里，桩代表了整个服务器的抽象，客户端可以通过桩调用背后所提供的服务，而不必了解具体的服务实现，也不必处理任何的通讯需求。

我们首先新建一个桩，它的初始化需要一个channel类型，可以通过`grpc::CreateChannel`这个函数来来建立。而该函数`grpc::CreateChannel`需要两个参数，一个是服务器的地址，这里我们知道服务器是在本地地址上监控47852这个端口，所以直接给出地址`127.0.0.1:47852`。另一个是一个认证信息，和服务器一样，我们不需要进行认证，因此继续用未加密认证。

```c++
int main(int argc, char** argv){
	std::string addr = "127.0.0.1:47852";
	std::unique_ptr<SimpleMath::Stub> stub;
	//建立stub
	stub = SimpleMath::NewStub(grpc::CreateChannel(addr, grpc::InsecureChannelCredentials()));
	Numbers nums;
	Status status;  //用来接受server返回的状态，如果状态为失败说明调用失败
	Result res;  // 准备好储存server回复的placeholder
    //设置好需要发送的request的参数
	nums.set_a(1);
	nums.set_b(2);
	nums.set_name("gRPC教程by Y.Zheng");
	//通过stub调用server端的函数
	ClientContext context;
	status = stub->intSum(&context, nums, &res);
	//解析结果
	if (status.ok()){
		std::cout<<"Computing Result is "<<res.res()<<std::endl;
	}
	else{
		std::cout<<"Error in calling remote procedure"<<std::endl;
	}
}
```

 虽然这是最简单的客户端调用方法，但是如果用类进行了再次包装，可以更方便地管理函数和进行调用，架构如下：

```c++
class SimpleMathClient{
public:
	SimpleMathClient(std::string addr, std::string name="gRPC教程by Y.Zheng"):
			stub(SimpleMath::NewStub(grpc::CreateChannel(addr, grpc::InsecureChannelCredentials()))),
			name(name)
			{}; // constructor，设定stub及客户名字
	std::unique_ptr<SimpleMath::Stub> stub; //客户端的“桩”，可以理解为客户端上用来链接所有服务器函数的一个接口
	std::string name; //客户端的名字，只是为了演示作用，可以没有
	void sum_int(int a, int b){
        ...
};
```

在Github客户端[代码](https://github.com/ywzheng07/gRPCSimpleMath/blob/master/math_client.cpp)里，有以上实现的完整版。

#### 编译和运行

代码的编译即分别将gRPC提供的源码和我们自己的源码编译为二进制文件(*.o)，然后分别编译客户端和服务端的可执行文件。官方的C++教程中提供了Makefile，考虑到可能有些读者不太熟悉Makefile中的很多语法，这里提供直接用命令行编译的方式。

生成二进制目标文件：

```bash
g++ `pkg-config --cflags protobuf grpc` -c -o simple_math.pb.o simple_math.pb.cc
g++ `pkg-config --cflags protobuf grpc` -c -o simple_math.grpc.pb.o simple_math.grpc.pb.cc
g++ `pkg-config --cflags protobuf grpc` -c -o math_client.o math_client.cpp
g++ `pkg-config --cflags protobuf grpc` -c -o math_server.o math_server.cpp
```

生成客户端执行文件：

```bash
g++ simple_math.pb.o simple_math.grpc.pb.o math_client.o `pkg-config --libs protobuf grpc++` -o mathclient
```

生成服务器执行文件：

```bash
g++ simple_math.pb.o simple_math.grpc.pb.o math_server.o `pkg-config --libs protobuf grpc++` -o mathserver
```

接下来就可以运行了，在一个终端输入：

```bash
./mathserver
# 应该看到以下内容：
Simple Math Server listening on 127.0.0.1:47852
```

然后打开另一个终端输入：

```bash
./mathclient
# 用户可以自行输入两个整数：
Please enter two integers, a: 42
and b: 1984
Computing Result is 2026
```

客户端启动后，服务器端应该输出以下内容，并继续等待连接：

```bash
Processing sum request from client: gRPC教程by Y.Zheng
Result is: 2026
```

为了方便大家编译，我们也将上述命令行放入了[Makefile](https://github.com/ywzheng07/gRPCSimpleMath/blob/master/Makefile)，注意编译的时候需要先`make proto`再`make`。

### 总结

本文主要简单介绍了gRPC是什么，为什么要用以及简单的怎么用。当然，这个介绍只是冰山一角，真正的gRPC使用有更多的技巧，需要大家在实践中学习。后续预计有关gRPC还会有两篇文章左右，一篇利用官方的C++教程解释stream data相关的内容。另一篇将包括一些简单的小窍门，例如定义空的message，和message之中repeated参数用法。
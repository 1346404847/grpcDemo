#### grpc 简介
##### grpc 是什么
+ grpc 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计。目前提供 C、Java 和 Go 语言版本，分别是：grpc, grpc-java, grpc-go. 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持。

+ grpc 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特。这些特性使得其在移动设备上表现更好，更省电和节省空间占用

#### 环境安装
##### 首先安装 protobuf：
+ 基于MAC环境，打开终端，执行如下命令：  
```
brew install protobuf
```
+ 查看是否安装成功：  
```
protoc --version
```

##### 再编译安装 grpc-java 插件：
+ 使用git下载源码：

```
git clone https://github.com/grpc/grpc-java.git

```
+ 进入源码 compiler 目录：

```
cd compiler

```

+ 依次执行命令：

```
    ../gradlew java_pluginExecutable
    ../gradlew test
    ../gradlew install
```
可能需要翻墙，并执行成功为止。


#### 使用示例
##### Maven 依赖

+ 在服务端和客户端的 pom.xml 中添加相关依赖：

```
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-netty</artifactId>
  <version>1.12.0</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-protobuf</artifactId>
  <version>1.12.0</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-core</artifactId>
  <version>1.12.0</version>
</dependency>
<dependency>
  <groupId>io.grpc</groupId>
  <artifactId>grpc-stub</artifactId>
  <version>1.12.0</version>
</dependency>

```

#### 编写服务端 proto 文件
+ hello.proto
```
// 语法类型
syntax = "proto3";
//指定在proto文件中定义的所有消息、枚举和服务在生成java类的时候都会生成对应的java类文件，而不是以内部类的形式出现。
option java_multiple_files = false;
// 指定包名
option java_package = "com.example.client.grpc";
// 指定类名
option java_outer_classname = "HelloProto";

//package hello;

service Greeter {
// 处理请求
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 定义通用的 Grpc 请求体
message HelloRequest {
  string name = 1;
}

// 定义通用的 Grpc 响应体
message HelloReply {
  string message = 1;
}
```

#### 使用 proto 工具生成 java 代码
```
protoc --java_out=. hello.proto

```
#### 使用 grpc-java 工具生成 java 代码

注意插件地址为以上编译生成的实际地址
```
protoc --plugin=protoc-gen-grpc-java=/Users/alina/project/grpc-java/compiler/build/exe/java_plugin/protoc-gen-grpc-java --grpc-java_out=. --proto_path=. hello.proto
```

#### 编写服务端 proto 文件

```
修改 option java_package = "com.example.server.grpc"; 后再次执行以上命令则生成服务端端 java 代码
```

#### 将代码放到对应目录
即：com.example.server 子目录 grpc 下

#### 编写服务端

```
package com.example.server.grpc.server;

import com.example.server.grpc.GreeterGrpc;
import com.example.server.grpc.HelloProto.HelloReply;
import com.example.server.grpc.HelloProto.HelloRequest;
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;

import java.io.IOException;
import java.util.logging.Logger;

public class HelloServer {
    private static final Logger logger = Logger.getLogger(HelloServer.class.getName());
    private int port = 50051;
    private Server server;

    private void start() throws IOException {
        // 使用ServerBuilder来构建和启动服务，通过使用forPort方法来指定监听的地址和端口
        // 创建一个实现方法的服务GreeterImpl的实例，并通过addService方法将该实例纳入
        // 调用build() start()方法构建和启动rpcserver
        server = ServerBuilder.forPort(port)
                .addService(new GreeterImpl())
                .build()
                .start();
        logger.info("Server started, listening on " + port);

        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                // Use stderr here since the logger may have been reset by its JVM shutdown hook.
                System.err.println("*** shutting down gRPC server since JVM is shutting down");
                HelloServer.this.stop();
                System.err.println("*** server shut down");
            }
        });
    }

    private void stop() {
        if (server != null) {
            server.shutdown();
        }
    }

    /**
     * Await termination on the main thread since the grpc library uses daemon threads.
     */
    private void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }

    /**
     * Main launches the server from the command line.
     */
    public static void main(String[] args) throws IOException, InterruptedException {
        final HelloServer server = new HelloServer();
        server.start();
        server.blockUntilShutdown();
    }

    // 我们的服务GreeterImpl继承了生成抽象类GreeterGrpc.GreeterImplBase，实现了服务的所有方法
    private class GreeterImpl extends GreeterGrpc.GreeterImplBase {

        @Override
        public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
            HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
            // 使用响应监视器的onNext方法返回HelloReply
            responseObserver.onNext(reply);
            // 使用onCompleted方法指定本次调用已经完成
            responseObserver.onCompleted();
        }
    }
}
```

#### 编写客户端 
```
编写客户端
package com.example.client.grpc.client;

import com.example.client.grpc.GreeterGrpc;
import com.example.client.grpc.HelloProto.HelloReply;
import com.example.client.grpc.HelloProto.HelloRequest;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.StatusRuntimeException;

import java.util.concurrent.TimeUnit;
import java.util.logging.Level;
import java.util.logging.Logger;

public class HelloClient {
    private static final Logger logger = Logger.getLogger(HelloClient.class.getName());

    private final ManagedChannel channel;
    private final GreeterGrpc.GreeterBlockingStub blockingStub;

    /** Construct client connecting to HelloWorld server at {@code host:port}. */
    // 首先，我们需要为stub创建一个grpc的channel，指定我们连接服务端的地址和端口
    // 使用ManagedChannelBuilder方法来创建channel
    public HelloClient(String host, int port) {
        channel = ManagedChannelBuilder.forAddress(host, port)
                // Channels are secure by default (via SSL/TLS). For the example we disable TLS to avoid
                // needing certificates.
                .usePlaintext(true)
                .build();
        // 使用我们从proto文件生成的GreeterGrpc类提供的newBlockingStub方法指定channel创建stubs
        blockingStub = GreeterGrpc.newBlockingStub(channel);
    }

    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }
    // 调用服务端方法
    /** Say hello to server. */
    public void greet(String name) {
        logger.info("Will try to greet " + name + " ...");
        // 创建并定制protocol buffer对象，使用该对象调用服务端的sayHello方法，获得response
        HelloRequest request = HelloRequest.newBuilder().setName(name).build();
        HelloReply response;
        try {
            response = blockingStub.sayHello(request);
            // 如果有异常发生，则异常被编码成Status，可以从StatusRuntimeException异常中捕获
        } catch (StatusRuntimeException e) {
            logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
            return;
        }
        logger.info("Greeting: " + response.getMessage());
    }

    /**
     * Greet server. If provided, the first element of {@code args} is the name to use in the
     * greeting.
     */
    public static void main(String[] args) throws Exception {
        HelloClient client = new HelloClient("localhost", 50051);
        try {
            /* Access a service running on the local machine on port 50051 */
            String user = "hans";
            if (args.length > 0) {
                user = args[0]; /* Use the arg as the name to greet if provided */
            }
            client.greet(user);
        } finally {
            client.shutdown();
        }
    }
}
```

#### 启动服务端
+ 后台打印输出：

```
Server started, listening on 50051

```

#### 执行客户端

+ 后台打印输出：

```
Will try to greet hans ...
Greeting: Hello hans
```

#### Github 完整代码
```
https://github.com/hxun123/spring-boot-demo/tree/master/spring-boot-grpc
```
转载：https://segmentfault.com/a/1190000015042409

官网案例：https://github.com/grpc/grpc-java

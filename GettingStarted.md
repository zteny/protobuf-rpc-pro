# Getting Started #

The [example](http://protobuf-rpc-pro.googlecode.com/svn/trunk/protobuf-rpc-pro-demo/src/main/java/com/googlecode/protobuf/pro/duplex/example/) source package contains several runnable examples.

The examples use a simple [PingPong](http://protobuf-rpc-pro.googlecode.com/svn/trunk/protobuf-rpc-pro-duplex/src/test/protos/pingpong.proto) service where a client can call "ping" on a server.

## Client Code ##

Firstly we declare who the client is and who the server is that we're going to connect to. Note that the client does not actually bind to port 1234, it is just used as a "name".
```
	PeerInfo client = new PeerInfo("clientHostname", 1234);
	PeerInfo server = new PeerInfo("serverHostname", 8080);
```
The main client class to start with is a DuplexTcpClientPipelineFactory  which works together with Netty Bootsrap to construct client channels.
```
	DuplexTcpClientPipelineFactory clientFactory = new DuplexTcpClientPipelineFactory(client);
```
If a client is also going to be acting as a server, it is necessary to setup an RpcCallExecutor who's purpose it is to run the calls ( using threads separate from the IO Threads ).
```
	RpcServerCallExecutor executor = new ThreadPoolCallExecutor(3, 100);
	clientFactory.setRpcServerCallExecutor(executor);
```
In order to customize TCP settings, you can use all Netty socket options and the "connectResponseTimeoutMillis" which is introduced to put an upper bound on the "peering" time.
```
	clientFactory.setConnectResponseTimeoutMillis(10000);
```
In order to compress all data traffic to and from the server, you can switch on compression.
```
    	clientFactory.setCompression(true);
```
Refer to this [tuning-guide](http://fasterdata.es.net/TCP-tuning/) for buffer size tuning help. [Nagle's Algorithm](http://en.wikipedia.org/wiki/Nagle's_algorithm) can be disabled by setting tcpNoDelay to true.
```
	Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(new NioEventLoopGroup());
        bootstrap.handler(clientFactory);
        bootstrap.channel(NioSocketChannel.class);
        bootstrap.option(ChannelOption.TCP_NODELAY, true);
    	bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS,10000);
        bootstrap.option(ChannelOption.SO_SNDBUF, 1048576);
        bootstrap.option(ChannelOption.SO_RCVBUF, 1048576);
```
In order to open a TCP connection to the server it is necessary to "peerWith" it. A server will not allow the same client "named" to connect multiple times. ( You can still make more than one connection to the same server from the same "Process", just choose different ports to name them and separate Bootstraps ).
```
    	RpcClientChannel channel = clientFactory.peerWith(server, bootstrap);
```
Then you can use the pretty much standard Protocol Buffer services which you have like this.
```
	BlockingInterface pingpongService = PingPongService.newBlockingStub(channel);
	RpcController controller = channel.newRpcController();
			
	Ping request = Ping.newBuilder().set....build();
	Pong pong = pingpongService.ping(controller, request);
```
The same RpcClientChannel can be used multiple times for calls to the server, using any Service which the server handles.

In order to service RPC calls on the client side, you just need to register a service implementation with the bootstrap.
```
    	clientFactory.getRpcServiceRegistry().registerService(new PingPongServiceImpl());
```
Service implementations can be added and removed at runtime. Service methods are looked up by "shortname" so the server and client "packaging" need not be identical.

Finally to close the RpcClientChannel so it cannot be used anymore do, call close. On shutdown of the client application you need to call release resources to stop the low-level IO-Threads.
```
	channel.close();
```
You can register all Bootstraps and DuplexTcpClientPipelineFactory with an instance of the CleanShutdownHandler utility class to perform a clean shutdown on exit.

## Server Code ##

The server side is pretty similar to the client above. The server needs to know "who" it is, and be given a port on which to bind to on the machine it is running on. Note you can configure a local address to bind onto also ( multi-homing support ) through the Netty localAddress option. The server's hostname should normally be the server's hostname which resolves in DNS to the server machine, however it is just a name ( like the client's port is just a name ).
```
    	PeerInfo serverInfo = new PeerInfo("serverHostname", 8080);
```
You need then to create a DuplexTcpServerBootstrap and provide it an RpcCallExecutor.
```
    	RpcServerCallExecutor executor = new ThreadPoolCallExecutor(3, 200);
    	
    	DuplexTcpServerPipelineFactory serverFactory = new DuplexTcpServerPipelineFactory(serverInfo);
    	serverFactory.setRpcServerCallExecutor(executor);
```
Now the DuplexTcpServerPipelineFactory needs to be registered as a child ChannelInitializer handler of the Netty ServerBootstrap.
```
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(new NioEventLoopGroup(0,new RenamingThreadFactoryProxy("boss", Executors.defaultThreadFactory())),
        		new NioEventLoopGroup(0,new RenamingThreadFactoryProxy("worker", Executors.defaultThreadFactory()))
        		);
        bootstrap.channel(NioServerSocketChannel.class);
        bootstrap.childHandler(serverFactory);
        bootstrap.localAddress(serverInfo.getPort());
```
The Netty ServerBootstrap can be used to set all TCP/IP settings.
```
        bootstrap.option(ChannelOption.SO_SNDBUF, 1048576);
        bootstrap.option(ChannelOption.SO_RCVBUF, 1048576);
        bootstrap.childOption(ChannelOption.SO_RCVBUF, 1048576);
        bootstrap.childOption(ChannelOption.SO_SNDBUF, 1048576);
        bootstrap.option(ChannelOption.TCP_NODELAY, true);
```
Then you need to register your server side services with the bootstrap. Again here the registration is dynamic and can change at runtime.
```
    	serverFactory.getRpcServiceRegistry().registerService(new DefaultPingPongServiceImpl());
```
Finally binding the bootstrap to the TCP port will start off the socket accepting and clients can start to connect.
```
        bootstrap.bind();
```
If you want to track the RPC peering events with clients, use a RpcClientConnectionRegistry or a TcpConnectionEventListener for TCP connection events. This is the mechanism you can use to "discover" RPC clients before they "call" any service.
```
    	RpcClientConnectionRegistry clientRegistry = new RpcClientConnectionRegistry();
    	serverFactory.registerConnectionEventListener(clientRegistry);
```
You can then also close the server by closing the channel which the bootstrap is bound to and finally releaseExternalResources on final shutdown. Also here you can use the CleanShutdownHandler to perform a clean close on exit.


## Reverse RPC ##

The client and server examples above show how a client can call a RPC service registered at the serverFactory. In order to enable a server to call a client, it is necessary first for there to be a RPC service registered at the client.
```
    	clientFactory.getRpcServiceRegistry().registerService(new DefaultPingPongServiceImpl());
```
Both client and server bootstraps have a RpcServiceRegistry. Secondly the server code needs to get hold of a RpcClientChannel to communicate back to the client. This is possible through the RpcController on the server side which is available during server call processing, or through the server's RpcClientRegistry at any time.





## Runtime Dependencies ##
The external dependencies have been kept to a minimum. Netty, slf4j and protobuf-java are the only compile time dependencies. Compiled against java 1.7. The dependencies can be seen in the project's maven pom.xml.

```
		<dependency>
			<groupId>com.google.protobuf</groupId>
			<artifactId>protobuf-java</artifactId>
			<version>2.6.1</version>
		</dependency>
		<dependency>
			<groupId>io.netty</groupId>
			<artifactId>netty</artifactId>
			<version>4.0.23.Final</version>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-api</artifactId>
			<version>1.7.2</version>
		</dependency>
```
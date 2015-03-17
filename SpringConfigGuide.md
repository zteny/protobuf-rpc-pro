Thanks to Dieter Mayrhofer for contributing this example code. It shows one way to configure a server side implementation using spring. The code and runner is checked into the ( [demo package](http://protobuf-rpc-pro.googlecode.com/svn/trunk/protobuf-rpc-pro-demo/src/main/java/com/googlecode/protobuf/pro/duplex/example/spring) ). The example starts a DefaultPingPongServerImpl service which is provided as standalone example.

# Dependency on Spring #

Introduce the following maven dependencies to spring.
```
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
			<version>3.0.5.RELEASE</version>
			<type>jar</type>
			<scope>compile</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>3.0.5.RELEASE</version>
			<type>jar</type>
			<scope>compile</scope>
		</dependency>
```


# Server Spring Component #

The main spring component uses PostConstruct and PreDestroy annotations to startup and tear down the component cleanly.

```
package com.googlecode.protobuf.pro.duplex.example.spring;
...
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
...
import org.springframework.beans.factory.annotation.Autowired;
...
public class PingSpringServer {
	
	@Autowired(required = true)
	private DefaultPingPongServiceImpl pingPongServiceImpl;

	int port;
	String host;

	protected final Log log = LogFactory.getLog(getClass());

	private DuplexTcpServerBootstrap bootstrap;

	public PingSpringServer(String host, int port) {
		this.host = host;
		this.port = port;
	}

	@PostConstruct
	public void init() {
		runServer();
	}

	public void runServer() {
		PeerInfo serverInfo = new PeerInfo(host, port);
		RpcServerCallExecutor executor = new ThreadPoolCallExecutor(10, 10);

		bootstrap = new DuplexTcpServerBootstrap(serverInfo,
				new NioServerSocketChannelFactory(
						Executors.newCachedThreadPool(),
						Executors.newCachedThreadPool()), executor);
		log.info("Proto Serverbootstrap created");

		// Register Ping Service
		Service pingService = PingService.newReflectiveService(pingPongServiceImpl);

		bootstrap.getRpcServiceRegistry().registerService(pingService);
		log.info("Proto Ping Registerservice executed");

		bootstrap.bind();
		log.info("Proto Ping Server Bound to port " + port);
	}

	@PreDestroy
	protected void unbind() throws Throwable {
		super.finalize();
		bootstrap.releaseExternalResources();
		log.info("Proto Ping Server Unbound");
	}
}
```

# Spring Configuration #

The following configuration is used to setup the server component on localhost, port 8090. Note: other examples use port 8080.

```
public class SpringConfig {
	private static String PROTOSERVERHOST = "localhost";
	private static int PROTOSERVERPORT = 8090;

	// Implementation of the Service Interface
	@Bean(name = "pingPongServiceImpl")
	public DefaultPingPongServiceImpl pingPongServiceImpl() {
		return new DefaultPingPongServiceImpl();
	}

	// Will start the server
	@Bean(name = "pingSpringServer")
	public PingSpringServer pingSpringServer() {
		return new PingSpringServer(PROTOSERVERHOST, PROTOSERVERPORT);
	}
}
```


# Running in Standalone JVM #

The following code starts the entire spring application and stops it after 10s.

```
	public static void main(String[] args) throws Exception {
		ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringConfig.class);

		Thread.sleep(10000);
		
		System.exit(0);
	}
```
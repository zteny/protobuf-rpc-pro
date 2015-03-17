This project provides an java implementation for Google's Protocol Buffer RPC services. The implementation builds upon Netty for low-level NIO.

## Features ##

  * TCP connection keep-alive.
  * Bi-directional RPC calls from client to server and server to client.
  * SSL socket layer encryption option.
  * Data Compression option.
  * RPC call cancellation.
  * RPC call timeout.
  * Out-of-Band RPC server replies to client.
  * Non RPC Protocol Buffer messaging for peer to peer communication.
  * Protocol Buffer wire protocol.
  * Semantics for calls handling on TCP connection closure.
  * Pluggable logging facility.

For more details see the wiki page http://code.google.com/p/protobuf-rpc-pro/wiki/Features.

## Maven Dependency ##

protobuf-rpc-pro is available via the maven central repository http://repo1.maven.org/maven2. The demo examples are available under the artifactId "protocol-rpc-pro-demo".

```
		<dependency>
			<groupId>com.googlecode.protobuf-rpc-pro</groupId>
			<artifactId>protobuf-rpc-pro-duplex</artifactId>
			<version>3.3</version>
			<type>jar</type>
		</dependency>
```

## Statistics ##

&lt;wiki:gadget url="http://www.ohloh.net/p/protobuf-rpc-pro/widgets/project\_basic\_stats.xml" height="220" border="1" /&gt;

# Introduction #

This guide explains how to setup keystores and certificate truststores in order to secure TCP communications between RPC client and server. The code required to secure communications is trivial and shown below for both client and server. The effort or price for the additional security is the effort and maintenance involved in managing the "trusted material" ie. keys, certificates and keystores.

## Client Code ##

The client must initialize a RpcSSLContext and register this with the bootstrap prior to peering with the server. The RpcSSLContext must be pointed to the client's keystore and it's truststore.
```
    	RpcSSLContext sslCtx = new RpcSSLContext();
    	sslCtx.setKeystorePassword("changeme");
    	sslCtx.setKeystorePath("../lib/client.keystore");
    	sslCtx.setTruststorePassword("changeme");
    	sslCtx.setTruststorePath("../lib/truststore");
    	sslCtx.init();
    	
	DuplexTcpClientBootstrap bootstrap = new DuplexTcpClientBootstrap(...);

	bootstrap.setSslContext(sslCtx);
```

## Server Code ##

On the server, similarly to the client, a RpcSSLContext needs initializing, where it points to the server's keystore and it's truststore.
```
    	RpcSSLContext sslCtx = new RpcSSLContext();
    	sslCtx.setKeystorePassword("changeme");
    	sslCtx.setKeystorePath("../lib/server.keystore");
    	sslCtx.setTruststorePassword("changeme");
    	sslCtx.setTruststorePath("../lib/truststore");
    	sslCtx.init();
    	
        DuplexTcpServerBootstrap bootstrap = new DuplexTcpServerBootstrap(...);
        
        bootstrap.setSslContext(sslCtx);
```

Only connections bound after setSslContext has been done are secured.

## Security Architecture ##

As soon as a RPC client and server connect via TCP, the SSL handshake takes place. The SSL engines  of the participants are configured for each to want to check the other's certificate. So we get mutual authentication.

Each client is issued with a certificate which is issued by the server. We put the server's certificate in the client's truststore. The client will check the server's certificate against it's truststore during the SSL handshake. This means the client will trust ( and allow secure connection ) to only the server ( or servers setup with the same keys and certificates ).

The server will request and check the client's certificate. The client supplies it's certificate which is in the client keystore during SSL handshake. The server checks the client's certificates against the server's truststore. The server's certificate is in it's own truststore, and the server certificate signed the client's so it is trusted.

Different clients can be given different certificates signed by the server, they will all be trusted. The clients do not need to share the same certificate.

In summary the setup is as follows

  * client keystore: private key + public certificate of client(signed by the server).
  * client truststore: public certificate of server(signed by certification authority) + certificationauthority certificate

  * server keystore: private key + public certificate of server
  * server truststore: public certificate of server(signed by certification authority) + certificationauthority certificate





## Prerequisites ##

In order to generate and sign certificates, OpenSSL should be installed and usable. ( On windows, you can use Cygwin's openssl ). We also generate and use a self-signed root Certificate-Authority which we use as the root truster of the certificates which we generate. You can create such a certificate by following these steps
```
1] generate a 2048 bit RSA key which is encrypted using Triple-DES and stored in a 
PEM format so that it is readable as ASCII text.

$ openssl genrsa -des3 -out certificateauthority.key 2048
providing a passphrase ( changeme )

2] self sign it with a long validity

$ openssl req -new -x509 -days 1000 -key certificateauthority.key -out certificateauthority.crt
...Common Name (eg, YOUR name) []:ca
Email Address []:ca@trustworthy-inc.com

3] remove passhprase from private key

$ cp certificateauthority.key certificateauthority.key.original
$ openssl rsa -in certificateauthority.key.original -out certificateauthority.key

```


# Setup #

## Server Keystore ##

Follow the steps here to create a server keypair, and sign the server public certificate with our trusted CA.
```
1] need to create a server keypair

$ openssl genrsa -des3 -out server.key 2048
Generating RSA private key, 2048 bit long modulus
...Enter pass phrase for server.key:(changeme)

2] generate a certificate signing request CSR

$ openssl req -new -key server.key -out server.csr 
Enter pass phrase for server.key:
...Common Name (eg, YOUR name) []:server
Email Address []:server@myorganization.com

3] remove passhprase from private key

$ cp server.key server.key.original
$ openssl rsa -in server.key.original -out server.key

4] sign the CSR with our CA cert.

$ openssl x509 -req -days 365 -in server.csr -CAkey certificateauthority.key -CA certificateauthority.crt -set_serial 01 -out server.crt

5] transform private key to DER for for inporting into a JKS keystore

$ openssl pkcs8 -topk8 -nocrypt -in server.key -out server.key.der -outform der

6] transform public key to DER format for inporting into the JKS keystore

$ openssl x509 -in server.crt -out server.crt.der -outform der

7] add both the public and private keys into the server.keystore

$ ./RunKeyTool.cmd server.key.der server.crt.der server.keystore changeme server-key

8] verify the result

$ "$JAVA_HOME/bin/keytool" -list -keystore server.keystore
Enter keystore password:  changeme

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 1 entry

server-key, Aug 23, 2010, PrivateKeyEntry, 
Certificate fingerprint (MD5): 3C:E9:38:06:23:DC:3D:0B:EA:F1:B3:4C:EC:BC:6D:5F

```

## Client Keystore ##

We create now a client keypair which is signed by our server certificate. This means that the client certificate is trusted by the server certificate.

```

1] generate a client private key ( giving passphrase 'changeme' )

$ openssl genrsa -des3 -out client.key 2048

2] create a CSR for the client

$ openssl req -new -key client.key -out client.csr
...Common Name (eg, YOUR name) []:client
Email Address []:client@myorganization.com

3] remove passhprase from private key

$ cp client.key client.key.original
$ openssl rsa -in client.key.original -out client.key

4] sign client certificate with our SERVER cert.

$ openssl x509 -req -days 364 -in client.csr -CAkey server.key -CA server.crt -set_serial 01 -out client.crt

5] convert client key to DER form

$ openssl pkcs8 -topk8 -nocrypt -in client.key -out client.key.der -outform der

6] convert client public cert to DER form

$ openssl x509 -in client.crt -out client.crt.der -outform der

7] add both private and public client keys into the JKS client keystore

$ ./RunKeyTool.cmd client.key.der client.crt.der client.keystore changeme client-key

d:\Development\Workspace\protobuf-rpc-pro\protobuf-rpc-pro-duplex\ssl>java -classpath ..\target\protobuf-rpc-pro-duplex-1.1.0.jar com.googlecode.protobuf.pro.duplex.util.KeyStoreImportUtil client.key.der client.crt.der client.keystore pwd client-key 
Using keystore-file : client.keystore
One certificate, no chain.
Key and certificate stored.
Alias:client-key  Password:changeme

```

## Truststore ##

We need to put our CA and server certificates into a truststore.

```
1] create a truststore with the RootCA in it.

$ "$JAVA_HOME/bin/keytool" -import -alias rootca -file certificateauthority.crt -trustcacerts -keystore truststore -storepass changeme 
....
Trust this certificate? [no]:  yes
Certificate was added to keystore

2] add the server certificate to the truststore.

$ "$JAVA_HOME/bin/keytool" -import -alias server -file server.crt -trustcacerts -keystore truststore -storepass changeme 
Certificate was added to keystore

3] verify both are in the truststore

$ "$JAVA_HOME/bin/keytool" -list -keystore truststore            
Enter keystore password:  changeme

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 2 entries

rootca, Aug 23, 2010, trustedCertEntry,
Certificate fingerprint (MD5): 00:13:D2:E5:7B:9F:63:A5:88:F0:AE:69:94:44:4F:33
server, Aug 23, 2010, trustedCertEntry,
Certificate fingerprint (MD5): 75:EC:46:5D:52:AD:C0:D4:9A:82:0F:1E:E3:3F:AE:01

```
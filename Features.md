## TCP connection re-use ##

RPC calls from an RPC client can be multiplexed over a single TCP socket to an RPC server. The RPC client has full control of the TCP socket, which is kept open between calls. This avoids the extra overhead of TCP connection establishment for each call ( SYN, SSL handshake ).

## Bi-directional RPC calls client to server and server to client ##

It features a bidirectional architecture, in which a client can itself be a server and vice versa. In security conscious "zoned" environments, it is typical that TCP connectivity between zones is restricted to one direction ( normally client to server ). Firewall configurations are always a pain to setup and change, so being able to communicate as equal "peers" in either direction using Protocol Buffer services is handy. In particular when multiplexing calls over the same TCP connection. This is interesting because if the connection breaks, both client and server (eventually) realize that their peering is broken and both can have a common understanding of their respective states. It is even possible for a server to perform a blocking RPC call to the client who's blocking call it is currently servicing. It is possible for a client to open a connection to a server and then the server starts acting as client to the client which then acts as server, without the client ever having called a server call. This is enabled due to notifications issued on connection establishment, loss,  and reestablishment. Apart from the initiation of the TCP connection to a remote server, the RPC client and server are equivalent "Peers".

## SSL socket layer encryption option ##

Encrypting all data traffic between client and server is as easy as configuring a certificate truststore and keystore on both client and server. Both client and server authentication is used.

## Data Compression ##

Clients can choose whether data communications between client and server are compressed or not. The server can accept both compressing and un-compressing clients simultaneously. Compression sacrifices CPU resources ( and increases response times ) for IO usage reduction.

## RPC call cancellation ##

Fairly tricky to get right, but RPC call cancellation is supported with full semantics on both client and server sides. In process client calls are immediately failed, irrespective of whether the server side call processing is cancelled. The server side call will be cancelled on receipt of the client cancellation. This involves interrupting any thread which is currently processing the call. The processing thread can stop processing when the server controller indicates that cancellation has taken place. The RPC implementation will not return any reply message to the client after cancellation has been received on the server side. The RPC layer performs the appropriate cancel notification to the callbacks registered by the service implementation.

## RPC call timeout ##

Although not specified in the protobuf RPC API, we support setting a timeout for RPC calls at the client. The semantics of RPC call timeout are almost identical to RPC call cancellation and more information is provided in the wiki.

## Out-of-Band RPC server replies to client ##

Enables parts of responses to be transferred from server to client before a call ends. For example a percent complete indicator for long running calls, or chunks of later data transfers.

## Non RPC / One-way Protocol Buffer messaging ##

General purpose one way messaging between peers is enabled with "Out-of-Band" messages which are not correlated to any RPC calls. See "StatusServer" and "StatusClient" demo.

## Semantics for calls handling on TCP connection closure ##

When an underlying TCP socket "breaks" or is closed by either client or server, all pending client calls are immediately failed towards the client. On the server side, pending calls are immediately cancelled.

## Pluggable logging facility ##

A pluggable logging facility is included to log RPC calls and their outcomes in Protocol Buffer format.

## Protocol Buffer wire protocol ##

The wire-protocol is Protocol Buffer defined, so as to be potentially cross-platform.
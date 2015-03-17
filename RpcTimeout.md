# RPC timeout #

The RPC timeout feature allows a client to specify a time in milliseconds for the maximum allowed duration of a RPC call ( irrespective of whether the call is called using a blocking or non blocking method ).
```
	final ClientRpcController controller = channel.newRpcController();
	controller.setTimeoutMs(1000);
```

To enable monitoring of RpcServer side timeouts and RpcClient non blocking timeouts, it is necessary to register a RpcTimeoutChecker with the respective Bootstrap. The frequency of timeout checking and sizing of thread pool executors is configurable. For example:
```
        RpcTimeoutExecutor timeoutExecutor = new TimeoutExecutor(1,5);
	RpcTimeoutChecker checker = new TimeoutChecker();
	checker.setTimeoutExecutor(timeoutExecutor);
	checker.startChecking(bootstrap.getRpcClientRegistry());

```

Since the original Google RPC API does not specify any semantics for RPC timeout, it is prudent to define here exactly the behaviour of the library.

## RpcClient ##
The client can determine the RPC timeout value per call, by using the ClientRpcControllers.setTimeoutMs() method. If a controller is "reset", then the timeout values is also cleared ( setting to 0 means infinite timeout ).

If a timeout takes place on the client, the RPC result is always a failed result with errorReason("Timeout");

The RPC timeout management is different depending on whether a client is using blocking or non blocking calls.

> For blocking calls, the calling thread blocks for at most the timeout value and will timeout with a high accuracy, therefore the blocking call timeout values can be made arbitrarily small. On timeout the RPC method ends with ServiceException with reason "Timeout";

For non blocking calls, the RPC timeout is monitored by a RpcTimeoutChecker thread, which periodially scans to see if there are non blocking client calls which have not received a response yet but have exceeded their timeout. The RpcTimeoutChecker then uses a RpcTimeoutExecutor thread pool to "timeout" the non blocking call. The RpcCallback handling is identical to the "cancel" handling ( callback with null ) and the errorText is "Timeout".

For both blocking and non blocking client calls, the RPC timeout value is transmitted from client to server. There is no explicit communication with the server once a timeout occurs on the client. The server handles a call timeout as if the client would send a cancellation ( but without the client sending anything ). How the server side handles the call timeout is described below in the RpcServer section. There is no communication back to the client if the server accertains that the call has taken longer than the timeout duration at the server.

Due to the symetry of the duplex client server, the RpcServer timeout handling on the TCP client side of the connection and the RpcClient side timeout handling of the TCP server side are identical to the handling on the opposite TCP sides.

## RpcServer ##
your application code registers service implementations at the ServerBootstrap's RpcServiceRegistry. It is possible to define how the server will handle timeouts and cancellations per Service implementation. When registering a service you stipulate with "allowTimeout" ( default true ) what to do when the server's processing time of a client call exceeds the timeout.
> true  - the server call is "cancelled" ( equivalent of a client issueing an explicit cancel ). The running thread is "interrupted" which might cause transactional rollback.

> false - the server call is not cancelled and allowed to finish.
If an RPC server call has timed out before starting processing due to buffering in the RpcServerCallExecutor, the service implementation will not be called, and the received data silently discarded.
When a RPC server call completes and the timeout of the call has exceeded the client's timeout, NO response will be transferred to the client.	This mandates that the client sets up an RpcTimeoutChecker on the client side for both blocking and nonblocking calls.

The mechanism that timeouts are monitored on the server side does not depend on whether blocking server calls are implemented or nonblocking calls( reflexive service vs blocking service ), in the same way that client cancellation is handled the same way in both.

The server's RpcTimeoutChecker will periodically scan all on-going server calls and use it's RpcTimeoutExecutor to initiate timeout of individual calls ( subject to the service's timeout policy ). This periodic sweeping mechanism is designed to be less accurate but providing more efficiency given that the server side number of calls can be very high, and it is expected that timeouts are rare. Also, if timeouts are very small, the overhead of "cancellation" theoretically outweighs the time which would take to complete but just not send the reply to the client.
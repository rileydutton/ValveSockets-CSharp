<p align="center"> 
  <img src="https://i.imgur.com/CS43fv4.png" alt="alt logo">
</p>

[![PayPal](https://drive.google.com/uc?id=1OQrtNBVJehNVxgPf6T6yX1wIysz1ElLR)](https://www.paypal.me/nxrighthere) [![Bountysource](https://drive.google.com/uc?id=19QRobscL8Ir2RL489IbVjcw3fULfWS_Q)](https://salt.bountysource.com/checkout/amount?team=nxrighthere)

This repository provides a managed C# wrapper for [GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets) library which is created and maintained by [Valve Software](https://www.valvesoftware.com). You will need to [build](https://github.com/ValveSoftware/GameNetworkingSockets#building) the native library with all required dependencies before you get started.

Usage
--------
Before starting to work, the library should be initialized using `Valve.Sockets.Library.Initialize(StringBuilder errorMessage);` function.

After the work is done, deinitialize the library using `Valve.Sockets.Library.Deinitialize();` function.

### .NET environment
##### Start a new server:
```c#
NetworkingSockets server = new NetworkingSockets();

uint listenSocket = server.CreateListenSocket(port);

StatusCallback status = (info) => {
	switch (info.connectionInfo.state) {
		case ConnectionState.None:
			break;

		case ConnectionState.Connecting:
			server.AcceptConnection(info.connection);
			break;

		case ConnectionState.Connected:
			Console.WriteLine("Client connected - ID: " + info.connection + ", IP: " + info.connectionInfo.remoteIP.ParseIP());
			break;

		case ConnectionState.ClosedByPeer:
			server.CloseConnection(info.connection);
			Console.WriteLine("Client disconnected - ID: " + info.connection + ", IP: " + info.connectionInfo.remoteIP.ParseIP());
			break;
	}
};

NetworkingMessage[] netMessages = new NetworkingMessage[1];

while (!Console.KeyAvailable) {
	server.DispatchCallback(status);

	int netMessagesCount = server.ReceiveMessagesOnListenSocket(listenSocket, ref netMessages, 1);

	if (netMessagesCount > 0) {
		Console.WriteLine("Message received from - ID: " + netMessages[0].connection + ", Channel ID: " + netMessages[0].channel + ", Data length: " + netMessages[0].length);
		netMessages[0].Destroy();
	}

	Thread.Sleep(15);
}
```

##### Start a new client:
```c#
NetworkingSockets client = new NetworkingSockets();

uint connection = client.Connect(ip, port);

StatusCallback status = (info) => {
	switch (info.connectionInfo.state) {
		case ConnectionState.None:
			break;

		case ConnectionState.Connected:
			Console.WriteLine("Client connected to server - ID: " + connection);
			break;

		case ConnectionState.ClosedByPeer:
			client.CloseConnection(connection);
			Console.WriteLine("Client disconnected from server");
			break;

		case ConnectionState.ProblemDetectedLocally:
			client.CloseConnection(connection);
			Console.WriteLine("Client unable to connect");
			break;
	}
};

NetworkingMessage[] netMessages = new NetworkingMessage[1];

while (!Console.KeyAvailable) {
	client.DispatchCallback(status);

	int netMessagesCount = client.ReceiveMessagesOnConnection(connection, ref netMessages, 1);

	if (netMessagesCount > 0) {
		Console.WriteLine("Message received from server - Channel ID: " + netMessages[0].channel + ", Data length: " + netMessages[0].length);
		netMessages[0].Destroy();
	}

	Thread.Sleep(15);
}
```

##### Create and send a new message:
```c#
byte[] data = new byte[64];

sockets.SendMessageToConnection(connection, data);
```

##### Copy payload from a message:
```c#
byte[] buffer = new byte[1024];

netMessages[0].CopyTo(buffer);
```

##### Set a custom configuration:
```c#
sockets.SetConfigurationValue(ConfigurationValue.FakePacketLagSend, 80);
sockets.SetConfigurationValue(ConfigurationValue.FakePacketLossSend, 25);
sockets.SetConfigurationValue(ConfigurationValue.FakePacketReorderSend, 25);
sockets.SetConfigurationValue(ConfigurationValue.TimeoutSecondsInitial, 30);
sockets.SetConfigurationValue(ConfigurationValue.TimeoutSecondsConnected, 60);
```

##### Set a hook for debug information:
```c#
DebugCallback debug = (type, message) => {
	Console.WriteLine("Debug - Type: " + type + ", Message: " + message);
};

Library.SetDebugCallback(10, debug);
```

### Unity
Usage is almost the same as in the .NET environment, except that the console functions must be replaced with functions provided by Unity. If the `NetworkingSockets.DispatchCallback()` will be called in a game loop, then keep Unity run in background by enabling the appropriate option in the player settings.

API reference
--------
### Enumerations
#### SendType
Definitions of a flags for `NetworkingSockets.SendMessageToConnection()` function:

`SendType.Unreliable` unreliable, delivery of message is not guaranteed.

`SendType.Reliable` reliable, a message must be received by the target connection and resend attempts should be made until the message is delivered.

`SendType.NoNagle` a message will not be grouped with other messages within a timer.

`SendType.NoDelay` a message will not be buffered if it can't be sent relatively quickly.

#### ConnectionState
Definitions of connection states for `ConnectionInfo.state` field:

`ConnectionState.None` dummy state, the connection doesn't exist or has already been closed.

`ConnectionState.Connecting` in-progress of establishing a connection initiated by `NetworkingSockets.Connect()` function.

`ConnectionState.FindingRoute` if the server accepts the connection, then this connection switch into the rendezvous state, but the end-to-end route is still have not yet established (through the relay network).

`ConnectionState.Connected` a connection request initiated by `NetworkingSockets.Connect()` function has completed.

`ConnectionState.ClosedByPeer` a connection has been closed by the peer, but not closed locally. If there are any messages in the inbound queue, they can be retrieved. Otherwise, nothing may be done with the connection except to close it. The connection still exists from an API perspective and must be closed to free up resources.

`ConnectionState.ProblemDetectedLocally` a disruption in the connection has been detected locally. Attempts to send further messages will fail. Any remaining received messages in the queue are available. The connection still exists from an API perspective and must be closed to free up resources.

#### ConfigurationString
Definitions of configuration strings: 

`ConfigurationString.ClientForceRelayCluster` code of relay cluster to use. If not empty, only relays in that cluster will be used.

`ConfigurationString.ClientDebugTicketAddress` generate an unsigned ticket for debugging, using the specified gameserver address. Router must be configured to accept unsigned tickets.

`ConfigurationString.ClientForceProxyAddr` comma-separated list for debugging, to override relays from the config with this set. 

#### ConfigurationValue
Definitions of configuration values: 

`ConfigurationValue.FakeMessageLossSend` randomly discard unreliable messages instead of sending. Expected value 0-100.

`ConfigurationValue.FakeMessageLossRecv` randomly discard unreliable messages upon receive. Expected value 0-100.

`ConfigurationValue.FakePacketLossSend` randomly discard packets instead of sending. Expected value 0-100.

`ConfigurationValue.FakePacketLossRecv` randomly discard packets upon receive. Expected value 0-100.

`ConfigurationValue.FakePacketReorderSend` globally reorder outbound packets by N percentage. Expected value 0-100.

`ConfigurationValue.FakePacketReorderRecv` globally reorder inbound packets by N percentage. Expected value 0-100.

`ConfigurationValue.FakePacketLagSend` globally delay all outbound packets by N milliseconds before sending.

`ConfigurationValue.FakePacketLagRecv` globally delay all inbound packets by N milliseconds before processing.

`ConfigurationValue.FakePacketReorderTime` amount of delay in milliseconds, to apply packets reordering.

`ConfigurationValue.SendBufferSize` an upper limit of buffered pending bytes to be sent. If this limit is reached, then `NetworkingSockets.SendMessageToConnection()` function will return `Result.LimitExceeded`. Default is 524,288 bytes.

`ConfigurationValue.MaxRate` a maximum send rate clamp. This value will control the maximum allowed sending rate that congestion is allowed to reach. Default is 0 which means no limit.

`ConfigurationValue.MinRate` a minimum send rate clamp. This value will control the minimum allowed sending rate that congestion is allowed to reach. Default is 0 which means no limit.

`ConfigurationValue.NagleTime` set the Nagle timer in microseconds.  When `NetworkingSockets.SendMessageToConnection()` is called, if the outgoing message is less than the size of the MTU, it will be queued for a delay equal to the Nagle timer value. This is to ensure that if the application sends several small messages rapidly, they have coalesced into a single packet. See historical [RFC 896](https://tools.ietf.org/html/rfc896). Default is 5000 microseconds.

`ConfigurationValue.LogLevelAckRTT` set to true (non-zero) to enable logging of RTT based on acks. This doesn't track all sources of RTT, just the inline ones based on acks, but those are the most common.

`ConfigurationValue.LogLevelPacket` a log level of SNP packet decoding.

`ConfigurationValue.LogLevelMessage` a log when messages are sent/received.

`ConfigurationValue.LogLevelPacketGaps` a log level when individual packets drop.

`ConfigurationValue.LogLevelP2PRendezvous` a log level for P2P rendezvous.

`ConfigurationValue.LogLevelRelayPings` a log level for sending and receiving pings to relays.

`ConfigurationValue.ClientConsecutitivePingTimeoutsFailInitial` if the first N pings to a port fail, mark that port as unavailable for a while, and try a different one. Some ISPs and routers may drop the first packet, so setting this to 1 may greatly disrupt communications.

`ConfigurationValue.ClientConsecutitivePingTimeoutsFail` if N consecutive pings to a port fail, after having received successful communication, mark that port as unavailable for a while, and try a different one.

`ConfigurationValue.ClientMinPingsBeforePingAccurate` the minimum number of lifetime pings that need to send, before think that estimate is solid. The first ping to each cluster is very often delayed because of NAT, routers not having the best route, etc. Until a sufficient number of pings is sent, our estimate is often inaccurate.

`ConfigurationValue.ClientSingleSocket` set all datagram traffic to originate from the same local port. By default, a new UDP socket is open up (on a different local port) for each relay. This is not optimal, but it works around some routers that don't implement NAT properly. If intermittent problems occur talking to relays that might be NAT related, try toggling this flag.

`ConfigurationValue.IPAllowWithoutAuth` don't automatically fail IP connections that don't have a strong authenticator. On clients, this means connection attempts will be made even if it can't get a cert. On the server, it means that a connection will not be automatically rejected due to authentication failure.

`ConfigurationValue.TimeoutSecondsInitial` a timeout value in seconds, to use when first connecting.

`ConfigurationValue.TimeoutSecondsConnected` a timeout value in seconds, to use after connection is established.

#### Result
Definitions of operation result: 

`Result.OK` success.

`Result.Fail` generic failure.

`Result.NoConnection` failed network connection.

`Result.InvalidParam` a parameter is incorrect.

`Result.InvalidState` called object was in an invalid state.

`Result.Ignored` target is ignoring sender.

### Delegates
#### Socket callbacks
Provides per socket events.

`StatusCallback(StatusInfo info)` notifies when dispatch mechanism on the listen socket returns a connection state.

#### Library callbacks
Provides per application events.

`DebugCallback(int type, string message)` notifies when debug information with the desired verbosity come up.

### Structures
#### StatusInfo
Contains marshalled data with connection state.

`StatusInfo.connection` the connection ID.

`StatusInfo.connectionInfo` essentially `ConnectionInfo` structure with marshalled data.

#### ConnectionInfo
Contains marshalled data with connection info.

`ConnectionInfo.userData` an user-supplied data set using `NetworkingSockets.SetConnectionUserData()` function.

`ConnectionInfo.listenSocket` the listen socket for this connection.

`ConnectionInfo.remoteIP` the remote IP address of the connection. Can be parsed into a printable form using `remoteIP.ParseIP()` function.

`ConnectionInfo.remotePort` the remote port of the connection.

`ConnectionInfo.state` the high-level state of the connection described in the `ConnectionState` enumeration.

`ConnectionInfo.endReason` the basic cause of the connection termination or problem.

`ConnectionInfo.endDebug` the explanation in a human-readable form for connection termination or problem. This is intended for debugging diagnostic purposes only, not for displaying to users. It might have some details specific to the issue.

#### ConnectionStatus
Contains marshalled data with connection status for frequent requests.

`ConnectionStatus.state` the high-level state of the connection described in the `ConnectionState` enumeration.

`ConnectionStatus.ping` the current ping in milliseconds.

`ConnectionStatus.connectionQualityLocal` a connection quality measured locally (percentage of packets delivered end-to-end in order).

`ConnectionStatus.connectionQualityRemote` a packet delivery success rate as observed from the remote host.

`ConnectionStatus.outPacketsPerSecond` 

`ConnectionStatus.outBytesPerSecond` 

`ConnectionStatus.inPacketsPerSecond` 

`ConnectionStatus.inBytesPerSecond` 

`ConnectionStatus.sendRateBytesPerSecond` the estimated rate at which data can be sent to a peer. It could be significantly higher than `ConnectionStatus.outBytesPerSecond`, meaning the capacity of the channel is higher than sent data.

`ConnectionStatus.pendingUnreliable` 

`ConnectionStatus.pendingReliable` 

`ConnectionStatus.sentUnackedReliable` the number of bytes of reliable data that has been placed the wire, but for which not yet received an acknowledgment, and thus might have to be re-transmitted.

`ConnectionStatus.queueTime` 

#### NetworkingMessage
Contains marshalled data of networking message.

`NetworkingMessage.userData` an user-supplied set using `NetworkingSockets.SetConnectionUserData()` function.

`NetworkingMessage.timeReceived` the local timestamp when the message was received.

`NetworkingMessage.messageNumber` the message number assigned by the sender.

`NetworkingMessage.data` the payload of a message. Can be copied using `data.CopyTo(byte[] destination)` function.

`NetworkingMessage.length` the length of the payload.

`NetworkingMessage.connection` the connection ID from which the message came from.

`NetworkingMessage.channel` the channel number the message was received on.

### Classes
#### NetworkingSockets
Contains a managed pointer to the sockets.

`NetworkingSockets.CreateListenSocket(string ip, ushort port)` creates a socket and returns a socket ID that listens for incoming connections which initiated by `NetworkingSockets.Connect()` function. An IP address parameter is optional.

`NetworkingSockets.Connect(string ip, ushort port)` initiates an IPv4 connection to a foreign host. Returns a local connection ID.

`NetworkingSockets.AcceptConnection(Connection connection)` accepts an incoming connection that has received on a listen socket. When a connection attempt is received (perhaps after a few basic handshake packets have been exchanged to prevent trivial spoofing), a connection interface object is created in the `ConnectionState.Connecting` state and a `StatusCallback()` is called. Returns a result described in the `Result` enumeration.

`NetworkingSockets.CloseConnection(Connection connection, int reason, string debug, bool enableLinger)` disconnects from the host and invalidates the connection handle. Any unread data on the connection is discarded. The reason parameter is an optional user-supplied code that will be received on the other end and recorded (when possible) in backend analytics. The value should be less than `Library.maxCloseReasonValue`. The debug parameter is an optional human-readable diagnostic string that will be received on the other end and recorded (when possible) in backend analytics. If the user wishes to put the socket into a lingering state, where an attempt is made to flush any remaining sent data, the linger parameter should be enabled, otherwise reliable data is not flushed. If the connection has already ended, the reason code, debug string and linger parameter is ignored. Returns true on success or false on failure.

`NetworkingSockets.CloseListenSocket(ListenSocket socket, string remoteReason)` destroys the listen socket, and all the client sockets generated by accepting connections on the listen socket. The remote reason determines what cleanup actions are performed on the client sockets being destroyed. If cleanup is requested and the user has requested the listen socket bound to a particular local port to facilitate direct IPv4 connections, then the underlying UDP socket must remain open until all clients have been cleaned up. Returns true on success or false on failure.

`NetworkingSockets.SetConnectionUserData(Connection peer, long userData)` sets a user-supplied data for the connection. Returns true on success or false on failure.

`NetworkingSockets.GetConnectionUserData(Connection peer)` returns a user-supplied data or -1 if a handle is invalid or if any data has not be set for the connection.

`NetworkingSockets.SetConnectionName(Connection peer, string name)` sets a name for the connection, used mostly for debugging.

`NetworkingSockets.GetConnectionName(Connection peer, StringBuilder name, int maxLength)` fetches connection name to the mutable string. Returns true on success or false on failure.

`NetworkingSockets.SendMessageToConnection(Connection connection, byte[] data, SendType flags)` sends a message to the host on the connected socket. The send type parameter is optional. Multiple types can be specified at once. Returns a result described in the `Result` enumeration.

`NetworkingSockets.FlushMessagesOnConnection(Connection connection)` 

`NetworkingSockets.ReceiveMessagesOnConnection(Connection connection, ref NetworkingMessage[] messages, int maxMessages)` 

`NetworkingSockets.ReceiveMessagesOnListenSocket(ListenSocket socket, ref NetworkingMessage[] messages, int maxMessages)` 

`NetworkingSockets.GetConnectionInfo(Connection connection, ref ConnectionInfo info)` 

`NetworkingSockets.GetQuickConnectionStatus(Connection connection, ConnectionStatus status)` 

`NetworkingSockets.GetDetailedConnectionStatus(Connection connection, StringBuilder status, int statusLength)` 

`NetworkingSockets.GetListenSocketInfo(ListenSocket socket, uint ip, ushort port)` 

`NetworkingSockets.CreateSocketPair(Connection connectionOne, Connection connectionTwo, bool useNetworkLoopback)` 

`NetworkingSockets.GetConnectionDebugText(Connection connection, StringBuilder debugText, int debugLength)` 

`NetworkingSockets.GetConfigurationValue(ConfigurationValue configurationValue)` 

`NetworkingSockets.SetConfigurationValue(ConfigurationValue configurationValue, int value)` 

`NetworkingSockets.GetConfigurationValueName(ConfigurationValue configurationValue)` 

`NetworkingSockets.GetConfigurationString(ConfigurationString configurationString, StringBuilder destination, int destinationLength)` 

`NetworkingSockets.SetConfigurationString(ConfigurationString configurationString, string inputString)` 

`NetworkingSockets.GetConfigurationStringName(ConfigurationString configurationString)` 

`NetworkingSockets.GetConnectionConfigurationValue(Connection connection, ConfigurationValue configurationValue)` 

`NetworkingSockets.SetConnectionConfigurationValue(Connection connection, ConfigurationValue configurationValue, int value)` 

`NetworkingSockets.DispatchCallback(StatusCallback callback)` dispatches one callback per call if available.

#### Library
Contains constant fields.

`Library.maxCloseReasonValue` the maximum value of the reason code when a connection is closed.

`Library.maxCloseMessageLength` the maximum length of the reason string in bytes when a connection is closed.

`Library.Initialize(StringBuilder errorMessage)` initializes the native library. Capacity of a mutable string for an error message must be equal to `Library.maxErrorMessageLength`.

`Library.Deinitialize()` deinitializes the native library. Should be called after the work is done.

`Library.SetDebugCallback(int detailLevel, DebugCallback callback)` setups callback for debug output.

`Library.Time` returns a current local monotonic time in microseconds. It never reset while the application remains alive.

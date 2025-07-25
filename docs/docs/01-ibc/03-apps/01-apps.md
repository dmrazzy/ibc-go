---
title: IBC Applications
sidebar_label: IBC Applications
sidebar_position: 1
slug: /ibc/apps/apps
---

# IBC Applications

:::warning
This page is relevant for IBC Classic, naviagate to the IBC v2 applications page for information on v2 apps
:::

Learn how to configure your application to use IBC and send data packets to other chains.

This document serves as a guide for developers who want to write their own Inter-blockchain
Communication Protocol (IBC) applications for custom use cases.

Due to the modular design of the IBC protocol, IBC
application developers do not need to concern themselves with the low-level details of clients,
connections, and proof verification, however a brief explaination is given.  Then the document goes into detail on the abstraction layer most relevant for application
developers (channels and ports), and describes how to define your own custom packets, and
`IBCModule` callbacks.

To have your module interact over IBC you must: bind to a port(s), define your own packet data and acknowledgement structs as well as how to encode/decode them, and implement the
`IBCModule` interface. Below is a more detailed explanation of how to write an IBC application
module correctly.

:::note

## Pre-requisites Readings

- [IBC Overview](../01-overview.md)
- [IBC default integration](../02-integration.md)

:::

## Create a custom IBC application module

### Implement `IBCModule` Interface and callbacks

The Cosmos SDK expects all IBC modules to implement the [`IBCModule`
interface](https://github.com/cosmos/ibc-go/tree/main/modules/core/05-port/types/module.go). This
interface contains all of the callbacks IBC expects modules to implement. This section will describe
the callbacks that are called during channel handshake execution.

Here are the channel handshake callbacks that modules are expected to implement:

```go
// Called by IBC Handler on MsgOpenInit
func (k Keeper) OnChanOpenInit(ctx sdk.Context,
  order channeltypes.Order,
  connectionHops []string,
  portID string,
  channelID string,
  counterparty channeltypes.Counterparty,
  version string,
) error {

  // ... do custom initialization logic

  // Use above arguments to determine if we want to abort handshake
  // Examples: Abort if order == UNORDERED,
  // Abort if version is unsupported
  err := checkArguments(args)
  return err
}

// Called by IBC Handler on MsgOpenTry
OnChanOpenTry(
  ctx sdk.Context,
  order channeltypes.Order,
  connectionHops []string,
  portID,
  channelID string,
  counterparty channeltypes.Counterparty,
  counterpartyVersion string,
) (string, error) {
  // ... do custom initialization logic

  // Use above arguments to determine if we want to abort handshake
  if err := checkArguments(args); err != nil {
    return err
  }

  // Construct application version 
  // IBC applications must return the appropriate application version
  // This can be a simple string or it can be a complex version constructed
  // from the counterpartyVersion and other arguments. 
  // The version returned will be the channel version used for both channel ends. 
  appVersion := negotiateAppVersion(counterpartyVersion, args)
  
  return appVersion, nil
}

// Called by IBC Handler on MsgOpenAck
OnChanOpenAck(
  ctx sdk.Context,
  portID,
  channelID string,
  counterpartyVersion string,
) error {
  // ... do custom initialization logic

  // Use above arguments to determine if we want to abort handshake
  err := checkArguments(args)
  return err
}

// Called by IBC Handler on MsgOpenConfirm
OnChanOpenConfirm(
  ctx sdk.Context,
  portID,
  channelID string,
) error {
  // ... do custom initialization logic

  // Use above arguments to determine if we want to abort handshake
  err := checkArguments(args)
  return err
}
```

The channel closing handshake will also invoke module callbacks that can return errors to abort the
closing handshake. Closing a channel is a 2-step handshake, the initiating chain calls
`ChanCloseInit` and the finalizing chain calls `ChanCloseConfirm`.

```go
// Called by IBC Handler on MsgCloseInit
OnChanCloseInit(
  ctx sdk.Context,
  portID,
  channelID string,
) error {
  // ... do custom finalization logic

  // Use above arguments to determine if we want to abort handshake
  err := checkArguments(args)
  return err
}

// Called by IBC Handler on MsgCloseConfirm
OnChanCloseConfirm(
  ctx sdk.Context,
  portID,
  channelID string,
) error {
  // ... do custom finalization logic

  // Use above arguments to determine if we want to abort handshake
  err := checkArguments(args)
  return err
}
```

#### Channel Handshake Version Negotiation

Application modules are expected to verify versioning used during the channel handshake procedure.

- `ChanOpenInit` callback should verify that the `MsgChanOpenInit.Version` is valid
- `ChanOpenTry` callback should construct the application version used for both channel ends. If no application version can be constructed, it must return an error.
- `ChanOpenAck` callback should verify that the `MsgChanOpenAck.CounterpartyVersion` is valid and supported.

IBC expects application modules to perform application version negotiation in `OnChanOpenTry`. The negotiated version
must be returned to core IBC. If the version cannot be negotiated, an error should be returned.

Versions must be strings but can implement any versioning structure. If your application plans to
have linear releases then semantic versioning is recommended. If your application plans to release
various features in between major releases then it is advised to use the same versioning scheme
as IBC. This versioning scheme specifies a version identifier and compatible feature set with
that identifier. Valid version selection includes selecting a compatible version identifier with
a subset of features supported by your application for that version. The struct is used for this
scheme can be found in `03-connection/types`.

Since the version type is a string, applications have the ability to do simple version verification
via string matching or they can use the already implemented versioning system and pass the proto
encoded version into each handhshake call as necessary.

ICS20 currently implements basic string matching with a single supported version.

### ICS4Wrapper

The IBC application interacts with core IBC through the `ICS4Wrapper` interface for any application-initiated actions like: `SendPacket` and `WriteAcknowledgement`. This may be directly the IBCChannelKeeper or a middleware that sits between the application and the IBC ChannelKeeper.

If the application is being wired with a custom middleware, the application **must** have its ICS4Wrapper set to the middleware directly above it on the stack through the following call:

```go
// SetICS4Wrapper sets the ICS4Wrapper. This function may be used after
// the module's initialization to set the middleware which is above this
// module in the IBC application stack.
// The ICS4Wrapper **must** be used for sending packets and writing acknowledgements
// to ensure that the middleware can intercept and process these calls.
// Do not use the channel keeper directly to send packets or write acknowledgements
// as this will bypass the middleware.
SetICS4Wrapper(wrapper ICS4Wrapper)
```

### Custom Packets

Modules connected by a channel must agree on what application data they are sending over the
channel, as well as how they will encode/decode it. This process is not specified by IBC as it is up
to each application module to determine how to implement this agreement. However, for most
applications this will happen as a version negotiation during the channel handshake. While more
complex version negotiation is possible to implement inside the channel opening handshake, a very
simple version negotiation is implemented in the [ibc-transfer module](https://github.com/cosmos/ibc-go/tree/main/modules/apps/transfer/module.go).

Thus, a module must define its custom packet data structure, along with a well-defined way to
encode and decode it to and from `[]byte`.

```go
// Custom packet data defined in application module
type CustomPacketData struct {
  // Custom fields ...
}

EncodePacketData(packetData CustomPacketData) []byte {
  // encode packetData to bytes
}

DecodePacketData(encoded []byte) (CustomPacketData) {
  // decode from bytes to packet data
}
```

Then a module must encode its packet data before sending it through IBC.

```go
// Sending custom application packet data
data := EncodePacketData(customPacketData)
packet.Data = data
// Send packet to IBC, authenticating with channelCap
sequence, err := IBCChannelKeeper.SendPacket(
  ctx, 
  sourcePort, 
  sourceChannel, 
  timeoutHeight, 
  timeoutTimestamp, 
  data,
)
```

A module receiving a packet must decode the `PacketData` into a structure it expects so that it can
act on it.

```go
// Receiving custom application packet data (in OnRecvPacket)
packetData := DecodePacketData(packet.Data)
// handle received custom packet data
```

#### Packet Flow Handling

Just as IBC expected modules to implement callbacks for channel handshakes, IBC also expects modules
to implement callbacks for handling the packet flow through a channel.

Once a module A and module B are connected to each other, relayers can start relaying packets and
acknowledgements back and forth on the channel.

![IBC packet flow diagram](https://media.githubusercontent.com/media/cosmos/ibc/old/spec/ics-004-channel-and-packet-semantics/channel-state-machine.png)

Briefly, a successful packet flow works as follows:

1. module A sends a packet through the IBC module
2. the packet is received by module B
3. if module B writes an acknowledgement of the packet then module A will process the
   acknowledgement
4. if the packet is not successfully received before the timeout, then module A processes the
   packet's timeout.

##### Sending Packets

Modules do not send packets through callbacks, since the modules initiate the action of sending
packets to the IBC module, as opposed to other parts of the packet flow where msgs sent to the IBC
module must trigger execution on the port-bound module through the use of callbacks. Thus, to send a
packet a module simply needs to call `SendPacket` on the `IBCChannelKeeper`.

```go
// Sending custom application packet data
data := EncodePacketData(customPacketData)
// Send packet to IBC, authenticating with channelCap
sequence, err := IBCChannelKeeper.SendPacket(
  ctx, 
  sourcePort, 
  sourceChannel, 
  timeoutHeight, 
  timeoutTimestamp, 
  data,
)
```

##### Receiving Packets

To handle receiving packets, the module must implement the `OnRecvPacket` callback. This gets
invoked by the IBC module after the packet has been proved valid and correctly processed by the IBC
keepers. Thus, the `OnRecvPacket` callback only needs to worry about making the appropriate state
changes given the packet data without worrying about whether the packet is valid or not.

Modules may return to the IBC handler an acknowledgement which implements the Acknowledgement interface.
The IBC handler will then commit this acknowledgement of the packet so that a relayer may relay the
acknowledgement back to the sender module.

The state changes that occurred during this callback will only be written if:

- the acknowledgement was successful as indicated by the `Success()` function of the acknowledgement
- if the acknowledgement returned is nil indicating that an asynchronous process is occurring

NOTE: Applications which process asynchronous acknowledgements must handle reverting state changes
when appropriate. Any state changes that occurred during the `OnRecvPacket` callback will be written
for asynchronous acknowledgements.

```go
OnRecvPacket(
  ctx sdk.Context,
  packet channeltypes.Packet,
) ibcexported.Acknowledgement {
  // Decode the packet data
  packetData := DecodePacketData(packet.Data)

  // do application state changes based on packet data and return the acknowledgement
  // NOTE: The acknowledgement will indicate to the IBC handler if the application 
  // state changes should be written via the `Success()` function. Application state
  // changes are only written if the acknowledgement is successful or the acknowledgement
  // returned is nil indicating that an asynchronous acknowledgement will occur.
  ack := processPacket(ctx, packet, packetData)

  return ack
}
```

The Acknowledgement interface:

```go
// Acknowledgement defines the interface used to return
// acknowledgements in the OnRecvPacket callback.
type Acknowledgement interface {
  Success() bool
  Acknowledgement() []byte
}
```

### Acknowledgements

Modules may commit an acknowledgement upon receiving and processing a packet in the case of synchronous packet processing.
In the case where a packet is processed at some later point after the packet has been received (asynchronous execution), the acknowledgement
will be written once the packet has been processed by the application which may be well after the packet receipt.

NOTE: Most blockchain modules will want to use the synchronous execution model in which the module processes and writes the acknowledgement
for a packet as soon as it has been received from the IBC module.

This acknowledgement can then be relayed back to the original sender chain, which can take action
depending on the contents of the acknowledgement.

Just as packet data was opaque to IBC, acknowledgements are similarly opaque. Modules must pass and
receive acknowledegments with the IBC modules as byte strings.

Thus, modules must agree on how to encode/decode acknowledgements. The process of creating an
acknowledgement struct along with encoding and decoding it, is very similar to the packet data
example above. [ICS 04](https://github.com/cosmos/ibc/blob/master/spec/core/ics-004-channel-and-packet-semantics#acknowledgement-envelope)
specifies a recommended format for acknowledgements. This acknowledgement type can be imported from
[channel types](https://github.com/cosmos/ibc-go/tree/main/modules/core/04-channel/types).

While modules may choose arbitrary acknowledgement structs, a default acknowledgement types is provided by IBC [here](https://github.com/cosmos/ibc-go/blob/main/proto/ibc/core/channel/v1/channel.proto):

```proto
// Acknowledgement is the recommended acknowledgement format to be used by
// app-specific protocols.
// NOTE: The field numbers 21 and 22 were explicitly chosen to avoid accidental
// conflicts with other protobuf message formats used for acknowledgements.
// The first byte of any message with this format will be the non-ASCII values
// `0xaa` (result) or `0xb2` (error). Implemented as defined by ICS:
// https://github.com/cosmos/ibc/tree/master/spec/core/ics-004-channel-and-packet-semantics#acknowledgement-envelope
message Acknowledgement {
  // response contains either a result or an error and must be non-empty
  oneof response {
    bytes  result = 21;
    string error  = 22;
  }
}
```

#### Acknowledging Packets

After a module writes an acknowledgement, a relayer can relay back the acknowledgement to the sender module. The sender module can
then process the acknowledgement using the `OnAcknowledgementPacket` callback. The contents of the
acknowledgement is entirely up to the modules on the channel (just like the packet data); however, it
may often contain information on whether the packet was successfully processed along
with some additional data that could be useful for remediation if the packet processing failed.

Since the modules are responsible for agreeing on an encoding/decoding standard for packet data and
acknowledgements, IBC will pass in the acknowledgements as `[]byte` to this callback. The callback
is responsible for decoding the acknowledgement and processing it.

```go
OnAcknowledgementPacket(
  ctx sdk.Context,
  packet channeltypes.Packet,
  acknowledgement []byte,
) (*sdk.Result, error) {
  // Decode acknowledgement
  ack := DecodeAcknowledgement(acknowledgement)

  // process ack
  res, err := processAck(ack)
  return res, err
}
```

#### Timeout Packets

If the timeout for a packet is reached before the packet is successfully received or the
counterparty channel end is closed before the packet is successfully received, then the receiving
chain can no longer process it. Thus, the sending chain must process the timeout using
`OnTimeoutPacket` to handle this situation. Again the IBC module will verify that the timeout is
indeed valid, so our module only needs to implement the state machine logic for what to do once a
timeout is reached and the packet can no longer be received.

```go
OnTimeoutPacket(
  ctx sdk.Context,
  packet channeltypes.Packet,
) (*sdk.Result, error) {
  // do custom timeout logic
}
```

### Routing

As mentioned above, modules must implement the IBC module interface (which contains both channel
handshake callbacks and packet handling callbacks). The concrete implementation of this interface
must be registered with the module name as a route on the IBC `Router`.

```go
// app.go
func NewApp(...args) *App {
// ...

// Create static IBC router, add module routes, then set and seal it
ibcRouter := port.NewRouter()

ibcRouter.AddRoute(ibctransfertypes.ModuleName, transferModule)
// Note: moduleCallbacks must implement IBCModule interface
ibcRouter.AddRoute(moduleName, moduleCallbacks)

// Setting Router will finalize all routes by sealing router
// No more routes can be added
app.IBCKeeper.SetRouter(ibcRouter)
```

## Working Example

For a real working example of an IBC application, you can look through the `ibc-transfer` module
which implements everything discussed above.

Here are the useful parts of the module to look at:

[Binding to transfer
port](https://github.com/cosmos/ibc-go/blob/main/modules/apps/transfer/keeper/genesis.go)

[Sending transfer
packets](https://github.com/cosmos/ibc-go/blob/main/modules/apps/transfer/keeper/relay.go)

[Implementing IBC
callbacks](https://github.com/cosmos/ibc-go/blob/main/modules/apps/transfer/ibc_module.go)

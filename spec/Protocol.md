# DOT Protocol

There are two DOT pub-sub protocols: the journal and the log protocols.
Both are JSON-based, full duplex websocket protocols that also share
the same messages.  So, they are described here together.

## References

Please read:

1. [Operations](Operations.md)
2. [Introduction to operational transforms](https://github.com/dotchain/site/blob/master/IntroductionToOperationalTransforms.md)
3. [URLs](Urls.md)

## Terms

| Term      |  Definition |
|-----------|-------------|
| Operation | Basic unit of change described [here](Operations.md) |
| Model     | The unit of synchronization.  Models are logically treated as if they were JSON entities. |
| Reconciliation | The process of synchronization described [here](https://github.com/dotchain/site/blob/master/IntroductionToOperationalTransforms.md) |
| Journal   | An ordered sequence of raw operations sent by clients |
| Log       | The same sequence of operations as in a journal but transformed to a form that clients can apply them to rebuild a model |

## Transport

Both Journal and Log protocols are websocket based protocols using JSON payloads.
The sub-protocol for journal is `dotj` and the sub-protocol for log is `dotl`.
If a client presents both protocols, the server should prefer `dotj`.

While websockets support compression, not all browsers implement it. So custom
compression protocols are defined: `dotjz` and `dotlz`.  These should use
deflate with FLEVEL `normal` (or `2`).  If the server supports deflate, it
should prefer deflate.

## Multiplexing

The protocol is defined to allow multiple models to be synchronized in parallel
using a single websocket connection.  All messages include a `ModelID` for this
purpose though the specific field name differs based on the message type.

## UTF16 string and floating point issues

All strings defined here (and in [Operations](Operations.md)) are expected to be
UTF strings.  This plays a role in two different ways:

Where IDs are compared for equality, there is no unicode equivalence algorithms
applied.  So, all clients should preserve the byte order presented here when the
IDs are read and written.

The other factor is that the offset and sizes used in Splice and Move
[operations](Operations.md) refer to UTF16 values.  That is, those operations
are implemented on UTF16 strings.

Similar issues arise with floating point numbers and their representations and a
careful approach must be taken to ensure full convergence. When in doubt, the
default representation should be Javascript.

## Messages

The following table summarizes the types of messages. The SubID column
identifies the field in the json payload where the subscription ID is
provided. It also acts as the field that identifies the message type.


| Name        | Sender | SubID  field |
|-------------|--------|---------------|
| Subscribe   | Client | Subscribe     |
| Unsubscribe | Client | Unsubscribe   |
| Append      | Client | Append        |
| Bootstrap   | Server | Bootstrap     |
| Notification| Server | Notification  |
| Error       | Server } Error         |

## Journal service

The journal service is a very thin service that persists messages sent by
clients and notifies clients of changes.  It does not do transforms.

Clients are expected to send the raw operations with two very simple
requirements:  

1.  Every operation must have the `Parents` array such that the
first entry is always the ID of the last operation the client has
received from the journal service for that model.

2. If the client had sent any operations (or operations are in flight)
for that model and those operations have not yet appeared in the journal
(as a notification from the journal service), any future operations must
include the ID of the last such operation as the second entry in the
`Parents` field.

These two entries are called the BasisID and ParentID respectively
and are required for proper reconciliation procedures and getting
all clients to converge to the same model.

## Log service, reconciliation

The log service maintains a sequence of transformed operations, one for each
entry in the journal (except for duplicates which are dropped).  It allows
the clients to send operations without ever having to transform them.
The service responds with `compensating operations` that the clients can
apply to get them to the same converged state as a client that receives
raw operations and does its own transformations.

In other words, clients of the `log service` can work with the OT system
without any real awareness of reconciliation or any code to do transforms.

For the reconciliation process to work correctly, clients MUST follow the
following rules:

1. All operations sent by the client must have two IDs in Parents field of
the operations.  The first is the BasisID and the second the ParentID. This
is the same as for the Journal service.

2. The BasisID must be set to the last operation that the client has known
to be persisted in the journal.  This is the last operation that a client
received for this model via a [Notification](#notification) from the log
service.  If the last message received by the client for the model is 
actually a [Bootstrap](#bootstrap) message, the last entry in the `Rebased`
field is the last known operation in the journal.

3. The ParentID is the id of the last operation the client sent to the
server via an [Append](#append) message for this model. If the last message
the client sent for this model is a [Subscribe](#subscribe), then
this will be the ID of the last operation in the `ClientOps` field of the
[Subscribe](#subscribe) message.  If the client is reconnecting without
any pending operations (i.e. it specifies a Reconnect field in the
[Subscribe](#subscribe) message), the client should send its local last
parent ID  in the subscribe so that the server knows where to continue
its messages from. 

4. Whenever a client receives a message from the server, it validates the
received operations against its BasisID and ParentID -- only accepting
the message if the basis ID of the operation matches the basis ID of
the client and the parent ID of the operation matches the parent ID 
tracked by the client.  If not, the operation is ignored.  Note that
when a client accepts an operation, it effectively updates its tracking
basis ID to be the ID of that operation since the log service only
ever sends the client messages that are committed to the log. 

A client can omit an empty ParentID entry in the `Parents` array but it
cannot omit an empty `BasisID` field (the first element of the array is
always the BasisID). The log service SHOULD not omit either field to
make it easy for clients to be developed.

## Subscribe

A client subscribes to the changes for a model with the **subscribe** request:

```js
{
	"Subscribe": SubID
        "ModelID": ModelID
	"ClientOps": LocalClientOperations
        "Reconnect" ReconnectInfo
}
```

| Field       | Type   | Optional |  Description |
|-------------|--------|----------|--------------|
| Subscribe   | String | No       | Sub ID       |
| ModelID     | String | Yes      | Model ID     |
| ClientOps   | Array of [ops](Operations.md) | Yes | Client operations not yet in journal |
| Reconnect   | Hash   | Yes      | BasisID and ParentID if client is reconnecting |


Note that the `Reconnect` can be empty or it can be a hash like so:

```js
{
	"BasisID": BasisID
        "ParentID": ParentID
}
```

### Journal protocol notes

There is no response to this message.

If the `Reconnect.BasisID` field is empty or not provided, the subscription
starts from the beginning of the journal.  Otherwise, it starts from the
operation past the one referred to by this ID.

Server does not check for invalid or duplicate operations, they are added to
the journal regardless.  There is no acknowledgement but a client can clean up
its operation once it appears in the journal.

A client cannot use a future `BasisID` to keep the subscription pending
until a matching operation is submitted.

The `Reconnect.ParentID` is ignored by the journal server.

### Log protocol notes

If the `Reconnect` field is null or absent, the server MUST respond with a
[Bootstrap](#bootstrap) message.  This allows a client to bootstrap its
model from scratch by applying the `Rebased` and `ClientRebased` operations
in sequence.

Note the server is free to compact or other-wise change the initial Rebased
field in the response (with the only caveat that the last op ID must be
properly maintained for the client to track its `BasisID` correctly).

A client MUST not send any [Append](#append) messages until the
[Bootstrap](#bootstrap) message if the original subscribe has a missing
or null `Reconnect` field.

When a client has a valid connection established with a local model, and
the websocket connection breaks, the client would need a quick way to 
re-establish the connection.  In that case, the client can do this via
a simple Subscribe and specify the `Reconnect` field with its current
[BasisID](#log-service-reconciliation) and `ParentID`.  The server SHOULD
ignore the `ParentID` if the client also has unacknowledged ops in the
Subscribe request -- instead the server should take the last client op
ID and consider that the `ParentID`

## Unsubscribe

A client can remove a prior subscription made in the current session via
an **unsubscribe** request which is a JSON object like so:

```js
{
	"Unsubscribe": SubID
        "ModelID": ModelID
}
```

| Field       | Type   | Optional |  Description |
|-------------|--------|----------|--------------|
| Unsubscribe | String | No       | Sub ID     |


## Append

The client can publish to the journal and have operations appended via
the **Append** request which is a JSON object like so:

```js
{
	"Append": SubID
	"Ops": <Array of operations structures>
}
```

| Field       | Type   | Optional |  Description |
|-------------|--------|----------|--------------|
| Append      | String | No       | Sub ID     |
| Ops         | Array of [ops](Operations.md) | No | Client operations not yet in journal |
|-------------|--------|----------|--------------|


The uploaded operations should be RAW operations for both the Log service
and the Journal service.

The Journal service will echo these messages back to the client  via a 
[Notification](#notification) message -- once the operations get committed
to persistent storage.  But there is no guarantee that the Notification
messages will be chunked the same way -- the array of operations should
be treated as a stream.

The log service will piggyback an acknowledgement on a [Notification](#notification)
message (or if there is no such message in its queue, it will simply send an
empty notification).  The acknowledgement is only sent when the operation is
persisted into the journal and so a client of the log service can safely remove
any operation so acknowledge from its local storage.

## Bootstrap

The log service will send a Bootstrap message to a client which subscribes
without a ReconnectInfo.

```js
{
        "Bootstrap": SubID
	"ModelID": ModelID
	"Rebased": <Array of operations structures>
	"ClientRebased": <Array of operations structures>
}
```

| Field       | Type   | Optional |  Description |
|-------------|--------|----------|--------------|
| Bootstrap   | String | No       | Sub ID       |
| ModelID     | String | No       | Model ID     |
| Rebased    | Array of [ops](Operations.md) | No | See notes below |
| ClientRebased | Array of [ops](Operations.md) | No | See notes below |

The `Rebased` field includes the set of transformed server operations
that the client can sequentially apply to get the initial model. The
`ClientRebased` field is the transformation of the `ClientOps` that the
client sent in its initial [Subscribe](#subscribe) message.  The client should
apply this on top of the `Rebased` ops to get itself to a good state that
captures the effect of the `ClientOps`.

At this point, the client can make changes to its model and get going.

The client should update its tracking `ParentID` to be the last
operation it had sent via the `ClientOps` (or empty if it sent nothing)
**even if the returned ClientRebased is empty**. The BasisID should be
set to the last operation in the `Rebased` field.

## Notification

Once a subscription is made by a client, the server notifies the client
of any appends to the journal via a **Notification** response which is
a JSON object like so:

```js
{
        "Notification": SubID
	"ModelID": ModelID
	"AckID": last client operation acknowledged
	"Operations": <Array of opertations structures>
}
```

| Field       | Type   | Optional |  Description |
|-------------|--------|----------|--------------|
| Notification  | String | No       | Sub ID       |
| ModelID     | String | No       | Model ID     |
| AckID       | String | Yes      | last client op id acknowledged in journal |
| Operations  | Array of [ops](Operations.md) | Yes | See notes below |


For the Journal service, the Operations refers to the raw untransformed
operations.  The journal service does not fill the AckID field.

For the Log service, this refers to the transformed operation
suitable for a client to apply to its model -- but only if the basisID
and parentID of the operations passes the validation test described
[here](#log-service-reconciliation).

The validation is needed mainly for the case where a client may have more
operations in flight and so any `compensating` actions by the log service
would be invalid.

The log service will also fill the AckID field if a previous client operation
was successfully acknowledged in the journal.  The log service may send an
empty Notification message (i.e. no operations) just for the purpose of
communicating the acknowledgement or it may piggy back this on top of another
Notification.

## Error

The server sends an error message when it encounters an error that cannot
be recovered.  The error payload looks like this:

```js
{
        "Error": SubID
	"ModelID": ModelID
	"Message": actual error message
}
```

| Field       | Type   | Optional |  Description |
|-------------|--------|----------|--------------|
| Error       | String | No       | Sub ID       |
| ModelID     | String | No       | Model ID     |
| Message     | String | No       | error message|

At this point the following self-explanatory messages have been defined:

1. "subscription already exists" (duplicate subscribe request)
2. "subscription does not exist" (append request without subscribing to the model)

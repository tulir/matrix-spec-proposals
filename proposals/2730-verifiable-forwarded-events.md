# Verifiable forwarded events
This is an alternative to [MSC2723](https://github.com/matrix-org/matrix-doc/pull/2723)
that handles the issue of faking forwards.

## Proposal
The proposed solution is copying the entire federation event data, which allows
recipients to validate the signatures even if they are not in the origin room.

As clients generally don't have access to signatures nor any way to validate
them, both sending and validating require server support. Sending is
implemented as a new endpoint, while validating happens automatically and the
server adds the validation result to the top-level `unsigned` object.

### `PUT /_matrix/client/r0/rooms/{roomId}/event/{eventId}/forward/{targetRoomId}/{txnId}`
This endpoint requests the server to find `eventId` from `roomId` and forward
it to `targetRoomId`. The `txnId` behaves the same way as in the existing
`/send` endpoints.

Only unredacted message events can be forwarded. If the given event ID is a
state event, a redaction or a redacted message event, the request will be
rejected with a standard error response using the code `M_NOT_FORWARDABLE`.

To forward an event, the server creates a new event with the same event type
and normal top-level fields. To determine the content, the server has to
inspect the content of the source event:

If the source event was already forwarded from some other room, the `content`
should simply be copied with no modifications. This means that an event
forwarded many times will only remember the original source, not any hops it
made on the way.

If the source event was not a forward, but contains (invalid) data in the
`m.forwarded` key, the request will be rejected with `M_NOT_FORWARDABLE` like
other unforwardable events. This limitation also can be used to intentionally
mark messages as unforwardable (e.g. `"m.forwarded": {"allow": false}`).

If the event does not contain `m.forwarded` at all, the server must generate a
new one. `m.forwarded` is an object that contains all the top-level keys of the
original event, except for `type`, `content` and `unsigned`. The following keys
are therefore at least required:

* `auth_events`
* `prev_events`
* `room_id`
* `sender`
* `depth`
* `origin`
* `origin_server_ts`
* `hashes`
* `signatures`

The following keys may also be present:

* `prev_state`, may be present as an empty array even in non-state events
* `event_id`, only in v1 and v2 rooms

Additionally, the server SHOULD include its own `unsigned` object in the
`m.forwarded` data that contains the `displayname` and `avatar_url` of the
sender.

If the resulting event is too large, the request is rejected with a standard
error response using the code `M_TOO_LARGE`. Before rejecting the request,
servers MAY check if the event would be small enough without the profile data
in `unsigned`, and send the event without that data if it is.

#### Example

<details>
<summary>Source event (federation format)</summary>

```json
{
  "auth_events": [
    "$wChClfXonLE8RZikJ446AXvRpbh_JjDK8sNpMpZbqPs",
    "$RaXN_RayMvoEmMnUHlZScIdSpShT8zggd4p6qcQk9L8",
    "$kFop6R7AohiYSTh_ijUctTujdVTg3rwBPdaMLeZMNrg"
  ],
  "prev_events": [
    "$pIFO6_sI1Ul_3jPixtbnJn_h0Pe0yB__TJD_VCW9Q-Q"
  ],
  "type": "m.room.message",
  "room_id": "!FIIWlyqwNLyMAtmRBF:maunium.net",
  "sender": "@tulir:maunium.net",
  "content": {
    "msgtype": "m.text",
    "body": "test"
  },
  "depth": 115,
  "prev_state": [],
  "origin": "maunium.net",
  "origin_server_ts": 1597257769634,
  "hashes": {
    "sha256": "xBR7NmH2WQBx0auQWEDEYNbcPf9ATlDSwkv9EBxueMI"
  },
  "signatures": {
    "maunium.net": {
      "ed25519:a_xxeS": "cc9XnH9ByO7yadC6CdMhh3c/TN1tQ9FiZdKYyRDi4Og1dZMylmBM9uSI7c4GUEqswLBLxW5DTFU3n7vMHAGhAw"
    }
  },
  "unsigned": {
    "age_ts": 1597257769634
  }
}
```

</details>
<details>
<summary>Forwarded event (client format)</summary>

```json
{
  "type": "m.room.message",
  "room_id": "!eVRGrjZQgJZGNllOkw:grin.hu",
  "event_id": "$r8h8W9A5KS8D65_Df8fwLkTe7aqOm48KmyaJ6tRNAmE",
  "sender": "@tulir:maunium.net",
  "origin_server_ts": 1597263764138,
  "content": {
    "msgtype": "m.text",
    "body": "test",
    "m.forwarded": {
      "auth_events": [
        "$wChClfXonLE8RZikJ446AXvRpbh_JjDK8sNpMpZbqPs",
        "$RaXN_RayMvoEmMnUHlZScIdSpShT8zggd4p6qcQk9L8",
        "$kFop6R7AohiYSTh_ijUctTujdVTg3rwBPdaMLeZMNrg"
      ],
      "prev_events": [
        "$pIFO6_sI1Ul_3jPixtbnJn_h0Pe0yB__TJD_VCW9Q-Q"
      ],
      "room_id": "!FIIWlyqwNLyMAtmRBF:maunium.net",
      "sender": "@tulir:maunium.net",
      "depth": 115,
      "prev_state": [],
      "origin": "maunium.net",
      "origin_server_ts": 1597257769634,
      "hashes": {
        "sha256": "xBR7NmH2WQBx0auQWEDEYNbcPf9ATlDSwkv9EBxueMI"
      },
      "signatures": {
        "maunium.net": {
          "ed25519:a_xxeS": "cc9XnH9ByO7yadC6CdMhh3c/TN1tQ9FiZdKYyRDi4Og1dZMylmBM9uSI7c4GUEqswLBLxW5DTFU3n7vMHAGhAw"
        }
      },
      "unsigned": {
        "displayname": "tulir",
        "avatar_url": "mxc://maunium.net/jdlSfvudiMSmcRrleeiYjjFO"
      }
    }
  },
  "unsigned": {
    "m.forwarded": {
      "valid": true,
      "event_id": "$BfxMy-oNFOeE0eFt6r-l3h7MtwNVIX0GrructyJq1wA"
    }
  }
}
```

</details>

### Receiving events with `m.forwarded`
When a server receives a message event that has the `m.forwarded` key in its
`content`, the server MUST use the data to validate the signatures, then add a
`m.forwarded` key to the top-level `unsigned` of the event with the validation
information.

#### Validating signatures
To validate a signature, the server should start with the `m.forwarded` object
and modify it as follows:

* If the object is missing any of the required keys, mark it as invalid without
  trying to validate it.
* Remove the `unsigned` key (if present).
* Copy `type` from the top level into `m.forwarded`.
* Make a copy of the top-level `content`, remove `m.forwarded` and put it in
  the `m.forwarded`.
* Using the result object, validate the hashes and signatures on the event, as
  specified in sections 26.2 through 26.4 of the server-server specification:
  https://matrix.org/docs/spec/server_server/r0.1.4#validating-hashes-and-signatures-on-received-events

#### Unsigned `m.forwarded` object
For any message event with `m.fowarded` in the content, the server MUST add or
override the `m.forwarded` key in the `unsigned` object of the event. The key
MUST be an object that contains `valid` and `event_id`.

If the `m.forwarded` object was valid and the signatures were validated, the
`valid` value should be `true`. In any other case (invalid signature, bogus
data, etc), the value should be `false`.

In v1 and v2 rooms, the `event_id` is copied from the `m.forwarded` object in
`content`. In v3 and up, the `event_id` is based on the reference hash that was
validated in the previous section. Copying the event ID in v1/v2 rooms is for
convenience of clients: they only need to look in one place regardless of the
room version.

## Potential issues
* This is not as simple as MSC2723 and requires server support
* Events with bogus data in `m.forwarded` can't be forwarded
* Events that are close to the 64 KiB size limit can't be forwarded. MSC2723
  has the same problem, but this proposal has even more extra data. The amount
  of extra data in both proposals is rather low (<1kb), so this should not be
  a problem in practice

## Alternatives
Instead of an endpoint for sending a forward, the new endpoint could be used to
generate the forward content and leave sending it up to the client with the
normal /send endpoint. However, this is an extra roundtrip for the client and
it is not clear if there are any significant benefits in doing so.

## Unstable prefix
While this MSC is not in a released version of the spec, implementations should
use `net.maunium.msc2730` as a prefix and as a `unstable_features` flag in the
`/versions` endpoint.

* `PUT /_matrix/client/unstable/net.maunium.msc2730/rooms/{roomId}/event/{eventId}/forward/{targetRoomId}/{txnId}`
* `net.maunium.msc2730.forwarded`
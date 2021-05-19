# Mina Networking Layer Refactor

## Summary
[summary]: #summary

NOTE: This is kept abstract from the RPC refactor that is part of moving to bitswap.

TODO:
  - security model
    - limiting max message size
      - work for executing on this may be predicated on bitswap
    - rate limiting decisions
    - identify potential state desyncs between 2 processes
  - consider transport layer abstractions per data stream
  - separate into pre hard fork and post hard fork work
    - post hard fork: message type prefix and dependent pipe splits

## Motivation
[motivation]: #motivation

## Detailed design
[detailed-design]: #detailed-design

- major changes
  - OCaml process is no longer stream-state-aware and is abstracted to level of request/response cycle
    - Go manages dynamic stream management, including opening, closing, and reusing multiplexed streams
  - Go process owns peer management logic
    - seed management
    - trustlist/banlist management
    - peer selection for requests
      - only edge cases involving requesting one specific peer are allowed in OCaml code
        - eg in bootstrap, request scan state from the node that sent us the block we are syncing to

### Libp2p Helper IPC

- pipes > shared memory for this
  - less complex code with less edge cases
  - no need for additional synchronization primitives (synchronization built into pipes)
  - does accrue some additional overhead

- data streams
  - stdin (used only for initialization)
  - stdout (used only for logging)
  - stderr (used only for logging)
  - stats\_in
  - block\_gossip\_in
  - mempool\_gossip\_in
  - response\_in
  - request\_in
  - validation\_out (all validations except request validations, which are bundled with responses)
  - data\_out (responses, broadcasts)

- data stream read priorities
  - daemon
    - block\_gossip\_in
    - response\_in
    - request\_in
    - mempool\_gossip\_in
  - helper (helper has better parallelism, so priorities here aren't as important)
    - validation\_out
    - data\_out

- think about how the ocaml code would structure this
  - limited parallelism per handler type
    - if mixed with max parallelism on all handlers, could be hard to ensure we don't lock on high prio pipes
  - use timestamps for staleness checks?

### Libp2p Helper Protocol

- TODO:
  - keypair generation
  - separate gater config?
  - manually adding peers (seed list)?
    - ^ thinking these 2 should be folded into config


```go
// the following old fields have been completely removed:
//   - `metricsPort` (moving over to push-based stats syncing, where we will sync any metrics we want to expose)
//   - `unsafeNotTrustIp` (only used as a hack in old integration test framework; having it makes p2p code harder to reason about)
//   - `gaterConfig` (moving towards more abstracted interface in which Go manages gating state data)
type Config struct {
  networkId           string      // unique network identifier
  privateKey          string      // libp2p id private key
  stateDirectory      string      // directory to store state in (peerstore and dht will be stored/loaded from here)
  listenOn            []Multiaddr // interfaces we listen on
  externalAddr        Multiaddr   // interface we advertise for other nodes to connect to
  floodGossip         bool        // enables gossip flooding (should only be turned on for protected nodes hidden behind a sentry node)
  directPeers         []Multiaddr // forces the node to maintain connections with peers in this list (typically only used for sentry node setups and other specific networking scenarios; these peers are automatically trustlisted)
  seedPeers           []Multiaddr // list of seed peers to connect to initially (seeds are automatically trustlisted)
  maxConnections      int         // maximum number of connections allowed before the connection manager begins trimming open connections
  validationQueueSize int         // size of the queue of active pending validation messages
  // TODO: peerExchange bool vs minaPeerExchange bool
  //   - peerExchange == enable libp2p's concept of peer exchange in the pubsub options
  //   - minaPeerExchange == write random peers to connecting peers
}

type Stats struct {
  ..................................................
}
```

When the Helper is first initialized by the Daemon, a `init(config Config)` is written once over stdin.

// TODO: identify possible pipes per message type

Daemon -> Helper
  sendRequestToPeer(requestId RequestId, to Peer, msgType MsgType, rawData []byte)
  sendRequests(requestId RequestId, to AbstractPeerGraph, msgType MsgType, rawData []byte)
  sendResponse(requestId RequestId, status ValidationStatus, rawData []byte)
  broadcast(msgType MsgType, rawData []byte)
  validate(validation ValidationHandle, status ValidationStatus)
  // TODO: reconfigure(config Config) (rethink this... should probably limit what can be reconfigured)

Helper -> Daemon
  handleRequest(requestId RequestId, from Peer, rawData []byte)
  handleResponse(requestId RequestId, validation ValidationHandle, rawData []byte)
  handleGossip(from Peer, validation ValidationHandle, rawData []byte)
  stats(stats Stats)

### Validation Control Flow

- gossip and response validation
  - `handle{Gossip,Response}` message is sent to daemon
  - `validate` is sent back to helper
- request validation
  - `handleRequest` message is sent to daemon
  - `sendResponse` message is sent back to helper, which contains both the response and validation state

### Code Organization

- `Libp2p` :: direct low-level access to `libp2p_helper` process management and protocol
- `Mina_net` :: high-level networking interface which defines supported RPCs and exposes networking functionality to the rest of the code (publicly exposed to the rest of the code)

```ocaml
module Mina_net : sig
  module Config : sig
    type t = (* omitted *)
  end

  type t

  val create : Config.t -> t

  ........................................
end
```

## Drawbacks
[drawbacks]: #drawbacks

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Prior art
[prior-art]: #prior-art

## Unresolved questions
[unresolved-questions]: #unresolved-questions

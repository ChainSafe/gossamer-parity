# Smoldot Networking/WebRTC Brain Dump

This is a brain dump of what I learned about networking and specifically WebRTC in smoldot while working on
[#20 (Investigating on how Smoldot uses libp2p-webRTC)](https://github.com/ChainSafe/gossamer-parity/issues/20) and
[#21 (Development environment setup for litep2p-webrtc)](https://github.com/ChainSafe/gossamer-parity/issues/21).

It also includes descriptions of the code I wrote for the "development environment" and the "performance behavior" for use in [lexnv/litep2p-perf](https://github.com/lexnv/litep2p-perf).

## General Networking Overview

Since the main target for running the smoldot light client is the browser (i.e. wasm) and there is no direct access to network I/O in this environment, smoldot implements networking using a
["sans I/O"](https://sans-io.readthedocs.io/) approach. This means the protocol logic is decoupled from actually sending and receiving bytes over the wire.

The central type that facilitates this is [`libp2p::read_write::ReadWrite`](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/read_write.rs#L58).
Types that implement protocol logic have a method called `read_write()` or `substream_read_write()` that takes a `ReadWrite` containing incoming messages and add outgoing messages to
`ReadWrite::write_buffers`.

Some of those types are:

### [`webrtc_framing::WebRtcFraming`](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/connection/webrtc_framing.rs#L30)

Removes the protobuf envelope from [incoming messages](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/connection/webrtc_framing.rs#L151)
and adds it to [outgoing messages](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/connection/webrtc_framing.rs#L250).

### [`collection::Network`](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/collection.rs#L314)

`Network`, aka the coordinator, keeps track of all connections and emits events related to them such as `InboundNegotiated`. The API user needs to poll for these events by calling
`Network::next_event()`.

### [`collection::multi_stream::MultiStreamConnectionTask`](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/collection/multi_stream.rs#L36)

This type keeps track of all substreams of a single connection. An instance of this type is returned by `Network::insert_multi_stream()`. When either side of the connection opens a new
substream, the API user needs to call `MultiStreamConnectionTask::add_substream()`. Whenever data needs to be sent or received on a substream, the API user needs to call
`MultiStreamConnectionTask::substream_read_write()`. In addition, the API user needs to pull messages from the connection to the coordinator and vice versa.

### [`connection::established::multi_stream::MultiStream`](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/connection/established/multi_stream.rs#L37)

State machine of a fully-established connection that, um... also keeps track of its substreams. 🤷 After the handshake, a `MultiStreamConnectionTask` contains an instance of this type and forwards
calls like `add_substream()`, `substream_read_write()` and `desired_outbound_substreams()` to it.

### [`connection::established::substream::Substream`](https://github.com/ChainSafe/smoldot/blob/37482d9a635a17f9e6a77aa245b4684b2f72e750/lib/src/libp2p/connection/established/substream.rs#L180)

Represents a single substream inside of a connection. This type implements multistream-select negotiation, notification, request/response and ping protocols. As far as I know, there is no
mechanism to use custom `Substream` types that implement other protocols (such as the performance behavior) in a `MultiStream`/`MultiStreamConnectionTask`.

## Development Environment Overview

The smoldot "development environment" is a pared-down version of smoldot running in the browser for the purpose of testing various parts of the WebRTC support in isolation. It currently lives
in the directory `lib/examples/webrtc-wasm` on [the branch `haiko-libp2p-webrtc`](https://github.com/ChainSafe/smoldot/tree/haiko-libp2p-webrtc). It consists of the JavaScript side in
[`index.html`](https://github.com/ChainSafe/smoldot/blob/haiko-libp2p-webrtc/lib/examples/webrtc-wasm/index.html) and the Rust/wasm side in
[`lib.rs`](https://github.com/ChainSafe/smoldot/blob/haiko-libp2p-webrtc/lib/examples/webrtc-wasm/src/lib.rs).

When connecting to [the libp2p server in timwu20/webrtc-poc at commit 851e459](https://github.com/timwu20/webrtc-poc/tree/851e4593f96d720ffd5f78c3d8b7b1971b1765ec), the noise handshake,
incoming pings and outgoing pings work.

### How to Run it

First make sure you have cloned and built `timwu20/webrtc-poc` and `ChainSafe/smoldot` at the correct commit/branch.

```
tmp$ git clone https://github.com/timwu20/webrtc-poc.git
Cloning into 'webrtc-poc'...
[...]
Resolving deltas: 100% (16/16), done.

tmp$ cd webrtc-poc

webrtc-poc$ git checkout 851e459
Note: switching to '851e459'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

HEAD is now at 851e459 fix up cargo

webrtc-poc$ cargo build
   Compiling proc-macro2 v1.0.101
   Compiling unicode-ident v1.0.18
   [...]
```

```
tmp$ git clone git@github.com:ChainSafe/smoldot.git
Cloning into 'smoldot'...
[...]
Resolving deltas: 100% (29516/29516), done.

tmp$ cd smoldot

smoldot$ git checkout haiko-libp2p-webrtc
branch 'haiko-libp2p-webrtc' set up to track 'origin/haiko-libp2p-webrtc'.
Switched to a new branch 'haiko-libp2p-webrtc'

smoldot$ cd lib/examples/webrtc-wasm

webrtc-wasm$ wasm-pack build --target web
[INFO]: 🎯  Checking for the Wasm target...
[INFO]: 🌀  Compiling to Wasm...
   Compiling version_check v0.9.5
   Compiling typenum v1.18.0
   [...]
    Finished `release` profile [optimized] target(s) in 19.87s
[INFO]: ⬇️  Installing wasm-bindgen...
[INFO]: Optimizing wasm binaries with `wasm-opt`...
[INFO]: Optional fields missing from Cargo.toml: 'description', 'repository', and 'license'. These are not necessary, but recommended
[INFO]: ✨   Done in 20.51s
[INFO]: 📦   Your wasm pkg is ready to publish at /Users/haiko/code/tmp/smoldot/lib/examples/webrtc-wasm/pkg.
```

#### Manually

Run the server:

```
webrtc-poc$ RUST_LOG=trace cargo run --package webrtc-poc-server --bin webrtc-poc-server
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.15s
     Running `target/debug/webrtc-poc-server`
2025-09-25T05:35:29.907787Z  INFO libp2p_swarm: local_peer_id=12D3KooWPP93A4tYA6eEKX7o8XEuFXjRRiGnF1gsNBxXvZ9j6Zxu
2025-09-25T05:35:29.909416Z  INFO webrtc_poc_server: Listening address=/ip4/192.168.1.126/udp/55442/webrtc-direct/certhash/uEiBl6ZmotWYf3W6u4ukM96IgWBilRx3h-Wo0jpQaVwlVeg
2025-09-25T05:35:29.909622Z  INFO webrtc_poc_server: Peer address: /ip4/192.168.1.126/udp/55442/webrtc-direct/certhash/uEiBl6ZmotWYf3W6u4ukM96IgWBilRx3h-Wo0jpQaVwlVeg/p2p/12D3KooWPP93A4tYA6eEKX7o8XEuFXjRRiGnF1gsNBxXvZ9j6Zxu
```

Run smoldot:

- Run a webserver that serves `index.html` and the wasm blob on port 8082: `python3 -m http.server 8082` (in directory `lib/examples/webrtc-wasm`)
- Navigate to http://localhost:8082/ in Chrome
- Open the Chrome developer console to see logging output
- Copy'n'paste the peer address from the servers output into the text field (in this case `/ip4/192.168.1.126/udp/55442/webrtc-direct/certhash/uEiBl6ZmotWYf3W6u4ukM96IgWBilRx3h-Wo0jpQaVwlVeg/p2p/12D3KooWPP93A4tYA6eEKX7o8XEuFXjRRiGnF1gsNBxXvZ9j6Zxu`)
- Click the button

#### Semi-automated

The tedious task of copying the peer address, pasting it into the text field and clicking the button can be automated with [`smoldot-browser-poke`](https://github.com/haikoschol/smoldot-browser-poke).
This tool reads the peer address from a file whenever it changes and controls Chrome to start the connection. The `webrtc-poc` server needs to be modified to write this file.
This can be done by adding the following line just below where the peer address is printed in line 62 of `server/src/main.rs`:

```Rust
    tokio::fs::write("peer_address.txt", addr.to_string()).await?;
```

In order to use the browser automation you need to run a special build of Chrome called Chrome for Testing, plus a command line tool called `chromedriver`. Both can be downloaded from
https://googlechromelabs.github.io/chrome-for-testing/.

First run Chrome, e.g. on Mac, and open the developer console:

```
$ chrome-mac-arm64/Google\ Chrome\ for\ Testing.app/Contents/MacOS/Google\ Chrome\ for\ Testing  --remote-debugging-port=9222 --user-data-dir="/Users/haiko/Desktop/chromecruft"

DevTools listening on ws://127.0.0.1:9222/devtools/browser/b387e83c-f1c4-4ff1-aea5-86938a640229
[...]
```

Then `chromedriver`:

```
$ ./chromedriver-mac-arm64/chromedriver --port=9515
Starting ChromeDriver 140.0.7339.82 (bc93617e21c39ed4afa6ce1c08554e5aa76d132d-refs/branch-heads/7339_41@{#5}) on port 9515
Only local connections are allowed.
Please see https://chromedriver.chromium.org/security-considerations for suggestions on keeping ChromeDriver safe.
ChromeDriver was started successfully on port 9515.
```

Then `smoldot-browser-poke`:

```
$ cargo run -- ../webrtc-poc/peer_address.txt
   Compiling smoldot-browser-poke v0.1.0 (/Users/haiko/code/polkadot/smoldot-browser-poke)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.15s
     Running `target/debug/smoldot-browser-poke ../webrtc-poc/peer_address.txt`
Watching file: ../webrtc-poc/peer_address.txt
Watcher started. Waiting for file changes... (Press Ctrl+C to exit)
```

Then the webserver in the smoldot repo as above and the modified `webrtc-poc` server as above. When you start the latter you should see the page being loaded and the connection initiated in
Chrome.

## Performance Behavior Implementation Overview

The repository [`lexnv/litep2p-perf`](https://github.com/lexnv/litep2p-perf) contains performance measurements of litep2p vs libp2p using a simple custom protocol. Apart from measuring
performance, this is also useful for testing interoperability between various libp2p WebRTC-direct implementations, because it is much simpler than the protocols a proper light client
or full node uses, but also more complex than just ping. For now the code in the repo is mostly focused on TCP transport, but Tim already opened
[a PR that adds WebRTC transport](https://github.com/lexnv/litep2p-perf/pull/2) for libp2p client and server.

I've implemented the performance behavior based on the smoldot development environment on [the branch `haiko-webrtc-perf`](https://github.com/ChainSafe/smoldot/tree/haiko-webrtc-perf).
[The protocol state machine](https://github.com/ChainSafe/smoldot/blob/haiko-webrtc-perf/lib/examples/webrtc-wasm/src/perf.rs) is modeled after those in the networking types listed
above and uses existing smoldot code for multistream-select negotiation.

### How to Run it

Make sure you have cloned and built [`timwu20/litep2p-perf`](https://github.com/timwu20/litep2p-perf/tree/tim/support-webrtc) on the correct branch:

```
tmp$ git clone https://github.com/timwu20/litep2p-perf
Cloning into 'litep2p-perf'...
[...]
Resolving deltas: 100% (116/116), done.

tmp$ cd litep2p-perf

litep2p-perf$ git checkout tim/support-webrtc
branch 'tim/support-webrtc' set up to track 'origin/tim/support-webrtc'.
Switched to a new branch 'tim/support-webrtc'

litep2p-perf$ cargo build
   Compiling proc-macro2 v1.0.94
   Compiling unicode-ident v1.0.17
   [...]
```

You can use `smoldot-browser-poke` if you add writing the peer address to a file in `litep2p-perf/libp2p/src/main.rs`:

```diff
litep2p-perf$ git diff
diff --git a/libp2p/src/main.rs b/libp2p/src/main.rs
index 5904187..c8978a1 100644
--- a/libp2p/src/main.rs
+++ b/libp2p/src/main.rs
@@ -83,6 +83,16 @@ async fn main() -> Result<(), Box<dyn std::error::Error>> {

             swarm.listen_on(server_opts.listen_address.parse()?)?;

+            let address = loop {
+                if let SwarmEvent::NewListenAddr { address, .. } = swarm.select_next_some().await {
+                    break address;
+                }
+            };
+
+            let addr = address.with(Protocol::P2p(*swarm.local_peer_id()));
+            tracing::info!("Peer address: {addr}");
+            tokio::fs::write("/Users/haiko/code/polkadot/litep2p-perf/peer_address.txt", addr.to_string()).await?;
+
             loop {
                 let event = swarm.next().await;
                 tracing::info!("Event: {:?}", event);
```

Everything else is basically the same as for the development environment.

## Probable Bugs in WebRTC

As far as I remember, I had to make two changes to the smoldot code to make the noise handshake and inbound/outbound pings work. Those could be bugs in smoldot or a misunderstanding on my
part on how to correctly use the API. Both changes are in `lib/src/libp2p/connection/webrtc_framing.rs`.

The first one is changing [this `if` condition](https://github.com/ChainSafe/smoldot/blob/0ab3d2d0ca01d633cd4f650c2782d1fcd0ce2b4b/lib/src/libp2p/connection/webrtc_framing.rs#L97-L102) from

```Rust
if self
    .inner_stream_expected_incoming_bytes
    .map_or(true, |rq_bytes| rq_bytes <= self.receive_buffer.len())
{
    break;
}
```

to

```Rust
if self
    .inner_stream_expected_incoming_bytes
    .map_or(false, |rq_bytes| rq_bytes > 0 && rq_bytes <= self.receive_buffer.len())
{
    break;
}
```

I don't remember exactly how I came up with this change, but it was basically by logging everything and then tracing the control flow through the code to the point were the handshake didn't
advance. It's definitely possible that this is a consequence of how I'm using the `WebRtcFraming` type in the code and not necessarily a bug.

The second one is changing [line 265](https://github.com/ChainSafe/smoldot/blob/0ab3d2d0ca01d633cd4f650c2782d1fcd0ce2b4b/lib/src/libp2p/connection/webrtc_framing.rs#L258-L265) from

```Rust
if self.framing.inner_stream_expected_incoming_bytes.is_none()
    || self.inner_read_write.read_bytes != 0
{
    self.outer_read_write.wake_up_asap();
}

// Updating the timer and reading side of things.
self.outer_read_write.wake_up_after = self.inner_read_write.wake_up_after.clone();
```

to

```Rust
if self.framing.inner_stream_expected_incoming_bytes.is_none()
    || self.inner_read_write.read_bytes != 0
{
    self.outer_read_write.wake_up_asap();
} else {
    self.outer_read_write.wake_up_after = self.inner_read_write.wake_up_after.clone();
}
```

With this one I'm much more inclined to call it a bug because without the `else`, `self.outer_read_write.wake_up_after` is overwritten right after calling
`self.outer_read_write.wake_up_asap()`, making that call pointless.

A third and more nebulous issue is that the handling of flags in the protobuf envelope doesn't seem to work properly. When the handshake is done, 
`MultiStreamConnectionTask::substream_read_write()`
[returns a `SubstreamFate::Reset`](https://github.com/ChainSafe/smoldot/blob/0ab3d2d0ca01d633cd4f650c2782d1fcd0ce2b4b/lib/src/libp2p/collection/multi_stream.rs#L910). However, after the
remote has sent the last message in the handshake flow, it sends another three bytes. Those are the protobuf envelope with the `FIN` flag set and without a message. The receiver is
supposed to respond with a `FIN_ACK` to that and there is
[code for this in `WebRtcFraming::read_write()`](https://github.com/ChainSafe/smoldot/blob/0ab3d2d0ca01d633cd4f650c2782d1fcd0ce2b4b/lib/src/libp2p/connection/webrtc_framing.rs#L140-L145).
But calling `MultiStreamConnectionTask::substream_read_write()` for the handshake substream again causes the wasm to crash. I believe the following log message on the libp2p server side
is related to this incorrect handling of flags: `INFO libp2p_webrtc_utils::stream::drop_listener: Stream dropped without graceful close, sending Reset`.

## Ideas for Next Steps

- Implement a litep2p server analogous to the libp2p one in `timwu20/webrtc-poc` and make the smoldot development environment work with that.

- Add a smoldot client to `lexnv/litep2p-perf` using browser automation.

- Investigate the probable bugs mentioned above and create issues/PRs in smoldot to fix them/discuss them with tomaka.

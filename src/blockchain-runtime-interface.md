# Blockchain runtime interface

The blockchain runtime provides [functions](#queries) to query the current session status and the
mixnodes for the previous/current sessions. It also provides [a function](#registration) to attempt
to register a mixnode for the next session.

## Mixnode type

`Mixnode` instances are passed to the registration function and returned by mixnode queries.

    struct Mixnode {
        kx_public: [u8; 32],
        peer_id: [u8; 32],
        external_addresses: Vec<Vec<u8>>,
    }

`kx_public` is the X25519 public key for the mixnode in the session (the session is implicit).

`peer_id` is the peer ID of the mixnode. It is a raw 32-byte Ed25519 public key.

`external_addresses` is a list of external addresses for the mixnode. Each external address is a
UTF-8 encoded multiaddr.

Note that the peer ID and external addresses for a mixnode are not validated _at all_ by the
blockchain runtime. Nodes are expected to handle invalid peer IDs and addresses gracefully. In
particular, an invalid peer ID or invalid external addresses for one mixnode should not affect a
node's ability to connect and send packets to other mixnodes.

## Queries

    struct SessionStatus {
        current_index: u32,
        phase: u8,
    }

    fn MixnetApi_session_status() -> SessionStatus

`MixnetApi_session_status` returns the index and phase of the current session.

    enum MixnodesErr {
        InsufficientRegistrations { num: u32, min: u32 },
    }

    fn MixnetApi_prev_mixnodes() -> Result<Vec<Mixnode>, MixnodesErr>
    fn MixnetApi_current_mixnodes() -> Result<Vec<Mixnode>, MixnodesErr>

`MixnetApi_prev_mixnodes` returns the mixnodes for the previous session.
`MixnetApi_current_mixnodes` returns the mixnodes for the current session. The order of the
returned mixnodes is important; [routing actions](./sphinx.md#routing-actions) identify mixnodes by
their index in these vectors. These functions can return `Err(InsufficientRegistrations)` if too
few mixnodes were registered for the session (`num`, less than the minimum `min`). The mixnet is
not operational in such sessions, although nodes should still handle traffic for the previous
session in the first 3 phases.

Nodes should always call query functions in the context of the latest finalised block.

## Registration

    fn MixnetApi_maybe_register(session_index: u32, mixnode: Mixnode) -> bool

To register a mixnode for the next session, `MixnetApi_maybe_register` should be called in the
context of every new best block. `session_index` should be the index of the current session, and
`mixnode` should be the mixnode to register for the next session. If `true` is returned, a
registration extrinsic was created; `MixnetApi_maybe_register` should not be called for a few
blocks, to give the extrinsic a chance to get included.

`MixnetApi_maybe_register` may call the host functions `ext_crypto_sr25519_sign_version_1` and
`ext_offchain_submit_transaction_version_1`.

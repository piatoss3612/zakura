# Isolated Regtest Fixture

`scripts/regtest_fixture.py` provides a run-owned Zakura and lightwalletd
fixture for local integration tests. It creates a unique Docker network, node,
lightwalletd container, and lightwalletd volume; publishes only random
loopback ports; records parity and cleanup evidence; and removes only resources
whose IDs and ownership labels still match the run.

The fixture does not install tools, pull images, provide a faucet, or start a
separate THS server. The pinned lightwalletd image is packaged from the THS
lightwalletd build, but the fixture owns and starts that container directly.

## Prerequisites

Before running the smoke test, provide:

- a Python 3 interpreter (the current smoke workflow has been tested with
  Python 3.14; this document does not assert an older minimum version);
- a POSIX host supporting advisory `flock` (the current macOS/Linux workflow);
- a working Docker CLI and daemon;
- the two images pinned by digest in `scripts/regtest_fixture.py`, already
  present locally;
- a local `grpcurl` executable; and
- a proto directory containing `service.proto` and
  `compact_formats.proto` descriptors compatible with the pinned lightwalletd
  image, including its Ironwood fields.

The fixture verifies the local image IDs. It does not pull a missing image or
substitute a different tag. The proto descriptors are caller-supplied inputs;
this repository does not download them for the smoke test.

### Pinned lightwalletd descriptors

The lightwalletd image build recipe pins lightwalletd source commit
`d79cd1100575ff909d70e00d5514a4092df94934`. Supply the corresponding official,
immutable protocol files:

- [`service.proto`](https://raw.githubusercontent.com/zcash/lightwalletd/d79cd1100575ff909d70e00d5514a4092df94934/lightwallet-protocol/walletrpc/service.proto),
  SHA-256 `6fbfa974081fb02366a3118a25764af289bdbb86688978f5ff2bae661dfa346a`;
- [`compact_formats.proto`](https://raw.githubusercontent.com/zcash/lightwalletd/d79cd1100575ff909d70e00d5514a4092df94934/lightwallet-protocol/walletrpc/compact_formats.proto),
  SHA-256 `d8d0c8aaa5ceec7d5dcc188ef25b04011df2ec0254901620fce982fc13aeb32d`.

These files expose the Ironwood `TreeState.ironwoodTree` and
`CompactTx.ironwoodActions` fields required by the pinned protocol. The smoke
test checks `TreeState.ironwoodTree`; it does not inspect compact transaction
actions. The source commit and file hashes document compatible inputs; they
are not a cryptographic attestation that connects the Docker image digest to
that source tree. The CLI does not fetch or verify these files, so the caller
must place them in `--proto-dir`.

## Standalone smoke test

Node, lightwalletd, and lazily started donor ports are random loopback ports
chosen once per owned fixture and explicitly published in Docker. They share
the native coordinator's private per-UID cooperative port locks; the initial
socket reservation is handed off before container creation, while the lock
remains held until owned container absence is proved while the owning process
is alive. Process exit releases kernel locks; failed-cleanup evidence remains
retained and does not prove resource removal. Donor reset reuses its
lease. A bind collision or changed published port fails closed, without
retargeting callers, the lightwalletd shim, or genesis-proof provenance.
Stopping and restarting lightwalletd retains its original endpoint and cache.
Node state uses the pinned image's private writable layer at
`/home/zebra/.cache/zakura`, not an ephemeral database or a host bind mount.
Stopping a container therefore does not delete its on-disk state; removing
the container does. This is not a restart-equivalence or mempool-persistence
guarantee. The node's non-finalized-state backup is periodic, so its retained
files may not include the last in-memory tip.

Run the fixture without Vizor, a Rust wallet test, a node wallet, or an
external faucet:

```sh
python3 scripts/regtest_fixture_smoke.py \
  --grpcurl /absolute/path/to/grpcurl \
  --proto-dir /absolute/path/to/protos \
  --artifacts-dir /absolute/path/to/new-smoke-artifacts \
  --profile zakura-direct-height1 \
  --blocks 2 \
  --timeout 60
```

`--grpcurl` and `--proto-dir` are required. `--artifacts-dir`, when supplied,
must name a new directory beneath an existing directory owned by the current
user; the command will not reuse or overwrite an existing target. If it is
omitted, the command creates a private temporary artifact directory.
`--blocks` accepts 1 through 1000, and `--timeout` accepts a positive finite
value up to 300 seconds for each fixture phase.

The command bootstraps height 1, mines the requested blocks, and verifies exact
node/lightwalletd height, hash, block time, active pool trees, consensus branch,
and peerless-node state. It always attempts ownership-checked cleanup. After
argument parsing succeeds, stdout contains exactly one JSON result. The same
report is written to `smoke-report.json` when artifact persistence succeeds.
If that write fails, stdout records the failure separately in `report_error`
and the command does not report success. Invalid command-line usage is handled
by `argparse` before the JSON report is created.

Exit status is 0 for a passed smoke test, 1 for a runtime, cleanup, or report
persistence failure, 2 for invalid command-line usage, and 130 for
interruption.

The fixed profiles are:

- `zakura-direct-height1` (default): NU6.3 activates at height 1.
- `zakura-direct-activation500`: NU6.2 activates at height 1 and NU6.3 at
  height 500.

To exercise the activation boundary, use
`--profile zakura-direct-activation500 --blocks 499`. The height-1 bootstrap
starts on NU6.2; the final height-500 parity check includes Ironwood and the
NU6.3 consensus branch.

## Evidence and cleanup

### Explicit small networks

Retained failed fixtures also retain their Docker networks. Docker's default
address pools can therefore run out even with no running containers. Pass
`network_subnet="10.42.0.0/28"` to `RegtestFixture`, or
`--network-subnet 10.42.0.0/28` to the smoke CLI, to allocate a small explicit
network without pruning failure evidence or changing daemon configuration.
The CIDR must be canonical RFC1918 IPv4, with a prefix of /29 or shorter to
fit the node, lightwalletd, donor, and gateway. When omitted, the existing
Docker automatic-allocation behavior is unchanged.

Parallel callers must choose non-overlapping subnets; Docker remains the
authoritative allocator and rejects overlaps. A conflict is a startup failure,
not permission to reuse or delete another fixture's network. The start proof
records the requested subnet separately from resource identity. The existing
run labels, original network ID, and ownership-checked cleanup are unchanged.

The artifact directory is private and contains machine-readable evidence such
as:

- `start-proof.json`, including pinned image identities, resource IDs,
  endpoints, profile, bootstrap blocks, and initial parity;
- `mine-<height>.json`, including generated hashes and the synchronized tip;
- `smoke-report.json`, the standalone command result;
- `cleanup-proof.json`, listing exactly removed resources, port-lock release
  or retention evidence, and any cleanup
  errors; and
- `retention-proof.json`, when requested, recording owned container stops and
  retained resources/port locks without authorizing deletion; and
- owned container inspect data and logs captured for startup failure or close.

Cleanup is fail-closed. A resource is removed only after its current ID and
run labels match the fixture's recorded ownership. If ownership changes,
inspection fails, or deletion cannot be proved, cleanup reports the error and
retains the evidence instead of deleting a similarly named resource.
Only an exit-1, empty JSON inspect array with the exact kind/name-specific
Docker daemon absence message is accepted as proof of absence. Missing sockets,
credential-helper failures and generic `not found` messages do not release
locks or authorize cleanup. Other CLI response formats fail closed.

## Library API

Callers that need wallet-specific funding or synchronization can import
`RegtestFixture` and compose the public `start`, `rpc`, `grpc`, `grpc_stream`,
`mine`, `wait_synced`, `close`, and `retain` methods. They remain responsible
for their own wallet processes and keys. A successful workflow should call
`close` and treat `complete: false` as a cleanup failure. Startup failure
automatically attempts ownership-checked rollback of partially created resources.

For a failed scenario whose state is needed for diagnosis, first stop and join
all caller-owned wallet/control writers, then call `retain` instead of `close`.
It stops only the original owned container IDs and independently verifies the
same IDs/labels, no running/paused/restarting process, and PID zero. It does not
remove containers, the network, the lightwalletd volume or artifacts. A stop
failure is reported even when the other owned containers stop successfully.
Node on-disk state remains in its stopped container; volatile mempool state is
not promised. Diagnostics are captured before stopping.

Retention is terminal even on failure: execution and automatic deletion are
disabled, and repeated `retain` calls return the original attempt's evidence
without retrying or rewriting it. The caller must handle later manual recovery
explicitly. Cooperative port locks stay held for the owner's lifetime, even
after a verified stop, to avoid reusing retained endpoints. Process exit still
releases kernel locks; neither a retention report nor process-group completion
proves resource deletion or a passed scenario. The helper is single-owner;
callers must serialize operations rather than concurrently retain and execute.

The controlled-activation profile also exposes
`replace_tip_holding(required_txids)` and
`release_held_transactions(txids)`. The first method constructs a two-block
replacement fork with a lazy, run-owned, peerless donor node and proves that
the pending raw transactions remain unchanged. The second method releases the
fixture's scheduling gate after revalidating the exact held set and tip; it is
not a node RPC that removes or reinserts transactions. Evidence is written to
`reorg-hold-proof.json` and `reorg-release-proof.json`.

`replace_fork_holding(required_txids, fork_height=...)` explicitly replaces a
deeper branch in either fixture profile. The fork parent must be at or after
the profile's activation height, no higher than 10,000, and 1–512 blocks below
the old tip. The replacement contains one more block than the invalidated
branch. Required transactions may originate in the mempool or the invalidated
branch; their exact bytes must survive revalidation. Proofs include every
invalidated branch hash and its raw transaction digests. At most two explicit
holds are allowed, with full release required before the second hold. The
second operation replaces only the run-owned donor, after verifying its
recorded resource identity and ownership labels. Tip-only and explicit modes
cannot be mixed.

Explicit holds are a controlled wallet-test fixture, not a guarantee that
Zakura naturally reinserts every orphaned transaction. After observing the
exact fork parent (bounded to five seconds), the fixture reads the mempool and
restores only missing explicitly required transactions from bytes
captured before invalidation. It submits each missing transaction at most once,
in canonical branch order, and requires the same txid, exact raw bytes,
unconfirmed status, mempool presence, and unchanged peerless fork tip. Rejection,
changed bytes, a different txid, or confirmed state fails closed. No new
transaction, signing, replacement, or expiry adjustment is performed.
The native caller requires orphan-mined transactions. Pre-fork pending children
are captured and observed, not obligatorily resubmitted: rolling back their
denomination parent can remove their shielded anchor. Surviving pending bytes
must still match the capture exactly; required transaction rejection remains
fatal.

The explicit result and `reorg-hold-N.json` include `reintroduction` evidence
with separate `natural_txids` (new natural reinsertion), `resubmitted_txids`,
`preserved_pending_txids` (the actual intersection with the resulting pool),
`invalidated_pending_txids` and their captured raw-byte digests, and submission
counts. An absent pre-fork pending transaction is an observed eviction, not a
claim that its consensus rejection was individually probed. The existing
tip-only API retains its original natural
reinsertion behavior and response schema. An exact-byte restoration must not
be reported as proof of natural node reinsertion.

In explicit mode, `release_held_transactions` accepts only captured held subsets.
Repeating a release is an idempotent unhold and does not increment release
progress or overwrite prior proof. The final release may also name
previously released and mined denominations, as the migration tests require.
An explicit reorg with an empty resulting pool has no gate to release and is
immediately in the released state; no release RPC, counter, or proof is invented.
Until every transaction is released, `mine` extends the adopted chain using
the peerless donor and includes only non-held transactions. Every block proves
the exact permitted transaction bytes, held exclusion, and node/lightwalletd
tip parity. Held transactions remain byte-identical in the main mempool.
Evidence uses `reorg-hold-N.json`, `reorg-release-N-K.json`, and the
`held_safe_blocks` entries in `mine-HEIGHT.json`.

`hold_pending_transactions(required_txids, expiry_height=...)` uses the same
miner without replacing the current branch. It copies the exact main prefix
and tree state to a fresh owned donor, then excludes the registered pending
transactions. The optional expiry height must be a future height within
10,000 blocks. Missing held transactions are rejected before that height;
at or after it, expiry may remove them from the mempool. Changed bytes are
always rejected. Callers must separately prove that the transaction actually
expired, and `pending-hold-proof.json` records the original transaction
digests. Pending holds and reorg operations cannot be combined in one fixture.

The standalone smoke command verifies only tip, tree, branch, and cleanup
behavior. It does not exercise the donor-node reorg path or a wallet's compact
block cache.

## Unit tests

The fixture tests use Python's standard library and mocked subprocesses; they
do not start Docker resources:

```sh
PYTHONDONTWRITEBYTECODE=1 \
  python3 -m unittest \
    scripts/tests/test_regtest_fixture.py \
    scripts/tests/test_regtest_fixture_smoke.py
```

The optional standalone command above is the Docker-backed integration check.
Run it only when the pinned images and matching proto descriptors are already
available locally.

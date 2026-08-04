# Ops Doc: Trading-API SDK Dependency

**No corresponding source file — this describes an EXTERNAL artifact and the
terms on which it is adopted. The module built on it is `trading_api.md`.**

---

## Role

The vendor publishes an official SDK (MIT). It is adopted as a dependency
rather than reimplemented: it already covers token lifecycle, two-tier rate
governance, retry classification, continuation, and WebSocket
connect/reconnect/subscription-restore — the whole transport layer this
system would otherwise have specified itself.

This file records what was adopted, what was changed, why each change could
not be avoided, and what the adoption costs.

---

## Adoption Form

**Vendored.** A full clone lives at `vendor/dbsec-open-api/` at repository
root — not under `src/`, which would break the `docs/` ↔ `src/`
correspondence, since vendor code has no counterpart spec.

Full clone rather than a minimal subset: keeping the upstream tree intact is
what makes our modifications visible as a diff against it. A trimmed copy
saves space and loses the only mechanism that shows what we changed.

**Why vendored rather than referenced externally.** The decision follows
from modification being necessary (below), not from convenience:

- A modification made to an external clone is recorded nowhere and is
  reproducible only on the machine that made it
- The vendor also ships an MCP server that runs `git fetch` +
  `git reset --hard` against its own checkout on startup, preserving only
  untracked files. A shared external clone would silently lose every
  tracked-file modification. Vendoring makes that conflict structurally
  impossible: the MCP server keeps its own checkout and never touches ours
- Upstream states it may change without notice; pinning the exact state the
  system was designed against is worth more here than automatic tracking

**Installation.** The vendored copy is installed editable by the USER, never
by an implementation agent — see `CLAUDE.md`. Editable installation is also
what makes the SDK's two relative paths resolve: both its endpoint rate-limit
matrix and its token cache are located relative to the package's parent
directory, which is the vendored root.

**Configuration.** The SDK's `Config` takes a path argument, so its
configuration file does not have to live in the vendor tree. This project
owns the location and injects the path; credentials never sit inside the
vendored copy.

**Token cache.** The SDK writes its access-token cache inside the vendored
directory. This is accepted rather than redirected — the path is hardcoded
relative to the package, and upstream's own `.gitignore` already excludes the
file, so it is never committed. "Read-only vendor material" describes
`api_doc/`; this copy permits both our fork edits and the SDK's own writes.

---

## Fork Surface

**Two modifications. The standing rule is that anything solvable in
`trading_api.md` is not modified here**, because every modification is
permanent re-sync debt (below).

**1. Control-response callback.** The SDK's message handler returns on any
response carrying `rsp_cd`/`rsp_msg` before reaching the registered callback,
logging a warning instead. Subscription success and subscription FAILURE are
therefore both invisible above the SDK: a server-side rejection (session cap
exceeded, unknown symbol) leaves the SDK's internal subscription list
recording a subscription the server refused, and a position can open with no
price stream watching it while the count still looks correct.

Three alternatives were rejected before modifying:

- Attaching a log handler and parsing the warning text — log wording is not
  a contract and upstream changes it freely
- Inferring failure from silence — indistinguishable from a low-liquidity
  ticker, and this universe is made of those
- Accepting it — the affected guarantee is a position's tp/sl coverage

**2. WebSocket connect pacer.** The SDK paces connections with a
minimum-interval limiter derived from the per-minute connection cap. Its own
documented reason for preferring minimum-interval over a window is the
server's one-second counting window — an argument that holds for per-second
limits and cannot apply to a per-minute one. Measurement confirms an opening
burst is accepted. The reconnect schedule is ours to own
(`live_mode_runner.md`), and the pacer is instance-internal to the SDK's
WebSocket object, so it cannot be bypassed from above.

**Explicitly NOT modified**, though each was considered:

- Callback exceptions being swallowed — the callback is our code and owns its
  own error path
- Rate-limit registry validation — `get_rate_limits()` is already public and
  can be checked at startup from our side
- Order-type signature limits — the client exposes a generic request path,
  so a typed method's narrower argument list is not binding

---

## Accepted Costs

**Re-sync debt.** Modification makes this a fork. Upstream changes must be
merged by hand, and nothing detects a forgotten merge. This is recorded in
the same manner as the `db_schema.md` ↔ `init_db.py` transcription
obligation (`init_db.md`): a known, manual, undetected-if-skipped duty
accepted deliberately in exchange for something worth more.

**Repository size.** Third-party source enters this repository's history;
roughly half of it belongs to market groups this system does not trade.

**Search noise.** Repository-wide searches match vendor code. The vendored
directory is an explicit exception to the `docs/` ↔ `src/` mirroring rule and
has no counterpart spec.

---

## Constraints

- The agent does not install anything, here or elsewhere — `CLAUDE.md`
- No module other than `trading_api.md`'s implementation imports the SDK
- A new fork modification requires showing that the problem cannot be solved
  in `trading_api.md`; convenience is not sufficient
- Multi-account operation would require a THIRD modification and is not in
  scope: the SDK's token cache is a single file holding a single token whose
  mode it validates by content, so two concurrent clients invalidate each
  other's cache in a loop against a one-per-minute issuance limit. This is
  not fixable above the SDK, since the cache path is hardcoded relative to
  the package

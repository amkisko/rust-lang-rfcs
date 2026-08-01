- Feature Name: `cargo_registry_mutation_idempotency`
- Start Date: 2026-08-01
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Cargo Issue: [rust-lang/cargo#0000](https://github.com/rust-lang/cargo/issues/0000)
- crates.io issue: [rust-lang/crates.io#0000](https://github.com/rust-lang/crates.io/issues/0000)

## Summary
[summary]: #summary

Extend [Cargo registry mutation authorization] with the
`idempotent-final` extension. A registry activating this extension turns the
authorization record's `mutation_id` into an execution identity: matching
retries cannot perform the logical mutation twice, and a committed terminal
response remains replayable after response loss.

This proposal defines request claiming, receive leases, execution recovery,
terminal response storage, and Cargo's bounded final-request retry. It does not
change how a mutation becomes authorized.

The core is complete without this extension. This proposal depends on the core
record and readiness semantics and can be reviewed, stabilized,
and deployed independently of `loopback-callback`.

## Motivation
[motivation]: #motivation

An authorized publish can be large. If the connection fails during upload,
Cargo cannot know whether the registry read no bytes, received the full body,
committed the publication, or committed and lost only the response.

Ordinary endpoint uniqueness can prevent a duplicate crate version, but it
does not reproduce the original response or cover owner changes, notifications,
webhooks, object storage, and other effects. Blindly retrying can repeat a
logical mutation; never retrying leaves avoidable ambiguity.

The core authorization protocol already creates an unpredictable mutation id
bound to exact request bytes and a primary credential. A registry can use that
identity to serialize execution and replay the result, but only endpoints with
suitable transactions or endpoint-specific deduplication can make the required
crash guarantee. Keeping that guarantee in an explicitly activated extension prevents
core-only registries from accidentally claiming exactly-once behavior.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Cargo that implements the extension includes `idempotent-final` in
`requested_extensions` and adds the exact ordinary HTTP method, origin-form
target, and media type to preflight. It relies on the guarantee only when the
registry echoes `idempotent-final` in `active_extensions`. After readiness it
sends the ordinary request under `Cargo-Mutation-Id` as in the core protocol.

A normal request is sent once. Cargo retries only after an interrupted or
response-ambiguous transport, or a response explicitly classified as
nonterminal. The retry uses byte-identical content and the same mutation id.

For example, suppose a publish upload reaches the registry and commits, but
the connection closes before Cargo receives the response. Cargo can retry once
under the same mutation id. It receives the stored success response, while the
registry does not create the version, index entry, audit event, or notification
a second time. If the first connection ended before the complete body matched,
the retry can instead claim the still-live record and perform the mutation.

The registry serializes attempts. If the first attempt committed, the retry
returns its stored response without repeating the effect. If the first attempt
was incomplete, a matching retry can continue within a bounded receive lease.
If the operation cannot provide these guarantees, the registry omits
`idempotent-final` while retaining core mutation authorization.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Selection and descriptor extension

Cargo includes `idempotent-final` in the core preflight's
`requested_extensions` when it can replay the exact final request. A registry
that implements the extension activates it for applicable operations, stores
it with the record, and echoes it in `active_extensions`. If the echo omits the
extension, Cargo uses core's single-use final-request behavior and does not
retry. A registry must also accept a preflight that does not request the
extension; that record uses core's `ready` to `consumed` transition.

When requesting this extension Cargo additionally sends:

```json
{
  "method": "PUT",
  "request_target": "/api/v1/crates/new",
  "content_type": "application/octet-stream"
}
```

`request_target` is the final URL's origin-form path and optional query with
exact percent encoding. It includes any path prefix from the registry API base.
Version 1 operations use no query; registries reject one.

Cargo sends `method` and `request_target` as declared request facts. The
registry derives the expected method and target from its own endpoint routing
and compares them; it never treats the client-declared values as authority.

`content_type` is `application/octet-stream` for publish,
`application/json` for owners, and `null` for bodyless yank and unyank.
For bodyless operations a registry may treat an omitted `content_type` as
equivalent to `null`; it still derives and binds the absence of a media type.
Parameters are prohibited. `Content-Encoding` remains prohibited. Transfer
framing is not part of the descriptor.

The registry derives method, request target, and media type independently from
the operation and its own API base. Cargo's values are consistency assertions
needed for safe replay, not authority for selecting an endpoint.

The registry's mutation fingerprint covers protocol version, validated method,
exact target, media type, absence of content encoding, raw body digest and
size, semantic operation fields, and publish archive digest and size. Primary
credential value, `preflight_id`, `allow_pending`, and optional extension
delivery metadata are excluded. Credential binding is stored separately.

Cargo constructs the replayable request body once. It can use memory or
file-backed storage and can compute digests incrementally; this proposal does
not require buffering an archive in RAM.

Every ready preflight or poll response for an active idempotent record also
contains:

```json
{ "receive_lease_secs": 300 }
```

`receive_lease_secs` is a positive whole number no greater than 3,600. Before
claim it is the minimum duration of the receive lease the registry will create;
after claim it is a conservative lower bound on the remaining lease. It is a
client retry bound, not mutation authority. Cargo uses the first value it
observes at readiness and starts a monotonic attempt deadline of that duration
immediately before beginning its first final request, which is no later than the
server's claim time. A later response never extends Cargo's deadline. Cargo
rejects a ready response that omits or invalidates this field while
`idempotent-final` is active.

### Lifecycle

An idempotent record has this normative lifecycle:

```text
pending --verification--> ready --request claim--> receiving
   |                         |                         |
   +--> denied               +--> expired              +--> expired
   +--> expired                                        |
                                                      v
                      terminal <--logical commit-- executing
```

- `pending` has a challenge deadline.
- `ready` has a grant deadline by which a first valid request must begin.
- `receiving` has a bounded receive lease established by the first claim.
- `executing` begins after the complete request and parsed operation match.
- `terminal` stores the committed effect's replayable response.
- `denied` and `expired` never permit execution.

This lifecycle replaces the core-only atomic `ready` to `consumed` transition.
It does not weaken the core rule that no more than one logical mutation can
execute under a record.

Challenge, grant, receive, and terminal-retention deadlines are separate
registry timestamps. The receive lease must accommodate the registry's maximum
request size and at least one realistic transport retry. Its exact timestamp is
registry state; only the conservative `receive_lease_secs` bound is exposed to
Cargo.

After claim, matching retries remain acceptable until the receive lease ends
even if the earlier grant deadline passes. An incomplete request does not enter
`executing`. Lease expiry before a complete match transitions the record to
`expired` without an effect or terminal response.

For deterministic status if a client polls after claim, the registry projects
`receiving`, `executing`, and retained `terminal` as `ready` while the final
endpoint can accept a matching retry. Such a response includes
`receive_lease_secs` based on the remaining receive or retention window;
`grant_expires_in` is a conservative window in which the endpoint currently
expects to accept a request and does not extend another deadline.

### Claim and validation

A final endpoint uses this order:

1. Parse and authenticate `Cargo-Mutation-Id` without reading an
   attacker-selected body.
2. Resolve the record and check primary credential binding and ordinary scope.
3. Check content encoding, declared `Content-Length`, media type, method, and
   exact path/query against the descriptor.
4. If the record is `ready` and its grant is live, atomically transition it to
   `receiving` and establish the receive lease.
5. Read no more than the declared descriptor size, require a complete body, and
   compare its raw digest.
6. Parse the ordinary operation, compare every semantic field, and transition
   a complete exact match to `executing`. Publish validation includes archive
   digest and size.

An unknown id, another credential, a non-ready state, an expired grant, or a
step 3 mismatch fails before claim. A body or semantic mismatch after claim
performs no effect and leaves a legitimate matching retry usable until the
receive lease ends.

Requests sharing a mutation id are serialized. At most one logical execution
is active. Concurrent matching requests wait for that execution or receive a
documented nonterminal retry response.

### Terminal outcomes

The mutation effect and terminal outcome are one logical commit. An endpoint
implements this with one database transaction or endpoint-specific idempotency
for external effects:

| Failure point | Required recovery |
| --- | --- |
| Before complete match | No effect or terminal outcome; matching retry remains possible within its lease. |
| After match, before logical commit | Effect and outcome roll back together, or retry resumes the same logical execution. |
| After commit, before client receives response | Effect is not repeated and the stored response is replayed. |

An external effect such as object storage, index update, notification, or
webhook is deduplicated under the mutation id or driven from a transactional
outbox. Existing crate-version uniqueness is additional defense, not a
substitute for recording the outcome.

A normal `2xx` or deterministic application `4xx` after complete validation
can be terminal. A response is not terminal when repeating admission could
legitimately change it.

These remain nonterminal:

- credential, scope, grant, binding, metadata, body, parsing, and size failures;
- incomplete or oversized requests;
- `408`, `425`, `429`, `500`, `502`, `503`, and `504`;
- failures of admission controls or transient dependencies.

Once `terminal` exists, every matching request returns the stored status, body,
and `Content-Type`. Registries do not store or replay hop-by-hop fields,
`Set-Cookie`, `Date`, authentication challenges, or dynamic rate-limit headers.
Terminal responses are retained for at least 24 hours from logical commit,
independently of earlier deadlines.

A fresh command uses a fresh `preflight_id` and therefore does not select a
previous terminal outcome merely because its mutation bytes match.

### Cargo retry behavior

Cargo retains an exact replayable body until the flow ends. Immediately before
its first attempt it establishes a monotonic attempt deadline using
`receive_lease_secs`. It performs at most a small implementation-defined number
of automatic attempts before that deadline. The registry remains authoritative:
if no attempt claimed the record before grant expiry, a later retry can still
fail even though Cargo's local receive window remains.

No complete response, an interrupted transport, or a complete `408`, `425`,
`429`, `500`, `502`, `503`, or `504` can trigger a retry with the same
mutation id. Cargo uses bounded backoff with jitter and honors valid
`Retry-After` on `429` or `503` when it fits the remaining attempt window.
It does not automatically retry another complete response.

An ordinary protected mutation still uploads once. A transport retry can send
bytes again because Cargo cannot know how many arrived; the registry ensures
the logical effect commits at most once.

Cancellation before Cargo transitions to `sending` initiates no attempt. Once
any attempt begins, interruption has ordinary in-flight ambiguity; Cargo does
not claim that no mutation was sent. A later invocation uses a new preflight
unless an independent future recovery protocol persists the old request and
capabilities.

### Security and limits

The mutation id is not sufficient authority: every attempt still requires the
bound primary credential, exact request, and a ready, receiving, or matching
terminal record.

Registries separately limit concurrent claims, receive bytes, execution time,
and retained terminal storage. Terminal bodies can be capped to the ordinary
API's bounded response size.

Implementations must monitor `executing` records and define endpoint-specific
recovery after process or database failure. Blindly changing an old
`executing` record back to `ready` is conforming only when repeating the
logical execution is safe.

## Drawbacks
[drawbacks]: #drawbacks

- Registries retain terminal responses and serialize requests per mutation id.
- Large requests need a receive lease and replayable client representation.
- Each protected endpoint needs transactional or explicitly deduplicated side
  effects; generic HTTP middleware alone is insufficient.
- At-most-once logical execution can require an outbox or recovery worker for
  non-database effects.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

HTTP method idempotency is insufficient: publish is a PUT in the Cargo API but
its processing can trigger multiple effects, and owner changes can have
notifications or audit events beyond the row update.

Storing only a completed mutation id prevents duplicate effects but cannot
resolve a crash during execution. The explicit `executing` state makes recovery
an endpoint responsibility and prevents middleware from claiming a guarantee
it cannot provide.

An idempotency key independent from the authorization record would add another
identity and binding. Reusing `mutation_id` keeps one logical invocation
coherent while the primary credential and exact fingerprint remain mandatory.

This is an independent extension because authorization remains valuable for a
registry whose endpoints cannot yet satisfy crash-safe replay.

## Prior art
[prior-art]: #prior-art

- Payment and infrastructure APIs commonly accept an idempotency key and store
  the first completed response. This proposal additionally defines an explicit
  receive and execution lifecycle because storing only completed keys does not
  resolve a crash during execution.
- HTTP distinguishes safe and idempotent methods, but those properties describe
  the intended semantics of a method rather than transactional deduplication of
  all endpoint side effects ([RFC 9110]).
- Transactional outboxes and operation-keyed consumers are established ways to
  make database state and asynchronous side effects one recoverable logical
  commit. This proposal requires the property without prescribing one storage
  architecture.

[RFC 9110]: https://www.rfc-editor.org/rfc/rfc9110.html

## Appendix: conformance cases

1. Concurrent matching requests cause at most one logical execution.
2. A mismatched request cannot claim or poison the record.
3. An incomplete upload can retry within its receive lease and performs no
   effect before complete validation.
4. Grant expiry after claim does not invalidate a matching retry within the
   receive lease.
5. Receive-lease expiry before a complete body produces `expired` and no
   terminal response.
6. A crash before logical commit rolls back or resumes without duplicate
   effects.
7. A crash after commit replays the stored response without repeating effects.
8. Terminal retention lasts 24 hours from commit, not challenge expiry.
9. Transient and credential failures remain nonterminal.
10. A deterministic application `4xx` after exact validation can be replayed.
11. Content type, encoding, target, length, raw digest, archive, or parsed-field
    alteration prevents execution.
12. Cargo retries only the defined ambiguous or transient outcomes using the
    same bytes and mutation id.
13. Notifications, audit entries, indexes, object writes, and webhooks commit
    transactionally or deduplicate under the mutation id.
14. A core-only client can request no extensions and receives core single-use
    behavior from a registry that also implements this extension.
15. Cargo retries only when `idempotent-final` is echoed as active and stops at
    its monotonic `receive_lease_secs` deadline.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should this extension stabilize with the core protocol or remain experimental
  until every protected endpoint has exercised crash recovery and terminal
  replay in production-like failure tests?
- Is 24 hours the right minimum terminal-response retention for registry
  operators, or should the protocol permit a shorter advertised retention once
  Cargo does not persist mutation ids across invocations?

## Future possibilities
[future-possibilities]: #future-possibilities

A future Cargo recovery protocol could persist an authorized record and
replayable request across process restarts. This proposal intentionally keeps
recovery within one invocation.

[Cargo registry mutation authorization]: 0000-cargo-registry-mutation-authorization.md

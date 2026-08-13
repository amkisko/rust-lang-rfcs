- Feature Name: `cargo_registry_mutation_authorization`
- Start Date: 2026-07-31
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Cargo Issue: [rust-lang/cargo#0000](https://github.com/rust-lang/cargo/issues/0000)
- crates.io issue: [rust-lang/crates.io#0000](https://github.com/rust-lang/crates.io/issues/0000)

## Summary
[summary]: #summary

Define a Cargo registry protocol that authorizes an exact mutation after
ordinary credential authentication. Cargo sends a compact preflight descriptor,
waits while the registry applies its chosen verification policy, and sends the
ordinary mutation only after the registry reports `ready`.

This RFC defines a complete poll-based workflow, including client modes and
safe display of registry instructions. Two companion proposals add independent
extensions:

| Proposal | Owns | Extension name |
| --- | --- | --- |
| This RFC | Preflight, poll, grant, binding, final request, modes, and display | Core protocol |
| [Idempotent mutation execution] | Claim, receive lease, crash recovery, and terminal replay | `idempotent-final` |
| [Loopback wake-up] | Local listener, callback delivery, and verification-page isolation | `loopback-callback` |

Core alone is complete for poll-only use. Companions depend on core and never
redefine how a record becomes `ready`.

The proposals preserve one security boundary: only the registry's server-side
grant makes a record ready. Neither displayed instructions, polling capability,
nor callback delivery authorizes a mutation.

## Motivation
[motivation]: #motivation

Cargo commonly authenticates registry mutations with a long-lived API token.
Scopes, crate restrictions, and credential providers reduce how often that
token is exposed. Website MFA does not protect a separately issued Cargo
credential.

Consider a maintainer who uses the same publish credential from a workstation
for months. Malware, a copied credentials file, or an accidentally retained CI
or inference log can give an attacker that credential, and enough of the
session that the crate, version, and request can be reconstructed. Token scopes
may limit the attacker to one crate, but within that scope the attacker can
still publish, yank, unyank, or change owners immediately. With this protocol
enabled for that account or crate, the registry treats the token as the first
factor and refuses the exact mutation until a fresh, descriptor-bound grant
exists; for example, that grant is created by a passkey or equivalent on the
verification page. Remote use of the stolen token then stops while the owner
still controls their authenticator.

There is a second failure mode after verification: a client or intermediary
must not be able to replace the approved archive, owner list, or yank direction
with a different mutation. The grant therefore covers the exact semantic
operation and request bytes.

Existing Cargo authentication RFCs improve token storage, authentication, and
replay resistance, but cannot require fresh authorization for one exact
publish, yank, unyank, or owner change ([RFC 2730], [RFC 2947], [RFC 3139],
[RFC 3231], [RFC 3981]).
A registry can reject an ordinary mutation with prose instructions, but Cargo
cannot safely determine whether to wait, when to continue, or whether a
non-interactive credential is exempt.

Preflight lets the registry evaluate policy before Cargo uploads a package. It
also gives the registry the exact semantic operation and request digest that a
later grant will cover. The registry controls verification: passkeys, an
administrator's approval, an SSH workflow, or an out-of-band mechanism require
no Cargo-specific factor protocol.

Malware active before preflight can change the request that Cargo subsequently
hashes and describes. Mutation authorization limits unattended use of a stolen
credential and substitution after preflight.

[RFC 2730]: https://rust-lang.github.io/rfcs/2730-cargo-token-from-process.html
[RFC 2947]: https://rust-lang.github.io/rfcs/2947-crates-io-token-scopes.html
[RFC 3139]: https://rust-lang.github.io/rfcs/3139-cargo-alternative-registry-auth.html
[RFC 3231]: https://rust-lang.github.io/rfcs/3231-cargo-asymmetric-tokens.html
[RFC 3981]: https://github.com/rust-lang/rfcs/pull/3981

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A protected publish has three phases:

1. Cargo packages the crate once and computes the ordinary request and archive
   digests.
2. Cargo preflights those facts. The registry either returns `ready`, reports
   that interaction is required, or creates a short-lived pending record.
3. After the record is `ready`, Cargo obtains an ordinary credential again and
   sends the unchanged publish request with `Cargo-Mutation-Id`.

From the maintainer's perspective, the extra step occurs between packaging and
upload. They inspect the registry's verification page, confirm the crate and
version, and optionally compare the displayed archive digest with one obtained
from their build environment. Closing the page or pressing Ctrl-C before Cargo
starts the final request publishes nothing; the short-lived record eventually
expires.

For a pending operation Cargo displays registry instructions and polls:

```console
$ cargo publish
   Packaging example v1.2.3 (/work/example)
       Note Instructions from registry https://crates.io:
            Additional authorization is required.
            https://crates.io/verify/mut_0123456789abcdefghijkl
    Waiting up to 5 minutes; press Ctrl-C to cancel
       Note Registry authorization ready; continuing
   Uploading example v1.2.3 (/work/example)
```

`ready` authorizes Cargo to attempt the mutation. Only the ordinary endpoint's
response reports the operation result.

The same workflow applies to yank, unyank, and owner changes. The verification
surface obtains a server-generated summary from the stored descriptor. For
publish it should show the archive SHA-256 so a user with an independently
obtained digest can compare it. Plain WebAuthn authenticates the user to the
registry; it does not cryptographically sign arbitrary transaction text shown
by the page.

Automatic mode waits only in an interactive terminal outside CI. With
non-terminal standard input or `CI=true`/`CI=1`, Cargo still preflights with
`allow_pending: false`. An exempt credential can proceed immediately; otherwise
Cargo reports that interaction is required without creating an abandoned
record. Explicit `poll` opts into waiting in CI or over SSH. `disabled` skips
the extension and sends the ordinary request, which a protected endpoint can
reject.

The loopback companion adds `loopback` and lets interactive `auto` use a
callback as a polling accelerator. The execution companion lets Cargo retry an
ambiguous final request under the same mutation id.

Registry operators can deploy the protocol before requiring it. During an
opt-in phase, preflight can return `ready` for exempt credentials while the
ordinary endpoints continue accepting older clients for unprotected accounts.
Enforcement starts only when a protected ordinary endpoint rejects requests
without a matching ready record. That transition must name the minimum Cargo
version and provide a recovery path for users who cannot complete the chosen
verification policy.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Discovery and extension negotiation

The core has these client-visible states:

| State | Meaning |
| --- | --- |
| `pending` | Registry authorization has not completed. |
| `ready` | A server-side grant permits the bound final request. |
| `denied` | Registry policy or an authorized verifier refused it. |
| `expired` | The challenge or grant deadline elapsed. |

Cargo discovers support by sending preflight for every version 1 mutation it
implements unless authorization is disabled. No index configuration advertises
protocol versions, operations, or extensions. The authenticated preflight is
both discovery and the registry's per-request policy decision.

A definitive `404 Not Found` from the preflight endpoint means that the
registry does not implement mutation authorization. Cargo then sends the
ordinary mutation unless an explicit mode requires an extension. Cargo does
not fall back after a transport failure, any
other status (including `401`, `403`, and `5xx`), an oversized body, or a
malformed response. This narrow fallback prevents an intermediary or server
failure from silently bypassing authorization. A registry protecting any final
endpoint must also implement preflight; final-endpoint enforcement remains the
security boundary.

Version 1 defines publish, yank, unyank, and owner changes. Each preflight
request and recognized preflight response carries `protocol_version: 1`;
poll responses are scoped by the versioned record and omit it. There is no
version-list negotiation. A different or missing preflight response version is
a protocol error.

Extension use is negotiated per preflight. Cargo requests distinct extensions
it implements. The registry activates the supported subset applicable to that
record, stores it, and echoes that subset as `active_extensions`. Cargo rejects
duplicate, unrequested, or unsupported active names and relies only on names
the response activates. Omission is safe: an explicit mode that needs an
extension fails, while an automatic mode may use core behavior. An empty
request or response set is the complete core protocol.

`idempotent-final` is defined by the execution companion; `loopback-callback`
is defined by the loopback companion. Unknown requested extension names are ignored, allowing independent client and
registry deployment.

### Client modes

The common `--mutation-authorization-mode` option has these core values:

| Value | Behavior |
| --- | --- |
| `auto` | Wait by polling only when locally interactive and outside CI. |
| `poll` | Wait by polling, including without a terminal or in CI. |
| `disabled` | Skip preflight and use the ordinary endpoint. |

The value can also come from
`CARGO_REGISTRY_MUTATION_AUTHORIZATION_MODE`,
`registries.<name>.mutation-authorization-mode`, or
`registry.mutation-authorization-mode` for crates.io. Precedence is command
option, environment, selected-registry configuration, then `auto`.

Interactive `auto` and explicit `poll` set `allow_pending` to true.
Non-interactive `auto` sets it to false, allowing an exempt credential to
proceed without creating an abandoned record. `disabled` sends no preflight.
The loopback companion adds an explicit `loopback` value and can optimize
interactive `auto`; it does not alter the core polling fallback.

### Preflight

Before a version 1 mutation Cargo sends an authenticated `POST` to
`/api/v1/auth/mutation-challenges`. Paths in this RFC are appended to the
`api` base URL from `config.json` without discarding a base path prefix, as in
the [Cargo Registry Web API].

Preflight uses JSON, requests JSON, follows no redirect, and sends the intended
operation's normal credential-provider operation.

A publish descriptor is:

```json
{
  "protocol_version": 1,
  "preflight_id": "pf_0123456789abcdefghijklmnopqr",
  "allow_pending": true,
  "requested_extensions": [],
  "operation": "publish",
  "crate": "example",
  "version": "1.2.3",
  "request_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "request_size": 124678,
  "archive_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "archive_size": 123456
}
```

`request_sha256` hashes the exact request-body content bytes, excluding HTTP
transfer framing. Version 1 prohibits `Content-Encoding`, so these are also the
bytes presented to the endpoint parser. `request_size` is their length. The
archive fields hash and size the package archive within the ordinary publish
body.

Yank and unyank descriptors contain `operation`, `crate`, `version`, and the
digest and size of the empty body. Owner descriptors contain `operation:
"owners"`, `crate`, `direction` (`add` or `remove`), the complete ordered
`owners` list, and the exact JSON body digest and size.

The registry validates every semantic field, derives the expected ordinary
method, endpoint, and media type from the operation and its own API base, and
stores those facts with the descriptor. The parsed final request remains
authoritative. It must match both the semantic descriptor and raw digest.

Cargo generates a fresh `preflight_id` for each logical invocation. It meets
the [protocol limits], is not a credential, and is reused only after an
interrupted or response-ambiguous preflight.

The registry atomically maps credential binding identity plus `preflight_id` to
one record. A retry returns the same record. Reuse with a different descriptor
or `allow_pending` fails without changing it. Different preflight ids create
different logical records even when descriptors match. The mapping remains
while any associated state can be used or observed.

`allow_pending` is required and is derived from the selected client mode.

`requested_extensions` is a required array of distinct names conforming to the
[protocol limits]. Every name Cargo sends must be implemented by Cargo. An
empty array requests the complete core behavior. The registry ignores names it
does not implement and activates the supported applicable subset. A recognized
extension field is invalid when its extension name is absent from
`requested_extensions`. If the name is requested but the registry does not
activate it, the registry ignores that extension's fields and need not store
them. When the registry activates the extension, it validates and binds its
fields. A recognized extension field set to JSON `null` is present for these
rules unless that extension explicitly assigns semantics to `null`. Unknown
JSON fields remain ignored under the ordinary
forward-compatibility rule.

Cargo reproduces the complete request when retrying a `preflight_id`. The
registry binds the protocol version, complete core descriptor, `allow_pending`,
active extension set, and recognized extension fields to that id. Ignored
unknown extension names and fields need not be stored. Extension delivery
metadata need not become part of the semantic mutation fingerprint, but it
cannot be changed by a preflight retry.

### Preflight results

All recognized preflight responses are JSON and include `Cache-Control:
no-store`.

If policy requires no additional interaction, the registry creates a ready
record and returns `200 OK`:

```json
{
  "status": "ready",
  "protocol_version": 1,
  "mutation_id": "mut_0123456789abcdefghijkl",
  "active_extensions": [],
  "grant_expires_in": 300
}
```

If interaction is required and `allow_pending` is true, it returns
`202 Accepted`:

```json
{
  "status": "pending",
  "protocol_version": 1,
  "mutation_id": "mut_0123456789abcdefghijkl",
  "active_extensions": [],
  "detail": "Additional authorization is required. Visit https://registry.example/verify/mut_0123456789abcdefghijkl.",
  "poll_url": "https://registry.example/api/v1/auth/mutation-challenges/poll/poll_0123456789abcdefghijkl",
  "challenge_expires_in": 300,
  "recommended_poll_interval_secs": 5
}
```

Required ready fields are `status`, `protocol_version`, `mutation_id`,
`active_extensions`, and `grant_expires_in`. Required pending fields are
`status`, `protocol_version`, `mutation_id`, `active_extensions`, `detail`,
`poll_url`, and `challenge_expires_in`. `active_extensions` must be a subset of
`requested_extensions`; Cargo rejects a duplicate, unrequested, or unsupported
active name. `detail` is complete untrusted interaction text. A recognized
field defined only for another status or inactive extension must be absent;
JSON `null` does not satisfy absence. Unknown response fields are ignored.

If interaction is required but `allow_pending` is false, the registry creates
no mutation-authorization record or capability and returns `403 Forbidden`:

```json
{
  "status": "interaction_required",
  "protocol_version": 1,
  "active_extensions": [],
  "detail": "This operation requires registry authorization."
}
```

Cargo recognizes this only when `mutation_id` and `poll_url` are absent.
`status`, `protocol_version`, `active_extensions`, and `detail` are required.
Invalid credentials or insufficient ordinary operation scope use normal `401`
or `403` registry errors.

A retry for an existing `denied` or `expired` record returns `200` with that
status, version, mutation id, active extension set, and optional detail. A retry
for a pending or ready record returns its current result in the corresponding
schema. A retry for a core-only record already consumed by a final request
returns `409 Conflict` and cannot make the record ready again.
`interaction_required` is a preflight JSON status.

`mutation_id` meets the [protocol limits]. It identifies the record but cannot
authorize a mutation without the bound primary credential and a ready
server-side grant.

Expiry and polling values conform to the [protocol limits]. Cargo rejects an
invalid expiry. The polling interval is advisory;
Cargo defaults or clamps it as specified in that appendix.

### Polling

`poll_url` meets the [protocol limits] and has the same scheme, host, and
effective port as the registry API. Cargo sends an unauthenticated `GET`,
follows no redirect, and rejects a response lacking `Cache-Control: no-store`.

The URL contains an independent token with at least 128 random bits. It is not
derived from `mutation_id` and does not appear in displayed instructions.
Cargo and registries redact it from logs. Possession permits observing
short-lived status and display metadata; it cannot change readiness or execute
the mutation.

A pending poll is:

```json
{ "status": "pending", "challenge_expires_in": 240, "recommended_poll_interval_secs": 5 }
```

The other core poll results are:

```json
{ "status": "ready", "grant_expires_in": 300 }
```

```json
{ "status": "denied", "detail": "Authorization was denied." }
```

```json
{ "status": "expired", "detail": "Authorization expired." }
```

Poll responses are JSON, include `Cache-Control: no-store`, and contain the
fields required by their status plus optional fields defined by active
extensions. `detail` is optional for `denied` and `expired` and prohibited for
`ready`. Unknown response fields are ignored. A `404` or unknown state stops
the flow as a protocol error. Once a core-only final request consumes a record,
its poll capability returns `404`; Cargo has already ended its wait before that
transition.
Registries retain denied and expired results for at least the retention interval
in [protocol limits].

Cargo establishes a monotonic deadline from the initial pending response. A
later pending response can shorten but never extend it:

```text
deadline = min(deadline, now + challenge_expires_in)
```

Connection failures and complete `408`, `425`, `429`, `500`, `502`, `503`,
and `504` responses are transient for this idempotent GET. Cargo retries with
bounded exponential backoff and jitter. It honors valid `Retry-After` on `429`
and `503` only within the original deadline. Other complete responses stop.

### Display and cancellation

`detail` and each response meet the [protocol limits]. Cargo enforces the body
limit while receiving, before JSON recognition.

Cargo renders registry-controlled detail as plain text, visibly neutralizes
terminal and bidirectional controls, identifies the registry origin, and marks
recognized cross-origin HTTP(S) URLs as external. The exact display transform
is an implementation detail provided it prevents control-sequence injection
and does not turn registry text into active content.

Cargo does not interpret Markdown, hyperlinks, ANSI escapes, shell syntax, or
commands and never opens a URL automatically. Apart from visibly annotating a
recognized cross-origin HTTP(S) URL, it preserves URL-looking text and never
changes a URL target. These measures do not make a malicious configured
registry trustworthy.

When waiting starts, Cargo reports the maximum duration and how to cancel.
Initial instructions, the wait bound, `interaction_required`, denial, expiry,
protocol errors, and cancellation outcomes are essential stderr output even
under `--quiet`. Quiet mode can suppress routine authorizing/ready status and
animation, but cannot make an interactive wait invisible. Animation is used
only on a terminal.

Cancellation observed before Cargo starts the final request sends nothing and
leaves the record to expire. Once the final request begins, interruption has
ordinary in-flight ambiguity; Cargo does not claim that no request was sent.

### Composition with RFC 3231 asymmetric tokens

Mutation authorization is layered on Cargo's ordinary credential-provider
operation. Cargo asks for a credential describing the intended mutation when
it authenticates preflight and asks again after readiness before the final
request. A registry validating an RFC 3231 PASETO against preflight compares
its mutation claims with the preflight descriptor: `publish` includes crate,
version, and archive checksum; `yank` and `unyank` include crate and version.
The final endpoint validates the later credential against the ordinary request
as RFC 3231 already requires. The mutation record is bound to the registered
key as an independently revocable credential.

RFC 3231's `mutation` claim is `publish`, `yank`, or `unyank`. This RFC adds
`owners`, which Cargo already serializes for owner-change credentials. For
that value, `name` is required and identifies the crate, while `vers` and
`cksum` are absent. The owner-change direction and complete ordered owner list
remain bound by this protocol's descriptor and final request; the PASETO
identifies the operation class and crate but does not duplicate the JSON body.

An RFC 3231 challenge and a mutation-authorization poll token are separate
capabilities. A registry must not substitute one for the other or treat either
as the server-side grant. Registries that do not implement RFC 3231 continue to
use their existing bearer or credential-provider authentication unchanged.

### Credential binding and final request

The registry binds each record to an internal credential identity no broader
than one independently revocable credential. A bearer-token row or registered
RFC 3231 key is suitable. Account identity alone is insufficient when an
account has multiple independently revocable credentials.

For rotating credentials, a token family or issuer-and-subject pair is suitable
only when it denotes rotations of that same independently revocable
credential. Otherwise the exact credential instance is the identity and must
remain valid through the final request.

Verification reads its summary from the stored descriptor and enforces registry
policy before creating a short-lived grant bound to that credential identity
and descriptor. Seeing instructions or loading a verification page never makes
the record ready.

After `ready`, Cargo obtains a credential normally and sends the ordinary
request with:

```http
Cargo-Mutation-Id: mut_0123456789abcdefghijkl
```

The registry first checks mutation id, readiness, grant deadline, credential
binding, server-derived method/target/media type, absence of content encoding,
and exact `Content-Length`. It hashes the complete body, parses the ordinary
operation, and compares every semantic field. It repeats ordinary authorization
and validation; mutation authorization never enlarges credential scope.

Without an execution extension, a ready grant is single-use. The registry may
buffer and validate candidate requests without consuming it, so a mismatched or
incomplete request cannot poison the record. After one complete request exactly
matches, the registry atomically transitions the record from `ready` to an
internal `consumed` state immediately before invoking the ordinary mutation.
Only the request that wins that transition can execute; concurrent or later
requests fail without an effect. `consumed` is not a poll state and remains
unusable until record cleanup.

Missing, expired, denied, mismatched, revoked, malformed, or incomplete
requests fail before performing an effect. A protected ordinary endpoint
rejects a request without a valid mutation id regardless of Cargo version.

Without `idempotent-final`, Cargo sends the final request once. A crash after
consumption or response loss has the ordinary mutation's existing ambiguity and
cannot be retried under that record. With that extension, the execution
companion replaces the minimal `ready` to `consumed` transition with claiming,
receive leases, retry, recovery, and terminal replay while preserving the
single-logical-execution rule.

### Security and transport

Version 1 requires HTTPS except for an explicitly configured registry whose
host is a loopback IP literal. A `localhost` name is not sufficient. TLS
authenticates the registry and protects credentials, descriptors, poll replies,
and instructions.

Copying a challenge response can reveal short-lived status and invite
verification attempts but cannot create a grant. A forged `ready` poll only
causes an unauthorized final request that the registry rejects. An attacker
with the ready mutation id, bound credential, and byte-identical request is
indistinguishable from the original client.

The display contract limits terminal injection and misleading cross-origin
instructions; it does not make registry-provided text trustworthy.

Registries rate-limit preflights, verification attempts, and polls. A
preflight does not consume successful-publication quota.

### Policy and rollout

Whether an account, crate, credential, or operation requires authorization is
registry policy. A registry can exempt a short-lived, request-scoped Trusted
Publishing credential, but does not infer an exemption merely from `CI`.

Old Cargo never sends preflight, so a registry protects an operation only
after its preflight and ordinary endpoint are deployed together and the
ordinary endpoint rejects bypasses. Account and crate rollout can still be
opt-in because preflight evaluates policy. Errors should name the minimum
supported Cargo release.

## Drawbacks
[drawbacks]: #drawbacks

- Every supported mutation gains an authenticated preflight round trip, even
  against an old registry where it first returns `404`.
- Interactive policy can pause Cargo while authorization completes.
- Cargo must retain replayable request bytes until the final request.
- Registries store short-lived records and implement exact descriptor binding.
- Protected operations require a Cargo version that implements this protocol.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Preflight avoids uploading a package before policy decides whether Cargo may
proceed. Holding an uploaded package for later approval would instead define a
staged-publication lifecycle with reservation, visibility, replacement, and
cleanup rules.

Defaulting registry tokens to the OS credential store ([RFC 3981], building on
[RFC 2730]) reduces plaintext-file exposure and changes where a usable token
lives. Once an attacker possesses that token, it remains sufficient authority
for every in-scope mutation. This RFC adds a registry-enforced second
requirement on the mutation itself. The two compose: safer storage reduces how
often tokens leak; mutation authorization limits what a leaked token can do
unattended.

A registry may require passkey verification, administrator approval, SSH, or
another policy. Cargo does not infer or participate in the verification method.

`allow_pending: false` lets a non-interactive client learn whether its credential
is exempt without creating state it cannot consume. Persisting pending records
and replayable requests across Cargo invocations would require a separate
on-disk recovery protocol.

Preflight discovery trades one small request to old registries for removal of
static configuration that duplicates deployed server code and can drift from
it. Explicit per-record extension activation prevents either peer from assuming
an extension that the other did not select. A single version field is enough:
an unsupported response cannot be used safely, and a future Cargo can choose
which request version to send without a separate version-list protocol.

## Prior art
[prior-art]: #prior-art

- npm can require 2FA for publish and package settings. The npm CLI prompts
  for an account OTP or WebAuthn assertion and sends it with the publish
  request. As of August 2026, package policy can require 2FA, allow a granular
  token that bypasses 2FA, or disallow tokens. Bypass granular tokens lost
  account- and package-governance actions in August 2026 and are scheduled to
  lose direct publish around January 2027. Automated publish then uses trusted
  publishing or staged publish. The second factor authenticates the npm user for
  that command; npm and the registry are versioned together.
- RubyGems can require MFA (OTP or WebAuthn) for `gem push`, owner changes, and
  sign-in. WebAuthn uses a localhost verification page; OTP is typed or passed
  as `--otp`. The factor authenticates the gem owner to RubyGems.org for the
  command. High-download gems must enable MFA.
- This RFC is the Cargo-side generic handshake those products did not need as a
  separate protocol. Cargo talks to many registries, so the client waits on a
  versioned pending/ready grant bound to the exact mutation descriptor,
  including the publish archive digest. The registry chooses the factor.
  Companion proposals cover loopback wake-up (RubyGems-like) and idempotent
  retry. crates.io website MFA, enrollment, and mandate policy remain registry
  decisions; existing tracker items include [#815], [#13253], and [#13369].
- [RFC 9470] communicates stronger authentication requirements but obtains a
  different token rather than a mutation-bound server grant.
- OAuth defines its framework in [RFC 6749], device polling and slowdown in
  [RFC 8628], and native-app loopback guidance in [RFC 8252]. This separation
  keeps the base flow independent of the local callback optimization.

[#815]: https://github.com/rust-lang/crates.io/issues/815
[#13253]: https://github.com/rust-lang/crates.io/discussions/13253
[#13369]: https://github.com/rust-lang/crates.io/discussions/13369
[RFC 9470]: https://www.rfc-editor.org/rfc/rfc9470.html
[RFC 6749]: https://www.rfc-editor.org/rfc/rfc6749.html
[RFC 8628]: https://www.rfc-editor.org/rfc/rfc8628.html
[RFC 8252]: https://www.rfc-editor.org/rfc/rfc8252.html
[Cargo Registry Web API]: https://doc.rust-lang.org/cargo/reference/registry-web-api.html
[idempotent mutation execution]: 0000-cargo-registry-mutation-idempotency.md
[loopback wake-up]: 0000-cargo-registry-loopback-callback.md
[protocol limits]: #appendix-protocol-limits

## Appendix: protocol limits

These version 1 bounds are normative. Keeping them together makes the wire
limits auditable without interrupting the protocol flow.

| Item | Version 1 bound |
| --- | --- |
| `preflight_id`, `mutation_id` | At least 128 random bits encoded as 22 through 128 URL-safe ASCII characters. |
| `requested_extensions` | At most 16 distinct entries. |
| Extension name | 1 through 64 lowercase ASCII letters, digits, or hyphens. |
| `challenge_expires_in`, `grant_expires_in` | Positive whole seconds from receipt, no greater than 300. |
| `recommended_poll_interval_secs` | Registries send 1 through 30; Cargo defaults to 5 when absent and clamps any other received value to this range. |
| `poll_url` | No more than 8,192 bytes. Its independent token has at least 128 random bits. |
| `detail` | Nonempty UTF-8, no more than 8,192 bytes whenever present. |
| Complete preflight or poll response | No more than 65,536 bytes. |
| Denied or expired poll result retention | At least five minutes after transition. |

## Appendix: conformance cases

1. Different preflight ids create distinct records. Retrying one id returns its
   record and rejects changed descriptor, `allow_pending`, active extension
   set, or recognized extension fields. Changes consisting only of ignored
   unknown extension names or fields do not change the record.
2. Non-interactive `auto` can receive immediate `ready` but cannot create a
   pending record.
3. A rotating credential succeeds only when both instances map to the same
   narrow binding identity.
4. Poll tokens are independent, unauthenticated, same-origin, non-redirecting,
   non-cacheable, and redacted.
5. Later pending responses cannot extend Cargo's original deadline.
6. Transient polls use bounded backoff, jitter, and applicable `Retry-After`.
7. Body, parsed fields, credential, endpoint, encoding, content length, or
   grant mismatch performs no mutation.
8. Two concurrent exact requests under a core-only ready grant cause at most one
   ordinary mutation; after atomic consumption neither can reuse the record.
9. A consumed core-only record cannot become ready again; its preflight retry
   fails with `409` and its poll capability returns `404`.
10. A protected direct request without a mutation id performs no mutation.
11. A path-prefixed API base produces matching preflight and final endpoints.
12. `--quiet` retains instructions, wait bounds, and terminal outcomes.
13. Controls and bidi formatting are neutralized and external URLs labeled.
14. Oversized response or detail is rejected before display.
15. Cancellation before the final request begins produces no final request.
16. A definitive preflight `404` falls back to the ordinary mutation. Transport
    errors, other statuses, malformed responses, and unsupported response
    versions fail without sending it.
17. The registry may activate any supported subset of requested extensions.
    Cargo rejects duplicate or unrequested active names and relies on an
    extension only when the response echoes it as active.

Companion extension conformance is defined in its proposal and applies whenever
that extension is activated for a record.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should stabilization cover the poll-based core first, with
  `idempotent-final` and `loopback-callback` tracked separately, or should Cargo
  stabilize the three proposals as one user-visible feature?
- Should version 1 define how an RFC 3231 registry challenge on preflight is
  obtained, consumed, and retried before mutation policy runs, or leave that
  composition to a later stabilization step?

## Future possibilities
[future-possibilities]: #future-possibilities

Later proposals can add staged publication, other interaction channels, or new
registry mutation operations without changing the core authorization boundary.

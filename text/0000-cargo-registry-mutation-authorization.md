- Feature Name: `cargo_registry_mutation_authorization`
- Start Date: 2026-07-31
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Cargo Issue: [rust-lang/cargo#0000](https://github.com/rust-lang/cargo/issues/0000)

## Summary
[summary]: #summary

Define a Cargo registry protocol for additional authorization of an already
authenticated mutation. Cargo preflights an exact mutation descriptor under a
fresh invocation identifier, waits if the registry returns structured
verification instructions, then sends the mutation under the registry's
mutation id. The registry chooses the verification method. The preflight
neither stages nor reserves a package; the operation succeeds only when the
bound mutation succeeds.

This RFC calls the general mechanism *mutation authorization*. *Step-up
authentication* is one policy that can make a mutation ready, but a registry
can instead require another party's approval and Cargo cannot attest which
method occurred. A *challenge* is a pending authorization request. `ready`
means Cargo may send the mutation, not that the mutation has succeeded.

## Motivation
[motivation]: #motivation

Cargo commonly authenticates registry mutations with a long-lived API token.
Scopes, crate restrictions, and credential providers reduce its exposure, but
a token with publish authority can still be used without the user's presence.
Credential-stealing malware, a leaked CI secret, or an exposed backup can
therefore enable an unattended supply-chain compromise.

Website account MFA does not address this threat because Cargo uses a
separately issued credential. Existing Cargo authentication RFCs improve
storage, authentication, and replay resistance but cannot require additional
authorization for an exact mutation descriptor ([RFC 2730], [RFC 3139], and
[RFC 3231]).

A registry can put verification instructions in an ordinary error, but Cargo
cannot then wait safely or know when to continue. Users must rerun commands
manually and wrappers must parse prose. Retrying a publish can also repeat a
large upload or leave the client uncertain whether the mutation succeeded.

Cargo can provide common preflight, polling, CI, and idempotent retry behavior.
Registries remain responsible for verification, exact request binding,
recovery, and policy, allowing alternatives without crates.io accounts or
passkeys. This protocol does not make a compromised publishing client
trustworthy: malware active before preflight can alter the package that Cargo
then faithfully describes and binds.

Version 1 covers publish, yank, unyank, and owner changes. The motivating
crates.io policy is opt-in API MFA using passkeys. Enrollment, recovery, and
exceptions for automation such as Trusted Publishing remain registry policy.

[RFC 2730]: https://rust-lang.github.io/rfcs/2730-cargo-token-from-process.html
[RFC 3139]: https://rust-lang.github.io/rfcs/3139-cargo-alternative-registry-auth.html
[RFC 3231]: https://rust-lang.github.io/rfcs/3231-cargo-asymmetric-tokens.html

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A protected publish looks like this:

```console
$ cargo publish
   Packaging example v1.2.3 (/work/example)
 Authorizing example v1.2.3 (/work/example)
       Note Instructions from registry https://crates.io:
            Additional authorization is required. Visit https://crates.io/verify/mut_0123456789abcdefghijkl.
    Waiting for registry authorization (up to 5m; press Ctrl-C to cancel)
       Note Authorization ready; continuing
   Uploading example v1.2.3 (/work/example)
   Uploaded example v1.2.3 to registry `crates-io`
```

Cargo will package the crate once and hash the complete ordinary publish
request body. The preflight will contain that digest plus the crate, version,
request size, archive size, and archive digest, but not the archive bytes. The
registry can identify and bind the exact operation, but it has not accepted,
published, indexed, or reserved the version. Reaching `ready` authorizes
Cargo to send the mutation. Only that request can produce the normal publish
success response.

Cargo will print the registry's instructions rather than collecting a password,
passkey assertion, OTP, or other factor. The registry chooses how to obtain
the required verification and displays the exact operation being authorized.
It might ask the user to visit a website, run an SSH command, use a hardware
device, or obtain an administrator's confirmation.

A protected yank, unyank, or owner change follows the same visible workflow.
For example, `cargo yank --version 1.2.3 example` can pause with instructions to
authorize "Yank example 1.2.3", while `cargo owner --add alice example` can
pause to authorize the exact owner addition. After the record becomes ready,
Cargo sends only the bound operation under its mutation id; the user does not
manually rerun the command.

On an interactive workstation, Cargo will request both a loopback callback and
a polling fallback. The callback makes local completion immediate. Polling
also works when verification happens on another machine, such as during an SSH
session:

```console
$ cargo publish --registry-authorization=poll
```

The common `--registry-authorization` option accepts `auto`, `loopback`,
`poll`, and `disabled` on every affected command. The same preference can be
set through `CARGO_REGISTRY_MUTATION_AUTHORIZATION_CHANNEL`,
`registries.<name>.mutation-authorization-channel` for a named registry, or
`registry.mutation-authorization-channel` for crates.io. Precedence is the
command-line option, environment variable, selected registry's configuration,
then the `auto` default. An explicit `loopback` or `poll` opts into waiting even
without a terminal or in CI. `disabled` skips the extension and uses the
existing ordinary mutation behavior; a policy-protected endpoint will reject
that request.

For example, a user can keep the default automatic behavior while selecting
polling for a remote registry:

```toml
[registry]
mutation-authorization-channel = "auto"

[registries.corporate]
mutation-authorization-channel = "poll"
```

In automatic mode without a terminal, or when `CI` is `true` or `1`, Cargo
still performs a non-waiting preflight so credentials exempted by registry
policy can proceed. If authorization would require a wait, the registry does
not create a challenge. Cargo exits with an error explaining that no challenge
was created and suggests rerunning with
`--registry-authorization=poll`. Following registry instructions can therefore
never authorize a record that the invoking Cargo process has abandoned.

Older Cargo versions receive the registry's direct-mutation error and exit. A
registry with a separate reactive compatibility path can include complete
verification instructions and permit a manual rerun, but the version 1
preflight protocol alone does not make that rerun succeed. Older clients cannot
automate polling or the callback.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The protocol progresses from an exact mutation descriptor to a mutation record,
then through the normative lifecycle defined below. Its client-visible
authorization states are `pending`, `ready`, `denied`, and `expired`. `ready`
can mean that no additional verification was required or that verification
completed; it never means that the mutation itself succeeded. The protocol
artifacts are:

| Artifact | Location | Purpose |
| --- | --- | --- |
| Primary credential | Cargo; validated by registry | Authenticates preflight and final mutation |
| `preflight_id` | Cargo and registry | Identifies one logical invocation and deduplicates preflight retries |
| Mutation descriptor | Cargo; stored by registry | Defines the exact intended request |
| `mutation_id` | Cargo and registry | Identifies record and idempotent execution |
| Poll token | Cargo and registry | Authorizes read-only mutation-record status |
| Callback state | Cargo and verification document | Rejects unsolicited loopback wake-ups |
| Grant | Registry | Authorizes a ready mutation |
| Terminal response | Registry | Replays the completed mutation's outcome |

### Scope and initial request

The protocol applies to authenticated Cargo registry Web API mutations. It does
not change downloads, registry indexes, `cargo login`, website sessions, or the
primary credential format.

Supporting registries advertise a version-independent envelope in index
`config.json`:

```json
{
  "mutation-authorization": {
    "versions": [1],
    "operations": ["publish", "yank", "unyank", "owners"]
  }
}
```

`versions` is a nonempty array of distinct positive integers. `operations` is a
nonempty array of distinct strings naming the ordinary mutations for which the
registry implements this protocol; unknown operations are ignored. Cargo
selects the highest version it supports from `versions`. If the requested
operation is listed but there is no mutually supported version, Cargo fails
before sending either a preflight or the ordinary mutation. An absent key, or
an operation absent from `operations`, retains existing Cargo behavior. The
version-list negotiation follows the precedent of the [Cargo credential
provider protocol].

Older Cargo versions ignore the unknown key and continue using the existing Web
API, so advertisement is not enforcement. A registry advertises an operation
only after its preflight and final endpoints satisfy version 1, including the
fail-closed and idempotency requirements below.

Before an advertised mutation, Cargo sends an authenticated `POST` to
`/api/v1/auth/mutation-challenges`. Registry Web API endpoint paths in this RFC
are appended to the `api` base URL from index `config.json`; they are not
resolved as origin-absolute references that discard a path prefix in that base.
The request describes the exact ordinary HTTP mutation without performing it.
The methods, media types, and ordinary paths are those of the [Cargo Registry
Web API]. For publish the descriptor contains:

```json
{
  "protocol_version": 1,
  "preflight_id": "pf_0123456789abcdefghijklmnopqr",
  "allow_pending": true,
  "operation": "publish",
  "method": "PUT",
  "request_target": "/api/v1/crates/new",
  "content_type": "application/octet-stream",
  "crate": "example",
  "version": "1.2.3",
  "request_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "request_size": 124678,
  "archive_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "archive_size": 123456
}
```

The preflight uses `Content-Type: application/json` and
`Accept: application/json`. Cargo does not follow a redirect for this
credential-bearing request. Ready, pending, and `interaction_required`
responses use `Content-Type: application/json` and
`Cache-Control: no-store`.

Cargo generates a new `preflight_id` for each logical command invocation. It
contains at least 128 bits of cryptographically secure randomness encoded as 22
through 128 URL-safe ASCII characters. It is not a credential. Cargo reuses it
only when retrying an interrupted or response-ambiguous preflight; a later
command, even for byte-identical input, generates a new value.
`preflight_id` and the Boolean `allow_pending` are required in every version 1
preflight descriptor. `allow_pending` is client policy, not mutation authority:
it controls whether the registry may create a pending record but never exempts
the operation from registry authorization.

Cargo resolves the channel before preflight. Explicit `loopback` and `poll`
set `allow_pending` to `true`. Interactive `auto` also sets it to `true`; `auto`
with non-terminal standard input or `CI` equal to `true` or `1` sets it to
`false`. `disabled` sends neither a preflight nor this field. `--quiet` does not
change channel selection. The precedence is the command-line option,
`CARGO_REGISTRY_MUTATION_AUTHORIZATION_CHANNEL`, per-registry Cargo
configuration, then the `auto` default.

`request_target` is the final URL's origin-form path and optional query, with
the exact percent-encoding Cargo will use. It therefore includes any path
prefix from the configured API base. Version 1 operations do not use a query,
and a registry rejects one. `content_type` is the normalized media type Cargo
will send: `application/octet-stream` for publish, `application/json` for owner
changes, and `null` for bodyless yank and unyank requests. Parameters are not
permitted. Version 1 does not use `Content-Encoding`; Cargo omits that header
and the registry rejects a final request containing it. Transfer framing is not
part of the descriptor.

`request_sha256` is the SHA-256 of the HTTP content bytes Cargo will send after
transfer framing has been removed. Because content encoding is prohibited,
these are also the bytes presented to the endpoint parser. Hashing raw bytes
avoids a cross-implementation canonical-JSON algorithm. The registry treats
every descriptor field as untrusted, validates its syntax, and uses it for
credential scope and the server-generated operation summary. The parsed final
request remains authoritative. Cargo sends `Content-Length` equal to
`request_size`; the registry rejects an absent or different value before
claiming the record.

For yank, unyank, and owner changes, the preflight similarly contains the HTTP
method, exact request target, content type, raw request-body digest and size,
and the crate, version, direction, and complete owner list as applicable. A
bodyless request uses the SHA-256 of the empty byte string.

The registry computes the mutation fingerprint from the selected protocol
version, validated method, exact request target, content type, absence of
content encoding, raw body digest and size, operation, and applicable crate,
version, or owner fields. A publish fingerprint additionally includes the
archive SHA-256 and size. This definition is used throughout the rest of the
protocol. The primary `Authorization` value is excluded because the stable
credential binding identity is stored separately. `allow_pending` and callback
delivery metadata are also excluded because they describe how this invocation
can wait, not what it intends to mutate.

When the selected channel includes loopback and `allow_pending` is `true`,
Cargo first binds an ephemeral IPv4 listener on the literal loopback address
`127.0.0.1` and adds a callback object to the preflight descriptor:

```json
{
  "callback": {
    "url": "http://127.0.0.1:49152/cargo/registry-authorization"
  }
}
```

The port is a decimal integer from 1024 through 65535. The callback URL uses
exactly the `http` scheme, the IPv4 literal `127.0.0.1`, the bound port, and the
path `/cargo/registry-authorization`; it contains no user information, query,
or fragment. Cargo binds only `127.0.0.1`, never the wildcard address
`0.0.0.0`. The verification client uses the same IP literal and does not
substitute `localhost`, so name resolution cannot select `::1` or a
non-loopback address. If IPv4 loopback is unavailable, Cargo retains the
polling path. Cargo omits `callback` in `poll` and `disabled` modes and for a
non-waiting automatic preflight.

The callback URL is delivery metadata, not part of the mutation fingerprint.
The registry stores it with the record but never connects to it. A retry using
the same `preflight_id` must contain a byte-for-byte identical callback object;
otherwise the registry rejects the retry without changing the record. This
keeps one logical invocation from replacing its registered listener.

The registry validates the primary credential and ordinary operation scope
during preflight. If `allow_pending` is `false` and registry policy would
require a pending record, it returns `403 Forbidden` with a non-cacheable,
preflight-only result:

```json
{
  "status": "interaction_required",
  "protocol_version": 1,
  "detail": "This operation requires registry authorization."
}
```

The registry does not create a mutation record, mutation id, poll token, or
actionable verification URL in this case. Cargo renders an optional `detail`
under the same limits and sanitization as other registry text, then reports
that no challenge was created and suggests the explicit `poll` channel. A
registry never returns `pending` when `allow_pending` is `false`; it can still
return `ready` when policy requires no interactive authorization.

Cargo recognizes this result only when the HTTP status is `403`, `status` is
exactly `interaction_required`, `protocol_version` is the selected version,
and `mutation_id`, `poll_url`, and `verification_url` are absent. Any mismatch
is a protocol or registry error, not an invitation to wait or send the
mutation.

Otherwise the registry atomically maps a credential binding identity and
`preflight_id` to one mutation record. While that mapping is retained, a
preflight retry returns the same record; an expired or denied record is not
silently replaced with a new challenge. Reusing a `preflight_id` with a
different `allow_pending` value, mutation fingerprint, or callback object fails
without changing the existing record. Otherwise the registry creates a record
with a fresh, unpredictable `mutation_id`. Concurrent byte-identical commands
with different `preflight_id` values remain distinct logical mutations, and a
completed record is never selected merely because a later command has the same
fingerprint.
Registries retain the mapping for at least as long as any state or terminal
outcome associated with the record.

`mutation_id` contains at least 128 bits of cryptographically secure randomness
encoded as 22 through 128 URL-safe ASCII characters. It identifies the record
and becomes the final mutation's idempotency key, but is not sufficient to
authorize that mutation.

Preflight uses the intended mutation's credential-provider operation; its RFC
3231 interaction is detailed under Compatibility.

If no additional authorization is required, the registry responds with a
non-cacheable ready record:

```json
{
  "status": "ready",
  "protocol_version": 1,
  "mutation_id": "mut_0123456789abcdefghijkl",
  "grant_expires_in": 300
}
```

Cargo then performs the mutation immediately. If additional authorization is
required and `allow_pending` is `true`, the registry returns the challenge
response below; the `false` case returns `interaction_required` as specified
above.

### Challenge response

When preflight requires additional authorization and permits a pending record,
the registry responds with `202 Accepted`. This means that preflight succeeded
and created a pending record; it is not success for the intended mutation. The
response includes `Cache-Control: no-store` so challenge state is not retained
by HTTP caches:

```json
{
  "status": "pending",
  "detail": "Additional authorization is required. Visit https://registry.example/verify/mut_0123456789abcdefghijkl.",
  "protocol_version": 1,
  "mutation_id": "mut_0123456789abcdefghijkl",
  "operation": "publish",
  "crate": "example",
  "operation_summary": "Publish example 1.2.3",
  "verification_url": "https://registry.example/verify/mut_0123456789abcdefghijkl",
  "poll_url": "https://registry.example/api/v1/auth/mutation-challenges/poll/poll_0123456789abcdefghijkl",
  "challenge_expires_in": 300,
  "recommended_poll_interval_secs": 5
}
```

#### Field contract

The required version 1 fields are `status`, `detail`, `protocol_version`,
`mutation_id`, `poll_url`, and `challenge_expires_in`. Cargo recognizes a
challenge only when all have the expected JSON types, `status` is `pending`,
and `protocol_version` is the selected version. Otherwise it reports a protocol
or registry error. All other members are optional, and unknown fields are
ignored. Invalid credentials or an ordinarily forbidden operation use the
registry's normal `401` or `403` error rather than a pending response.

#### Displaying instructions

`detail` is untrusted presentation data and never affects request binding,
readiness, or authorization. It is a nonempty UTF-8 string of at most 8192
bytes containing complete instructions for satisfying the challenge and
continuing the operation. The complete preflight or poll response body
is at most 65536 bytes. Cargo enforces that response limit while receiving the
body and rejects an oversized body or `detail` instead of recognizing or
waiting on the challenge.

Cargo renders `detail` as inert plain text. It does not interpret Markdown,
terminal escapes, hyperlinks, shell syntax, or commands, and never
automatically opens a URL. Before rendering, Cargo normalizes CRLF and bare CR
to LF. It then replaces every C0 control except LF, DELETE (`U+007F`), every C1
control, and Unicode bidirectional formatting or isolate control with a visible
replacement character. This mitigates terminal injection ([CWE-150]) and
bidirectional spoofing ([Unicode UTR #36]). Cargo prefixes the instructions
with `Instructions from registry <origin>:`, leaves cross-origin URLs
printable, and marks each recognized one as external. These measures do not
make instructions from a malicious configured registry trustworthy.

Registry instructions, the initial wait duration, an
`interaction_required` explanation, terminal `denied` or `expired` results,
protocol errors, and cancellation outcomes are essential output. Cargo writes
them to standard error even under `--quiet`; implementations must not route
them only through status or note helpers that quiet mode suppresses. Quiet mode
can suppress routine `Authorizing` and `Authorization ready` status lines and
periodic progress, but it does not turn an interactive wait into an invisible
one.

`verification_url` is an optional structured URL for a callback-capable web
verification flow. `detail` remains complete by itself for poll-only and
non-web mechanisms; Cargo never discovers a protocol URL by parsing that prose.
When a callback object was supplied, Cargo uses `verification_url` only if it
has the same origin as the registry API URL, is at most 8192 bytes, and contains
no fragment. Cargo generates at least 128 bits of cryptographically secure
random callback state, encodes it as 22 through 128 URL-safe characters, and
adds this client-held fragment locally before displaying the URL:

```text
https://registry.example/verify/mut_0123456789abcdefghijkl#callback_state=<callback-state>
```

The fragment is absent from the user agent's HTTP request. After capturing the
state, the verification document can navigate to the stored callback URL with
`?state=<callback-state>`. Cargo accepts only a bounded HTTP/1.x `GET` from a
loopback peer and compares `state` in constant time. A match is only a wake-up
signal: Cargo performs an immediate poll and continues solely if the registry
reports `ready`. The callback carries no server proof and never authorizes the
mutation. If `verification_url` is absent or invalid, Cargo displays `detail`
unchanged and uses polling. Cargo leaves URLs written in `detail` inert and
unmodified.

The callback state is accessible to every script executing in the
verification document. A registry that supports callbacks serves that URL
directly without an HTTP or script redirect, excludes third-party active
content, and applies a restrictive Content Security Policy. At minimum, the
policy denies content by default, permits only nonce- or hash-authorized
first-party script, limits connections and form submissions to the registry
origin, disables `base-uri` and plugins, and prevents framing. The page removes
the fragment from the visible URL and browser history with
`history.replaceState()` immediately after first-party code captures it. CSP is
defense in depth and does not make an intentionally allowed third-party script
safe; such a script must not execute in the callback-capable document.

#### Other fields and HTTP status

`operation`, `crate`, and `operation_summary` provide optional display and
diagnostic context. Cargo does not use them for authorization or URL
construction, and version 1 does not consume `operation_summary`.

`protocol_version`, `status`, `mutation_id`, and `grant_expires_in` are
required in an immediately ready response. `challenge_expires_in` and
`grant_expires_in` are positive integer counts of
whole seconds, measured from receipt of the response carrying them. The former
is required on `pending`; the latter is required on `ready`. Cargo rejects zero,
negative, non-integer, or values above 300 rather than silently changing the
registry's advertised lifetime. Relative lifetimes avoid dependence on the
client's wall clock. The corresponding registry deadlines are authoritative;
these fields are conservative client views of their remaining duration.

`recommended_poll_interval_secs` is advisory. Registries send a value from 1
through 30 seconds; Cargo defaults to 5 and defensively clamps other values.

An immediately ready preflight returns `200 OK`; a pending preflight returns
`202 Accepted`; and `interaction_required` returns `403 Forbidden`. A retry
whose existing record is `denied` or `expired` returns `200 OK` with that
status, `protocol_version`, `mutation_id`, and optional sanitized `detail`.
`interaction_required` is a preflight outcome, not a mutation-record state.
Cargo does not treat any preflight response as success for the intended
mutation and does not begin post-publish index observation until the ordinary
publish endpoint returns success.

#### Polling URL

`poll_url` has the same origin as the registry API URL from index `config.json`,
including scheme and effective port, and is at most 8192 bytes. Cargo rejects
the handshake if it differs or is oversized. The poll request does not follow
redirects.

The URL contains a poll token with at least 128 bits of cryptographically secure
randomness encoded as 22 through 128 URL-safe ASCII characters. The token is
independent of `mutation_id`; it is not derivable from an identifier shown in
`detail` or `verification_url`, and the registry does not place the poll URL in
human-readable instructions. Cargo and registries redact it from logs. Anyone
who obtains it can observe the short-lived status and display metadata, but it
cannot make a record ready or execute a mutation.

### Credential binding and readiness

Successful primary-credential validation produces a registry-internal
*credential binding identity*. Cargo does not send this identity and credential
providers are not required to expose one. The registry binds the record and
grant to that identity plus the mutation fingerprint defined above.

For a long-lived bearer token the identity can be its independently revocable
token record. For RFC 3231 it can be the registered key identity and subject.
For rotating short-lived credentials, an issuer-defined token family or an
authorized-party and subject pair is suitable only when it denotes rotations
of the same independently revocable credential. Binding only to an account is
insufficient when that account has multiple independently issued or revocable
credentials.

If validation cannot produce an identity stable across refresh, the registry
uses the exact validated credential instance as the binding identity. Such a
credential can use version 1 only if the same instance remains valid for the
final request. A final credential with another binding identity fails without
claiming, expiring, or terminalizing the record. Cargo can start a fresh
preflight using the new credential. If an allowed credential form necessarily
rotates between the two requests, the registry must define a suitably narrow
stable identity for that rotation, exempt it by policy, or reject its
preflight.

The registry shows a server-generated summary of this bound operation through
its chosen verification mechanism. Receiving or viewing the instructions alone
does not complete verification. The registry enforces its configured freshness,
user-presence, or approval properties before setting the challenge to `ready`.
A broad authorization window is insufficient because an
attacker holding the token could substitute another mutation.

The verification mechanism obtains this summary from the stored mutation
record, not from `detail` or verification-URL query parameters. For publish,
the verification UI should display the archive SHA-256 alongside crate name and
version so a user who has an independently obtained digest can compare it. The
digest does not imply that a user without such a reference inspected the
archive.

A verification surface does not describe a transition to `ready` as a
successful publish, yank, unyank, or owner change. It says that the operation
is authorized and that Cargo must still submit it. This remains true when the
client has disconnected or cancelled and the unused record will expire.

Until readiness, the registry has only the descriptor; it neither
receives the mutation body nor performs or stages the operation.

### Polling and completion

Cargo polls `poll_url` without the primary authorization credential and does
not follow redirects. Every poll response includes `Cache-Control: no-store`;
Cargo rejects a response that can be reused by a shared cache. A pending
response is:

```json
{ "status": "pending", "challenge_expires_in": 240, "recommended_poll_interval_secs": 5 }
```

Version 1 defines four polling states:

| Status | Cargo behavior |
| --- | --- |
| `pending` | Continue waiting and polling. |
| `ready` | Stop waiting and send the bound mutation. |
| `denied` | Stop without sending the mutation and report that registry authorization was denied. |
| `expired` | Stop without sending the mutation and report that registry authorization expired. |

A `ready` poll includes `grant_expires_in`:

```json
{ "status": "ready", "grant_expires_in": 300 }
```

After a final request has claimed the record, conforming Cargo no longer polls.
For deterministic behavior if it does, the registry projects `receiving`,
`executing`, and retained `terminal` records as `ready` while their final
endpoint can still accept a matching request. The returned
`grant_expires_in` is a conservative window for beginning that request and does
not extend a receive lease or terminal-retention deadline.

A `denied` or `expired` response can include an optional `detail` with a
registry-provided explanation. Cargo applies the same size limits,
sanitization, and origin prefix as challenge instructions. A `404` or unknown
status also stops the flow but is reported as a protocol or registry error
rather than as a user decision.

```json
{ "status": "denied", "detail": "The authorization request was denied." }
```

The registry keeps `denied` and `expired` observable through the poll token for
at least five minutes after the transition so an on-time client receives the
specific result rather than an ambiguous `404`.

A `pending` response includes `challenge_expires_in` as the registry's
conservative remaining challenge lifetime. Cargo establishes a monotonic wait
deadline from the initial pending response. A later pending response can only
shorten it:

```text
deadline = min(existing_deadline, now + challenge_expires_in)
```

It never extends the original wait. A transition to `ready` includes
`grant_expires_in` and establishes a separate deadline for beginning the final
request.

The registry can update `recommended_poll_interval_secs` in a `pending`
response. Version 1 treats transport errors and complete `408`, `425`, `429`,
`500`, `502`, `503`, and `504` responses as transient for this safe,
idempotent, read-only `GET` ([RFC 9110]). Cargo retries them with bounded
exponential backoff and jitter within the monotonic wait deadline. It honors a
valid `Retry-After` on `429` or `503` only when the delay fits within that
deadline; otherwise the deadline remains authoritative. Each request timeout
is itself bounded by the remaining wait. Redirects and other complete HTTP
responses stop the flow.

When Cargo begins waiting, it displays the maximum remaining duration and that
Ctrl-C cancels the wait. When standard error is a terminal and quiet mode is
off, Cargo also renders a transient progress indication with elapsed or
remaining time; it does not animate when standard error is not a terminal or
under `--quiet`. Registry instructions and the initial bounded-wait line remain
visible in those modes as essential output.

While a record is pending, Cargo handles Ctrl-C promptly and arbitrates it
against continuation with a single local transition from `waiting` to either
`cancelled` or `sending`. If cancellation wins, Cargo initiates no final
mutation request and reports that no mutation was sent and that the registry
record will expire server-side. Version 1 has no remote cancellation request.
If `sending` wins and transmission begins first, interruption has the
ordinary ambiguity of an in-flight mutation; Cargo must not claim that no
mutation was sent. A callback racing with cancellation is still only a wake-up,
so it cannot bypass this transition.

Because the poll token is a read-only capability, Cargo does not send the
primary credential on the `GET`. The security consequences of possessing it are
described below.

When the registry sets the challenge to `ready`, it creates a short-lived
server-side grant bound to the credential binding identity and mutation
fingerprint. The grant cannot authorize an altered request or another
operation. It is the only additional-authorization artifact used by the final
endpoint. Callback delivery merely advances the next poll; it does not create,
carry, or replace the grant.

### Idempotent mutation and implementation limits

The registry maintains this normative internal lifecycle:

```text
pending --verification--> ready --request claim--> receiving
   |                         |                         |
   +--> denied               +--> expired              +--> expired
   +--> expired                                        |
                                                      v
                      terminal <--logical commit-- executing
```

- `pending` has a challenge deadline by which verification must finish.
- `ready` has a grant deadline by which a valid final request must begin.
- `receiving` has a bounded receive lease established when a request first
  claims the record. The grant remains valid for matching attempts under the
  same mutation id until that lease ends, even if the ready deadline passes.
- `executing` begins only after the complete request and its parsed semantics
  match. It continues to a logical commit or rolls back/resumes as described
  below; it is not governed by the challenge, grant, or receive deadlines.
- `terminal` contains the committed effect and replayable response. Its replay
  retention is measured from the terminal commit.
- `denied` and `expired` never permit a final mutation.

The challenge and grant deadlines are distinct registry timestamps. The
receive lease is bounded by registry policy and must permit a reasonable upload
and at least one transport retry for the registry's maximum accepted request
size. A request can enter `receiving` only while `ready`, after its mutation id,
primary credential binding, method, request target, content type, absence of
content encoding, and declared HTTP content length have passed the checks that
can be performed before reading the body. An incomplete or mismatched body does
not execute or terminalize the record. Matching retries can use the remaining
receive lease; when it expires without a complete match, the record becomes
`expired`.

When policy protects an operation, the ordinary mutation endpoint must reject
a request lacking a valid ready or receiving `Cargo-Mutation-Id`, or a matching
terminal record, before performing any effect. The same applies to old Cargo,
direct HTTP clients, and clients that ignore the advertised extension. A
registry-specific reactive compatibility workflow is outside version 1 and
cannot claim its idempotency guarantees. A registry that has no such legacy
workflow returns its normal error envelope, ordinarily with `403 Forbidden`,
and names the minimum supported Cargo release in `detail`.

After the challenge reaches `ready`, Cargo sends the ordinary request with a
fresh credential obtained through the normal credential-provider lookup and:

```http
Cargo-Mutation-Id: mut_0123456789abcdefghijkl
```

While receiving the final request, the registry recomputes its content digest
and size and validates its parsed operation fields against the stored
fingerprint. For publish this includes parsing the ordinary body and
recomputing the archive digest and size. The registry repeats ordinary
authorization and validation; mutation authorization neither replaces the
primary credential nor enlarges its scopes.

The registry serializes requests with the same mutation id. At most one logical
execution is active; a rolled-back or resumable attempt does not permit a
second committed effect. Concurrent matching requests wait for that execution
or receive a nonterminal retry response. Once `terminal` exists, later matching
requests receive its stored response. A terminal response is retained for at
least 24 hours after the terminal commit, independently of earlier challenge,
grant, or receive deadlines. A fresh command uses a fresh `preflight_id` and can
therefore retry an operation whose earlier logical invocation completed with an
application error.

The stored terminal response consists of the endpoint's status, body, and
`Content-Type`. Registries do not persist or replay hop-by-hop headers,
`Set-Cookie`, `Date`, authentication challenges, or dynamic rate-limit headers
as part of the outcome.

Failures before execution fall into two classes:

- Missing or unknown mutation ids, non-ready or expired records, credential
  binding failures, revoked or insufficient credentials, metadata or body
  mismatches, oversized or incomplete bodies, and malformed requests fail
  without storing a terminal outcome.
- Admission controls and transient failures, including `408`, `425`, `429`,
  `500`, `502`, `503`, and `504`, fail without storing a terminal outcome. A
  valid `Retry-After` can accompany `429` or `503`.

These failures cannot poison the record. After the complete request matches
and ordinary authorization succeeds, a normal `2xx` response or a deterministic
application `4xx` response can be terminal. An implementation does not mark a
response terminal if repeating the same logical execution could legitimately
change it. In particular, primary-credential failures, admission control, and
transient server failures remain nonterminal even when discovered after the
body arrived.

The mutation effect and terminal outcome are one logical commit. A registry
advertises an operation for version 1 only when that endpoint can satisfy the
following crash behavior, using a database transaction or endpoint-specific
idempotency for external effects:

| Failure point | Required recovery |
| --- | --- |
| Before the complete request matches | No effect occurred and no terminal outcome is stored; the same mutation id can retry within its lease. |
| After matching but before logical commit | Effect and outcome roll back together, or retry safely resumes the same logical execution. |
| After commit but before Cargo receives the response | The effect is not repeated and the stored terminal response is replayed. |

An endpoint with a non-transactional side effect must deduplicate that effect
under the mutation id. Existing uniqueness constraints remain an additional
duplicate-publication defense, not a substitute for recording the outcome.

Cargo retains a replayable representation of the body until the flow ends,
building the publish metadata bytes once so retries remain byte-identical. It
can keep that representation in memory or use file-backed storage; the protocol
does not require buffering a package archive in RAM. Request and archive hashes
can be computed incrementally while reading the replayable bytes.

Cargo can make a bounded retry after an interrupted or response-ambiguous
transport using the same mutation id. Receiving no complete response, or a
nonterminal `408`, `425`, `429`, `500`, `502`, `503`, or `504`, can trigger a
retry within the receive lease; Cargo honors a valid bounded `Retry-After`.
If an intermediary hid a committed origin response, the retry retrieves the
stored terminal outcome. Cargo does not automatically retry other complete
final responses. A normal protected publish uploads once; a transport retry
may upload again because Cargo cannot know how many bytes arrived, but the
logical mutation cannot execute twice.

Cargo will limit the number of consecutive challenges and bound its callback
parser, wait duration, and polling rate. These are denial-of-service defenses,
not wire compatibility guarantees. Implementations can tune them without a
protocol revision.

Registries separately limit preflights, upload bytes, verification attempts,
and polls. A preflight does not consume a successful-publication quota.

### Security and transport threat model

Version 1 requires an `https` registry API origin except for an explicitly
configured loopback origin. Cargo rejects other cleartext `http` origins. The
protocol depends on TLS server authentication and integrity ([RFC 8446]); an
attacker controlling TLS or the configured origin can replace instructions,
poll replies, and bearer credentials. Same-origin checks cannot protect a
compromised trust anchor.

Copying a complete challenge response, including its poll token, can reveal its
status and display metadata or prompt verification attempts, but cannot make it
ready or execute the mutation. Another credential binding or fingerprint
fails. A forged `ready` poll reply can only prompt Cargo to send the request,
which fails without the registry's server-side grant. Expired records also
fail.

An attacker who obtains the ready mutation id, the bound primary credential,
and the byte-identical request is indistinguishable from the original client.
Exact binding prevents substitution after preflight; idempotency prevents two
committed effects under that mutation id.

Exact binding begins at preflight. Malware controlling the publishing machine
before that point can modify an archive and cause Cargo to describe and bind the
modified bytes. A server-generated summary such as crate name and version does
not let the user distinguish those bytes from the package they intended to
publish. Mutation authorization therefore limits unattended use of an
exfiltrated credential and alteration after preflight; it does not establish
the integrity of a compromised client.

The callback fragment is not transmitted in the HTTP request or `Referer`, but
scripts in the verification document can read it. Callback-capable verification
pages follow the isolation requirements under Displaying instructions. Leaked
callback state permits a forged local wake-up, but Cargo still polls and the
registry's grant remains authoritative. Callback state is therefore neither a
primary credential nor mutation authority.

### Policy and rollout

Whether an account, credential, crate, or operation requires additional
authorization is registry policy. A registry can exempt non-interactive
credentials whose own model is short-lived and request-scoped, such as an OIDC
Trusted Publishing token. It does not infer an exemption only because Cargo is
running in CI.

Registries provide an opt-in or gradual rollout and name the minimum fully
supported Cargo release in `detail`. Unsupported clients do not bypass
enforcement.

## Drawbacks
[drawbacks]: #drawbacks

- Every mutation gains an authenticated preflight round trip.
- Interactive authorization can pause a Cargo mutation while the user completes
  registry-provided verification.
- Cargo gains a small loopback HTTP listener and a polling and retry state
  machine.
- Registries must store challenges and correctly fingerprint every mutation.
- Registries must serialize mutation execution and retain terminal responses
  for idempotent retries.
- A client that obtains a ready mutation id, its bound primary credential, and
  identical request bytes is indistinguishable from the original client. Exact
  binding limits what that combination can execute.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Authorization before publication

Returning publish success and holding the version for human review would add
a publication lifecycle with rules for index visibility, reservation,
replacement, cancellation, and expiration.

Preflight retains only the mutation descriptor and idempotency state, not the
archive. Returning an error after receiving the ordinary publish body would
repeat the upload without giving retries a stable identity. Retaining those
bytes would add temporary-blob ownership and cleanup without solving
idempotency for other mutations. A review-before-release workflow can be
designed independently.

### Registry-controlled verification

Registry-controlled verification avoids linking Cargo to WebAuthn, SSH,
hardware, or administrator APIs and lets policy change without a Cargo release.
Cargo cannot attest which factor was used.

A terminal OTP prompt would couple Cargo to a factor, while a credential
provider cannot uniformly react to policy chosen after the registry sees the
mutation descriptor. Asymmetric credentials reduce replay without necessarily
providing user presence. Mutation authorization composes with these mechanisms.

### Loopback callback with polling fallback

Version 1 includes both the loopback callback and the polling path. Polling
supports local and remote verification, including through SSH and restrictive
networks, but adds latency and registry load. The callback prompts an immediate
poll; retaining polling as the authority prevents callback failure or forgery
from stranding or authorizing the command. This deliberately avoids a second,
single-use proof whose consumption would conflict with interrupted-upload
retry.

RFC 8252 recommends attempting both IPv4 and IPv6 for OAuth native-app
callbacks. Version 1 deliberately fixes its callback to the IPv4 literal
because its callback object carries one URL. Unlike a callback-only OAuth
flow, an IPv4 failure is nonfatal here because polling remains active. A
future protocol version can advertise an address family if IPv6-only systems
warrant the additional wire surface.

### Structured preflight response

OAuth and HTTP challenges generally obtain or upgrade a credential. Here Cargo
retains its credential and authorizes one mutation. A pending preflight is a
successfully created asynchronous resource, so version 1 uses `202 Accepted`
rather than an OAuth Bearer challenge or an ordinary registry error. Errors on
the final endpoint still use the existing registry error format.

### Non-waiting preflight instead of challenge resumption

Cargo must know whether registry policy exempts a credential before deciding
that a noninteractive command cannot proceed. Skipping preflight in automatic
CI would lose that decision and could repeat a large direct publish only to
receive the protected endpoint's rejection.

Persisting a pending record across Cargo invocations would instead require
Cargo to protect a poll capability, mutation id, exact replayable request, and
remaining deadlines on disk, then define how a later invocation selects and
cleans up that state. `allow_pending: false` keeps the policy decision in the
preflight while ensuring that Cargo never displays instructions for a challenge
it cannot consume. Explicit `poll` or `loopback` remains the opt-in mechanism
when a noninteractive invocation really should wait.

### Compatibility with existing Cargo authentication RFCs

The advertised extension does not change RFC 2730 or RFC 3139 tokens, the
publish body, or existing endpoint authorization. Registries that do not
advertise it continue to receive current requests.

Cargo uses the intended mutation's existing credential-provider operation for
preflight, not a broader operation. A publish preflight therefore uses RFC
3231's `publish` descriptor with the crate name, version, and archive checksum.

After readiness, Cargo performs the normal credential lookup again before
sending the final mutation. It does not attempt to parse or validate an
arbitrary token itself. A provider can return a still-valid cached credential
according to its cache policy, while an expiring, operation-specific, or RFC
3231 provider can mint a fresh credential. The registry correlates the two
requests through the credential binding identity described under Credential
binding and readiness, then repeats ordinary authorization on the final
request. Polling is unauthenticated so Cargo does not send a publish-scoped or
single-use credential on a status `GET`.

Older Cargo versions ignore the new index key and send the mutation directly.
A registry can retain its reactive challenge path for those clients, in which
case a protected publish can still upload twice. A registry that requires
preflight instead returns an error naming the minimum Cargo version. This may
make a protected operation unavailable to an old client but does not reinterpret
an existing RFC.

## Prior art
[prior-art]: #prior-art

- npm can require interactive two-factor authentication for publication and
  package settings ([npm 2FA]). Its staged publishing workflow separately lets
  CI upload a package for later human approval ([npm staged publishing]). These
  demonstrate demand for human-authorized publication, while also showing the
  factor coupling and separate package lifecycle that this RFC avoids.
- RubyGems supports WebAuthn or OTP for `gem push`, `gem yank`, and owner changes
  ([RubyGems MFA]). This demonstrates that additional authorization applies
  beyond publication; its CLI-visible factor handling is what this
  registry-neutral protocol avoids.
- OAuth Step Up Authentication ([RFC 9470]) uses `401` and
  `WWW-Authenticate` to obtain a token associated with stronger or more recent
  authentication. Its policy is relevant, while this proposal creates a
  mutation-specific server-side grant and can represent another party's
  approval.
- OAuth Device Authorization ([RFC 8628]) displays verification instructions
  and polls at a recommended interval. This proposal binds authorization to an
  existing authenticated mutation and also supports a loopback callback.
- OAuth 2.0 for Native Apps ([RFC 8252]) specifies loopback callback security
  considerations. This proposal similarly uses an IP literal, an ephemeral
  port, client-held state, and a listener bound only to loopback, while
  retaining polling when an IPv4 callback is unavailable.

[npm 2FA]: https://docs.npmjs.com/requiring-2fa-for-package-publishing-and-settings-modification/
[npm staged publishing]: https://docs.npmjs.com/staged-publishing/
[RubyGems MFA]: https://guides.rubygems.org/setting-up-multifactor-authentication/
[RFC 9470]: https://www.rfc-editor.org/rfc/rfc9470.html
[RFC 8628]: https://www.rfc-editor.org/rfc/rfc8628.html
[RFC 8252]: https://www.rfc-editor.org/rfc/rfc8252.html
[RFC 8446]: https://www.rfc-editor.org/rfc/rfc8446.html
[RFC 9110]: https://www.rfc-editor.org/rfc/rfc9110.html
[Cargo credential provider protocol]: https://doc.rust-lang.org/cargo/reference/credential-provider-protocol.html
[Cargo Registry Web API]: https://doc.rust-lang.org/cargo/reference/registry-web-api.html
[CWE-150]: https://cwe.mitre.org/data/definitions/150.html
[Unicode UTR #36]: https://www.unicode.org/reports/tr36/tr36-15.html

## Conformance cases

Cargo and registry implementations cover at least these cases in protocol
tests. The list is normative as to the required result, not the test-harness
structure:

1. Concurrent identical preflights with different `preflight_id` and callback
   URLs create distinct records; a transport retry with one `preflight_id`
   returns its original record and rejects changed `allow_pending` or callback
   data.
1. A noninteractive automatic preflight sets `allow_pending` to `false`; an
   exempt credential can receive `ready`, while a policy requiring interaction
   receives `interaction_required` without creating a mutation record or
   displaying instructions for an abandoned challenge.
1. A credential provider that returns a new `cache: never` token succeeds only
   when both tokens validate to the same credential binding identity; otherwise
   the final request fails without changing the record.
1. A callback and scheduled poll racing after readiness cause one immediate
   continuation, and a forged callback while pending causes only another poll.
1. An interrupted upload can retry within its receive lease; receive-lease
   expiry performs no mutation and stores no terminal response.
1. A mutation committed immediately before response loss is not repeated, and
   the stored response remains replayable after all earlier deadlines pass.
1. A request begun before grant expiry can finish within its receive lease; a
   first request begun after grant expiry cannot claim the record.
1. Altering the body, content type, content encoding, request target, crate,
   owner list, archive, or declared length fails without execution or a stored
   terminal outcome.
1. A mismatched request arriving before the legitimate request cannot poison
   the record.
1. `429` and transient `5xx` responses remain nonterminal; a deterministic
   application `4xx` after an exact authorized match can be replayed.
1. A protected direct mutation from an old or nonconforming client fails before
   any effect, regardless of extension advertisement handling.
1. Credential revocation or binding mismatch after readiness fails without
   executing or terminalizing the record.
1. An API base URL containing a path prefix produces matching preflight and
   final request targets without discarding that prefix.
1. IPv4 loopback failure leaves polling usable, including when `loopback` was
   explicitly selected.
1. Poll redirects, cross-origin or oversized URLs, oversized bodies, CR, bidi
   controls, and terminal escapes are rejected or sanitized as specified.
1. Repeated pending polls cannot extend Cargo's original challenge deadline.
1. Poll timeouts and `408`, `425`, `429`, `500`, `502`, `503`, and `504`
   responses retry with bounded backoff until a later `ready` response or the
   original monotonic deadline; redirects and deterministic errors stop.
1. Under `--quiet`, challenge instructions, the bounded-wait line, and terminal
   errors remain visible while routine status and animation are suppressed.
1. Channel selection follows flag, environment, selected-registry
   configuration, then `auto`; explicit `poll` or `loopback` can wait in CI
   while automatic mode cannot create a pending record there.
1. Ctrl-C winning the wait-to-send race emits a no-mutation-sent result and no
   final request; a send winning the race never produces that assurance.
1. Owner invitations, notifications, audit entries, index changes, object
   writes, cache invalidations, and webhooks are transactionally committed or
   deduplicated under the mutation id.
1. A listed operation with no mutually supported version fails before any
   authenticated request; an unlisted operation retains ordinary Cargo
   behavior.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None for version 1.

## Future possibilities
[future-possibilities]: #future-possibilities

Later versions could add other verification mechanisms. A separate staged
publication protocol could define review and release after an upload is
accepted. New registry mutation endpoints can opt into mutation authorization
while retaining exact request binding.

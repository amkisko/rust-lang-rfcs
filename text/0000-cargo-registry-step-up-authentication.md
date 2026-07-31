- Feature Name: `cargo_registry_step_up_authentication`
- Start Date: 2026-07-31
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Cargo Issue: [rust-lang/cargo#0000](https://github.com/rust-lang/cargo/issues/0000)

## Summary
[summary]: #summary

Define a Cargo registry protocol for additional authorization of an already
authenticated mutation. Cargo will first send a small mutation preflight. A
registry can return a structured `step_up_required` error with human-readable
instructions and a polling URL. Cargo will wait for the `acknowledged` state,
then send the exact mutation under the preflight's idempotency key. Cargo does
not implement an authentication factor; each registry can use a passkey,
hardware security key, or another method. Step-up is an authorization
precondition, not a publication state: the preflight does not stage or reserve
a package, and a protected publish remains unsuccessful until the registry has
verified any required fresh authentication, authorized its exact contents, and
completed the mutation. In this RFC, *step-up authentication* is fresh
user-presence authentication after the registry has accepted the primary
credential; a *challenge* is a pending authorization request bound to one exact
mutation; `acknowledged` means the registry's chosen authentication has been
verified, not that the mutation succeeded; and a server-side *grant* or
one-time callback *proof* authorizes the bound mutation after verification.

## Motivation
[motivation]: #motivation

Cargo commonly authenticates registry mutations with a long-lived API token.
Scopes and crate restrictions limit a token's authority, and credential
providers improve its storage, but a token with publish authority can still be
used without the user's presence. Malware, a leaked CI secret, or an exposed
backup can therefore lead directly to a package supply-chain compromise.

Website account MFA does not address this threat because Cargo uses a
separately issued credential. Existing Cargo authentication RFCs improve
credential storage, registry authentication, and replay resistance, but they
do not let a registry require fresh step-up authentication for an exact
mutation descriptor ([RFC 2730], [RFC 3139], [RFC 3231]).

A registry can put verification instructions in an ordinary error, but Cargo
cannot then wait safely or know when to continue. Users must rerun commands
manually and wrappers must parse prose. Retrying a publish can also repeat a
large upload or leave the client uncertain whether the mutation succeeded.

The common client behavior belongs in Cargo. Cargo will be able to preflight an
exact mutation, validate the polling URL, avoid hanging CI, and safely retry
under an idempotency key. The registry remains responsible for obtaining user
presence, mutation records, exact request binding, recovery, and policy. This
division lets alternative registries provide the same protection without
adopting crates.io accounts or passkeys.

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
       Note Additional authentication is required. Visit https://crates.io/verify/stp_abc123.
    Waiting for additional authentication
       Note verification complete; continuing
   Uploading example v1.2.3 (/work/example)
   Uploaded example v1.2.3 to registry `crates-io`
```

Cargo will package the crate once and hash the complete ordinary publish
request body. The preflight will contain that digest plus the crate, version,
request size, archive size, and archive digest, but not the archive bytes. The
registry can identify and bind the exact operation, but it has not accepted,
published, indexed, or reserved the version. Reaching `acknowledged` authorizes
Cargo to send the mutation. Only that request can produce the normal publish
success response.

Cargo will print the registry's instructions rather than collecting a password,
passkey assertion, OTP, or other factor. The registry chooses how to obtain
user presence and displays the exact operation being authorized. It might ask
the user to visit a website, run an SSH command, use a hardware device, or
obtain an administrator's confirmation.

A protected yank, unyank, or owner change follows the same visible workflow.
For example, `cargo yank --version 1.2.3 example` can pause with instructions to
authorize "Yank example 1.2.3", while `cargo owner --add alice example` can
pause to authorize the exact owner addition. After acknowledgment, Cargo sends
only the bound operation under its mutation id; the user does not manually
rerun the command.

On an interactive workstation, Cargo will request both a loopback callback and
a polling fallback. The callback makes local completion immediate. Polling
also works when verification happens on another machine, such as during an SSH
session:

```console
$ CARGO_REGISTRY_STEP_UP_CHANNEL=poll cargo publish
```

`CARGO_REGISTRY_STEP_UP_CHANNEL` accepts `auto`, `localhost`, `poll`, and
`disabled`. `auto` is the default. An explicit `localhost` or `poll` opts into
waiting; `disabled` prints `detail` and exits. In automatic mode, Cargo also
exits instead of waiting when standard input is not a terminal or `CI` is
`true` or `1`.

Older Cargo versions display the human-readable registry error and exit. The
error includes the complete verification instructions, so a registry can
support verification followed by a manual rerun. Older clients cannot automate
polling or the callback.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Scope and initial request

The protocol applies to authenticated Cargo registry Web API mutations. It does
not change downloads, registry indexes, `cargo login`, website sessions, or the
primary credential format.

Supporting registries will advertise `"step-up-auth": 1` in index `config.json`.
Older Cargo versions ignore the unknown key and continue using the existing Web
API. A registry does not advertise version 1 until it can accept both preflight
and idempotent mutation requests.

Before a mutation, Cargo will send an authenticated `POST` to
`/api/v1/auth/challenges`. The request describes the exact ordinary HTTP
mutation without performing it. For publish it contains:

```json
{
  "protocol_version": 1,
  "operation": "publish",
  "method": "PUT",
  "endpoint": "/api/v1/crates/new",
  "crate": "example",
  "version": "1.2.3",
  "request_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "request_size": 124678,
  "archive_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "archive_size": 123456
}
```

`request_sha256` is the SHA-256 of the raw bytes Cargo will send as the final
request body. Hashing raw bytes avoids a cross-implementation canonical-JSON
algorithm. The registry treats every descriptive field as untrusted during
preflight. It validates their syntax and uses them for ordinary credential
scope and the server-generated operation summary, but does not accept them as
crate contents. The parsed final request is authoritative and must agree with
the declared crate, version, archive digest, and sizes.

For yank, unyank, and owner changes, the preflight similarly contains the HTTP
method, endpoint, raw request-body digest and size, and the crate, version,
direction, and complete owner list as applicable. A bodyless request uses the
SHA-256 of the empty byte string. The registry computes its mutation
fingerprint from the validated descriptor, including method, endpoint, and raw
body digest.

In `auto` or `localhost` mode, Cargo will first bind an ephemeral IPv4 listener
on `127.0.0.1` and add both headers to the preflight:

```http
Cargo-Step-Up-Port: 49152
Cargo-Step-Up-Callback-Secret: <opaque URL-safe value>
```

The port is a decimal integer from 1024 through 65535. Cargo generates the
callback secret with a cryptographically secure random number generator and at
least 128 bits of entropy, encoded as 32 through 128 URL-safe characters.

The headers form a pair. A registry receiving only one rejects the preflight.
Cargo retains the plaintext callback secret; the registry stores only a
one-way hash. The registry does not copy or otherwise reflect the plaintext
secret into `detail`, a URL, or any response body. The callback secret and later
proof are credentials, so clients, registries, proxies, and error reporting
systems redact them from logs.

Cargo will omit both in `poll` and `disabled` modes.

The registry validates the primary credential and ordinary operation scope
during preflight. It atomically reuses an unexpired mutation record for the
same credential identity, method, endpoint, and fingerprint, including across
concurrent preflights. Otherwise it creates a record with a fresh,
unpredictable `challenge_id`. That identifier is also the idempotency key for
the final mutation.

For credential-provider and RFC 3231 purposes, preflight requests use the same
operation descriptor as the intended mutation. A publish preflight therefore
requests a `publish` credential with the name, version, and archive checksum
already defined by RFC 3231. Cargo obtains a fresh operation credential before
the final mutation when the provider does not return an operation-independent
cached credential. The registry binds short-lived or asymmetric credentials by
an equivalently narrow stable identity rather than requiring identical token
bytes.

If no additional authentication is required, the registry responds with a
non-cacheable ready record:

```json
{
  "status": "acknowledged",
  "challenge_id": "stp_abc123",
  "expires_at": "2026-07-31T12:05:00Z"
}
```

Cargo will then perform the mutation immediately. If additional authentication
is required, the registry returns the challenge response below.

### Challenge response

When preflight requires additional authentication, the registry responds with
`403 Forbidden` and the usual registry error envelope. The response includes
`Cache-Control: no-store` so challenge state is not retained by HTTP caches:

```json
{
  "errors": [{
    "detail": "Additional authentication is required. Visit https://registry.example/verify/stp_abc123 and retry this command.",
    "id": "step_up_required",
    "protocol_version": 1,
    "challenge_id": "stp_abc123",
    "operation": "publish",
    "crate": "example",
    "operation_summary": "Publish example 1.2.3",
    "poll_url": "https://registry.example/api/v1/auth/challenges/stp_abc123",
    "expires_at": "2026-07-31T12:05:00Z",
    "recommended_poll_interval_secs": 5
  }]
}
```

The required version 1 fields are `detail`, `id`, `protocol_version`,
`challenge_id`, and `poll_url`. Cargo will recognize the challenge only when
all five have the expected JSON types, `id` is `step_up_required`, and
`protocol_version` is `1`. Otherwise it reports an ordinary registry error.
All other members are optional, and unknown fields are ignored.

`detail` contains complete instructions for satisfying the challenge and
manually retrying the operation. Cargo will not execute any command it contains.
When callback headers were supplied, the registry returns any callback-capable
same-origin URL in `detail` without the callback secret or a URL fragment.
Cargo will add this client-held fragment to the first same-origin URL locally
before displaying the instructions:

```text
https://registry.example/verify/stp_abc123#callback_secret=<callback-secret>
```

The fragment is not included in the user agent's HTTP request. A verification
page can read it locally, then return the secret to the registry when completing
or recovering callback delivery. The registry compares the secret with its
stored hash. If `detail` has no same-origin URL, Cargo displays it unchanged and
polling remains available. Cargo rejects callback augmentation when the selected
URL already contains a fragment.

`operation`, `crate`, and `operation_summary` are optional display and
diagnostic context. Cargo does not require any of them, does not use them to
authorize the mutation or construct URLs, and version 1 does not consume
`operation_summary`.

`challenge_id` identifies the canonical mutation record returned by preflight;
Cargo will send it as the final request's idempotency key.

`expires_at` is an optional RFC 3339 timestamp. Cargo will wait for at most five
minutes and uses that duration when the field is missing or invalid.
`recommended_poll_interval_secs` is advisory. Registries send a value from 1
through 30 seconds; Cargo defaults to 5 and defensively clamps other values.

New implementations return `403`. For compatibility with registry APIs that
already encode errors this way, Cargo can also recognize a complete challenge
in a registry error body returned with a successful HTTP status. Cargo still
treats the mutation as unsuccessful and does not begin post-publish index
observation.

`poll_url` has the same origin as the registry API URL from index `config.json`,
including scheme and effective port. Cargo rejects the handshake if it differs.
The poll request does not follow redirects.

### Request binding and acknowledgment

The registry binds a challenge and the resulting authorization to:

- the authenticated account and exact credential, or an equivalently narrow
  credential identifier;
- the HTTP method and canonical API endpoint;
- the operation and declared crate, version, or owners; and
- the raw request-body digest and length.

For publish, the fingerprint additionally covers the declared archive SHA-256
and archive size. For yank and unyank it covers the crate and version. For
owner changes it covers the crate, direction, and complete owner list.

The registry shows a server-generated summary of this bound operation through
its chosen verification mechanism. Receiving or viewing the instructions alone
does not complete verification. The registry verifies the required
user-presence properties before setting the challenge to `acknowledged`. A
broad authorization window is insufficient because an attacker holding the
token could substitute another mutation.

The registry does not perform the mutation or receive archive bytes before
verification. A challenge is pending authorization, not an accepted or staged
package version. It reuses an unexpired mutation record for the same credential
and fingerprint.

### Polling and completion

Cargo will poll the unguessable `poll_url` without the primary authorization
credential, will not follow redirects, and will expect a non-cacheable
response:

```json
{ "status": "pending", "recommended_poll_interval_secs": 5 }
```

Version 1 defines `pending` and `acknowledged`. Cargo will continue waiting for
`pending` and send the mutation after `acknowledged`. A `404`, an unknown
status, or any other error stops the flow. The registry can update the advisory
interval in a poll response.

Treating `poll_url` as a short-lived read-only capability avoids replaying a
publish-scoped or single-use credential on a `GET`. Possession can reveal only
the challenge status and cannot acknowledge or execute the mutation.

When the registry sets the challenge to `acknowledged`, it marks the mutation
record ready and creates a short-lived server-side grant bound to the exact
credential and mutation fingerprint. The grant cannot authorize an altered
request or another operation. Keeping this result server-side avoids
transferring another credential.

When callback headers were supplied, the verification mechanism returns the
fragment-held callback secret to the registry during callback completion or
recovery. The registry compares its hash before returning a loopback URL
containing the proof and stored port, but no `state` parameter. The local
verification client adds `state` and requests:

```text
http://127.0.0.1:<port>/?code=<proof>&state=<callback-secret>
```

The registry generates the proof with at least 128 bits of entropy and encodes
it as 22 through 128 URL-safe characters. It is opaque, single-use,
short-lived, and bound like the polling grant. Cargo accepts only a bounded
HTTP/1.x `GET` from a loopback peer, compares `state` with its callback secret,
then sends the mutation with:

```http
Cargo-Mutation-Id: stp_abc123
Cargo-Step-Up-Proof: <proof>
```

The registry atomically consumes a valid proof. It creates the server-side grant
even when a callback was requested, and Cargo will wait for the callback and
polling concurrently. Either can therefore complete the same challenge without
making callback failure fatal. The registry server never connects to the
supplied loopback port.

### Idempotent mutation and implementation limits

After the challenge reaches `acknowledged`, Cargo will send the ordinary
request with its existing authorization header and:

```http
Cargo-Mutation-Id: stp_abc123
```

The registry computes the raw request-body SHA-256 and length while receiving
the final request. For publish, it also parses and validates the ordinary
publish body and computes the archive SHA-256 and size. It rejects the request
unless the method, endpoint, raw body, parsed crate and version, and archive
facts all match the preflight record. Other operations are compared with their
preflight descriptors in the same way. The registry also repeats ordinary
authorization and validation; step-up does not replace the primary credential
or enlarge its scopes.

The mutation id is idempotent for its credential identity and fingerprint. The
registry serializes concurrent requests using the same record. Exactly one
request performs the mutation; concurrent or later identical requests receive
the stored terminal response. Reusing a mutation id with different request
fields fails without performing either mutation. A registry retains the
terminal response for at least the mutation record's advertised lifetime.

The registry does not claim execution until it has received and matched the
complete request. A dropped upload can therefore be retried with the same
mutation id. Once execution begins, the registry records the endpoint's normal
status, response body, and security-relevant response headers with the mutation
outcome. Mutation effects and the successful outcome record must be committed
atomically where the endpoint's storage model permits it; existing uniqueness
constraints remain the final defense against duplicate publication.

Cargo will retain a replayable request body until the flow ends. For publish, it
will build and buffer the body once so transport retries contain identical
metadata and archive bytes. Cargo can automatically make a bounded retry after
an interrupted or response-ambiguous transport, always with the same mutation
id. A normal protected publish uploads the archive only once. A transport retry
may upload it again, because Cargo cannot know how many bytes reached the
registry, but it cannot execute the mutation twice.

Cargo will limit the number of consecutive challenges and bound its callback
parser, wait duration, and polling rate. These are denial-of-service defenses,
not wire compatibility guarantees. Implementations can tune them without a
protocol revision.

Registries separately limit preflights, upload bytes, verification attempts,
and polls. A preflight does not consume a successful-publication quota.

### Policy and rollout

Whether an account, credential, crate, or operation requires step-up is
registry policy. A registry can exempt non-interactive credentials whose own
model is short-lived and request-scoped, such as an OIDC Trusted Publishing
token. It does not infer an exemption only because Cargo is running in CI.

Registries provide an opt-in or gradual rollout and name the minimum fully
supported Cargo release in `detail`. Unsupported clients do not bypass
enforcement.

## Drawbacks
[drawbacks]: #drawbacks

- Every mutation gains an authenticated preflight round trip.
- Interactive step-up can pause a Cargo mutation while the user completes
  registry-provided verification.
- Cargo gains a small loopback HTTP listener and a polling and retry state
  machine.
- Registries must store challenges and correctly fingerprint every mutation.
- Registries must serialize mutation execution and retain terminal responses
  for idempotent retries.
- Two clients with the same stolen credential and identical request bytes are
  indistinguishable. Exact binding prevents substitution, not duplication.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Authorization before publication

Returning publish success and holding the version for human review would add
a separate publication lifecycle. Cargo would begin observing the index while
the registry still considered the version pending. The design would need rules
for visibility, reservation, replacement, cancellation, and expiration.

Preflight retains only the mutation descriptor and idempotency state, not the
archive. Returning an error after receiving the ordinary publish body would
repeat the large upload after acknowledgment and would not give retries a
stable execution identity. Privately retaining those bytes would add temporary
blob ownership and cleanup without solving generic mutation idempotency. A true
review-before-release workflow can be designed independently.

### Registry-controlled verification

Keeping verification behind the registry protocol avoids selecting a factor
protocol or linking Cargo to WebAuthn, SSH, hardware, or administrator
confirmation APIs. Registries can change their policy without a Cargo release.
Cargo cannot attest which factor was used, but that is already a registry
decision.

### Polling with an optional callback

Polling works with local and remote verification mechanisms, including through
SSH and restrictive networks, but it is slower and adds registry load. A
callback improves local responsiveness. Keeping polling available for every
challenge prevents callback restrictions or remote verification from stranding
the command.

### Structured registry error

OAuth and HTTP authentication challenges generally obtain or upgrade a
credential. Here Cargo retains the same credential and authorizes one exact
mutation. A structured registry error also gives older clients useful manual
instructions. The protocol therefore does not claim OAuth Bearer semantics.

### Compatibility with existing Cargo authentication RFCs

The protocol is an advertised extension to the registry Web API. It does not
change the meaning or storage of tokens from RFC 2730 and RFC 3139, the
existing publish body, or the authorization rules of an existing mutation
endpoint. Registries that do not advertise `step-up-auth` continue to receive
the current requests.

RFC 3231 already defines a `publish` credential request containing the crate
name, version, and archive checksum. Cargo uses that same operation descriptor
for the authenticated publish preflight. The preflight is not a new, broader
credential-provider operation. Cargo may request another `publish` credential
for the final request, and the registry binds both requests through the stable
credential identity and mutation fingerprint. Polling is deliberately
unauthenticated so Cargo does not send a publish-scoped or single-use RFC 3231
credential on a status `GET`.

Older Cargo versions ignore the new index key and send the mutation directly.
During rollout, a registry can retain its existing reactive challenge path for
those clients; a legacy protected publish can still upload twice. A registry
that requires preflight instead returns an actionable error naming the minimum
Cargo version. This can make a newly protected operation unavailable to an old
client, but does not reinterpret or weaken an existing RFC.

### Other authentication mechanisms

A terminal OTP prompt couples Cargo to a factor and does not naturally support
passkeys. A credential provider can launch its own interactive flow while
obtaining the primary credential, but it cannot uniformly react to policy
chosen after the registry sees a mutation body. Asymmetric tokens reduce
credential replay without necessarily requiring human presence. Step-up
composes with these mechanisms rather than replacing them.

## Prior art
[prior-art]: #prior-art

- npm can require interactive two-factor authentication for publication and
  package settings ([npm 2FA]). Its staged publishing workflow separately lets
  CI upload a package for later human approval ([npm staged publishing]). These
  demonstrate demand for human-authorized publication, while also showing the
  factor coupling and separate package lifecycle that this RFC avoids.
- RubyGems supports WebAuthn or OTP for `gem push`, `gem yank`, and owner changes
  ([RubyGems MFA]). This demonstrates that step-up applies beyond publication;
  its CLI-visible factor handling is what this registry-neutral protocol avoids.
- OAuth Step Up Authentication ([RFC 9470]) uses `401` and
  `WWW-Authenticate`. Its terminology is relevant, while this proposal uses a
  `403` registry error and does not obtain a replacement OAuth token.
- OAuth Device Authorization ([RFC 8628]) displays verification instructions
  and polls at a recommended interval. This proposal binds authorization to an
  existing authenticated mutation and also supports a loopback callback.

[npm 2FA]: https://docs.npmjs.com/requiring-2fa-for-package-publishing-and-settings-modification/
[npm staged publishing]: https://docs.npmjs.com/creating-and-publishing-scoped-public-packages/#staged-publishing
[RubyGems MFA]: https://guides.rubygems.org/setting-up-multifactor-authentication/
[RFC 9470]: https://www.rfc-editor.org/rfc/rfc9470.html
[RFC 8628]: https://www.rfc-editor.org/rfc/rfc8628.html

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None for version 1.

## Future possibilities
[future-possibilities]: #future-possibilities

Later versions could add explicit denial and expiration states or other
verification mechanisms. A separate staged publication protocol could define
review and release after an upload is accepted. New registry mutation endpoints
can opt into step-up while retaining exact request binding.

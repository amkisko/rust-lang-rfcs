- Feature Name: `cargo_registry_loopback_callback`
- Start Date: 2026-08-01
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Cargo Issue: [rust-lang/cargo#0000](https://github.com/rust-lang/cargo/issues/0000)
- crates.io issue: [rust-lang/crates.io#0000](https://github.com/rust-lang/crates.io/issues/0000)

## Summary
[summary]: #summary

Optionally extend [Cargo registry mutation authorization] with the
`loopback-callback` extension. A registry verification page can wake a waiting
local Cargo process, but readiness still comes only from the core poll
capability and server-side grant.

The core is already complete for poll-only terminals, CI, and SSH. This
proposal adds only a local-interactive latency optimization and its browser
isolation requirements. It never redefines how a record becomes `ready`.

## Motivation
[motivation]: #motivation

Polling alone works everywhere but adds completion latency and load. On a local
interactive workstation a browser can notify Cargo immediately after
verification. A callback carrying mutation authority would create a second
authorization path whose loss, replay, and consumption interact badly with
final-request retry. A wake-only callback avoids that authority split.

Browser delivery exposes the operation summary and registered loopback URL to
scripts executing in the verification document. A callback-capable page must
therefore isolate the flow from third-party active content.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Cargo can request `loopback-callback` for interactive `auto`. It binds an ephemeral listener on
`127.0.0.1` before preflight, adds `loopback-callback` to
`requested_extensions`, and registers an exact callback URL containing random
wake-up state:

```text
http://127.0.0.1:49152/cargo/registry-authorization?state=<random>
```

The registry includes its verification page URL in ordinary `detail` text.
After the user completes verification, that page obtains the registered
callback URL from the stored record and requests it unchanged.

Cargo compares the state and polls immediately. A forged callback merely causes
an extra poll; it cannot create the registry's grant.

In the common local flow, the maintainer runs `cargo publish`, follows the
registry link in a browser on the same machine, and verifies the operation.
The completion page wakes Cargo, Cargo confirms `ready` through the normal poll
URL, and upload starts without waiting for the next scheduled poll. When the
browser is on another machine, as in many SSH sessions, the maintainer selects
`poll` and the same authorization succeeds without a callback.

If binding or delivery fails, polling continues. Explicit `poll` omits the
callback, which is suitable when a browser runs on another machine. Explicit
`loopback` requests the optimization but still retains polling as fallback.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Selection

Cargo sends callback metadata only when it requests the extension for that
preflight. A registry that implements the extension stores and echoes it in the
record's `active_extensions` set. Cargo uses the listener only after verifying
that echo. Core defines `auto`, `poll`, and `disabled`, including their
configuration and precedence. This extension adds an explicit `loopback` value.

A registry must accept a core-only preflight with an empty
`requested_extensions` array and no `callback` object. Implementing the
extension never makes callback metadata mandatory for other clients or modes.

Interactive `auto` requests loopback when local binding succeeds and otherwise
uses core polling. If the registry does not activate it, `auto` closes the
listener and polls. Explicit `poll` never binds or sends callback metadata.
Explicit `loopback` sets core `allow_pending` to true, retains polling as a
transport fallback, and fails if the response does not activate the extension.

### Callback registration

Cargo binds an ephemeral IPv4 listener on literal `127.0.0.1`, never `0.0.0.0`,
before sending preflight. It adds:

```json
{
  "callback": {
    "url": "http://127.0.0.1:49152/cargo/registry-authorization?state=0123456789abcdefghijklmnopqr"
  }
}
```

The URL uses exactly:

- scheme `http`;
- IPv4 literal `127.0.0.1`;
- an explicit decimal port from 1024 through 65535;
- path `/cargo/registry-authorization`;
- exactly one `state` query parameter containing 22 through 128 URL-safe ASCII
  characters;
- no user information or fragment.

The state contains at least 128 random bits and prevents unrelated local pages
from guessing a valid wake-up request. The registry validates and stores the
exact URL as delivery metadata but never connects to it. Retrying one core
`preflight_id` for which `loopback-callback` is active requires a byte-identical
callback object. Changes consisting only of ignored unknown extensions do not
change the record. A callback object without a requested `loopback-callback`
extension is invalid, and Cargo requires the response to confirm activation.

If IPv4 loopback is unavailable, Cargo closes any partial listener state and
uses polling. RFC 8252 recommends trying both address families for OAuth native
apps; this extension fixes one URL to IPv4 because polling is always an
authoritative fallback. A future extension can add IPv6 support.

The listener exists only for the pending wait and closes before Cargo sends the
final mutation.

### Verification and callback delivery

The registry supplies complete user instructions only through core `detail`.
Cargo displays that bounded inert text under the core sanitization and external
URL-labeling rules.

The verification document reads its operation summary and callback URL from
the stored record. Only after successful authorization does it navigate to or
load the exact registered callback URL unchanged. It sends no protocol
credential.

### Verification document isolation

A callback-capable verification URL is served directly without an HTTP or
script redirect and executes no third-party active content.

Its response applies a dedicated restrictive Content Security Policy. A
static application may deliver more than one policy; in that case these
requirements apply to their combined enforcement. At minimum:

- `default-src 'none'`;
- script is limited to first-party code needed by the verification page,
  inline script requires a nonce or hash, and `unsafe-eval` is absent;
- `connect-src` and `form-action` are limited to the registry origin;
- callback delivery can reach only the registry origin and IPv4 loopback;
- `base-uri 'none'` and `object-src 'none'`;
- `frame-ancestors 'none'`.

Static CSP cannot express the callback's dynamic port and state. Immediately
before delivery, first-party code validates the exact stored URL shape and uses
that URL unchanged. Equivalent or stricter isolation is conforming. CSP is
defense in depth: a script intentionally allowed by policy can read the
operation and callback URL, so third-party script must not execute in this
document.

The verification UI describes completion as “authorized.” Cargo still must send
the final request.

Plain WebAuthn proves control of a relying-party-scoped credential and optional
user verification. It does not provide a trusted browser display or signature
over arbitrary mutation text. Registries do not claim transaction-signing
semantics unless a separate mechanism actually supplies them.

### Cargo callback server

Cargo accepts only a bounded HTTP/1.x `GET` from an IPv4 loopback peer.
It requires the exact path and exactly one `state` value. The request line,
individual header lines, combined headers, and per-connection read time are
bounded.

Cargo compares state in constant time. Invalid methods, paths, state, peers,
oversized requests, partial requests, and unsolicited connections receive a
small error response and do not end the wait. A valid request receives a
minimal non-cacheable response suitable for browser delivery.

A valid callback schedules one immediate poll and does not change the record.
Callback, scheduled poll, timeout, and cancellation races retain the core wait
semantics. Only a `ready` poll can start the final request.

Callback failure, browser blocking, listener shutdown, or forged wake-ups leave
scheduled polling usable until the original monotonic deadline.

### Logging and privacy

Cargo and registry logs redact the callback URL because it contains wake-up
state. Registries avoid retaining it longer than the mutation record.

Verification pages avoid analytics, third-party error reporting, or browser
extensions they control through injected scripts. The protocol cannot protect
against a compromised browser, registry origin, Cargo process, or operating
system.

## Drawbacks
[drawbacks]: #drawbacks

- Cargo implements a small bounded HTTP server.
- Callback-capable verification needs a separately hardened page and CSP.
- IPv4-only loopback can fail on unusual systems, though polling remains.
- Browser privacy tools can block callback delivery and add one poll interval.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Polling remains authoritative because it works remotely and keeps callback
delivery outside the authorization path. A wake-only callback is safe to
duplicate, lose, or forge.

Using `localhost` is more vulnerable to name-resolution and address-family
surprises than the literal loopback address. Fixing an exact path and port
range reduces local cross-protocol interaction.

Callback state prevents unrelated websites from reliably waking Cargo but is
not treated as secret mutation authority. This permits simple browser delivery
without weakening the registry grant.

## Prior art
[prior-art]: #prior-art

- [RFC 8252] defines literal-IP loopback redirects and ephemeral ports for
  native applications.
- [RFC 8628] establishes polling as sufficient when the authorization device
  has no return channel.
- [Content Security Policy Level 3] defines the document isolation directives
  used here.

[RFC 8252]: https://www.rfc-editor.org/rfc/rfc8252.html
[RFC 8628]: https://www.rfc-editor.org/rfc/rfc8628.html
[Content Security Policy Level 3]: https://www.w3.org/TR/CSP/

## Appendix: conformance cases

1. `auto` requests loopback only when locally interactive and uses it only when
   the registry activates it.
2. Explicit `poll` never binds or sends callback metadata.
3. Explicit `loopback` fails when the registry does not activate the extension.
4. The listener binds literal `127.0.0.1` and the registered URL has the exact
   required shape.
5. A preflight retry cannot replace its callback URL.
6. The callback URL contains one random state value and is stored exactly.
7. The verification document has no third-party active content.
8. Forged or repeated callbacks cause at most another poll.
9. A callback only schedules a poll; only a `ready` poll can begin the final
   request.
10. Callback failure leaves polling usable.
11. Invalid peer, method, path, state, size, or timeout does not end the wait.
12. Verification copy describes the operation as authorized.
13. A core-only or explicit-poll client activates no callback extension and the
    registry does not require callback metadata.
14. Cargo uses a listener only after the preflight response echoes
    `loopback-callback` as active.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should Cargo stabilize this optimization with the poll-based core or only
  after registries have deployed callback-page CSP and isolation monitoring?
- Is IPv4-only loopback acceptable for the first stable version, or should the
  initial extension define equivalent IPv6 listener and URL-selection rules?

## Future possibilities
[future-possibilities]: #future-possibilities

A future extension can add IPv6 loopback, OS deep links, or another
authenticated wake-up transport while retaining polling as authority.

[Cargo registry mutation authorization]: 0000-cargo-registry-mutation-authorization.md

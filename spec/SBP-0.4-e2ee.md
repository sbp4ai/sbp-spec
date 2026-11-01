# SBP-0.4 — End-to-end encryption over an untrusted relay

Status: proposed · Depends on: SBP-0.1 (envelope), SBP-0.3 (multi-server)

## 1. Threat model

| Party | Trust | Knows |
|---|---|---|
| **Client** (web console / mini program) | trusted | its own `(box_sk, sign_sk)`, `rk`, peer public keys, server URL |
| **Device** (sbp-agent on the user's ECS/home machine) | trusted | its own `(box_sk, sign_sk)`, `rk`, peer public keys |
| **Relay** (`sbp-dispatcher`) | **untrusted** | public keys only, opaque routing tags, ciphertext, signatures |

The relay is a **curious-but-passive** adversary: it stores and forwards everything that
crosses it, and may try to read prompt text, conversation history, and session metadata.
It must **not** be able to:

- decrypt any `payload` (no symmetric secret, no private key on the relay),
- learn which `sid`/`did` a frame belongs to (routing is blinded),
- forge a frame on behalf of a client or device (it lacks the signing key).

The relay MAY still observe metadata: timing, frame sizes, which endpoints are online,
and that *some* traffic happened. Cover-traffic is out of scope for v0.4.

## 2. Primitives

| Use | Algorithm | Notes |
|---|---|---|
| Key agreement | **X25519** (Curve25519) | via libsodium `crypto_box` |
| Bulk encryption | **XSalsa20-Poly1305** | inside `crypto_box`; 24-byte nonce |
| Authentication | **Ed25519** | sign/verify challenge + frame signature |
| Routing blind | **HMAC-SHA256** | `tag = HMAC(rk, scope)` |
| Key derivation (optional) | **Argon2id** | derive `rk` from a passphrase |

Each endpoint holds two keypairs:

- **box keypair** `(box_pk, box_sk)` — X25519, used by `crypto_box` to seal/opens frames.
- **sign keypair** `(sign_pk, sign_sk)` — Ed25519, used to authenticate the handshake and
  to sign every frame so the relay (and the receiver) can reject forgeries.

The relay's trust store is **only the signing public keys**:
`SBP_TRUSTED_KEYS = keyId:base64(sign_pk)[:base64(box_pk)]`. It never receives any secret.

## 3. Keys & provisioning

- Identity keypairs are generated once per endpoint and stored locally (device: on the
  ECS host; client: in the browser `localStorage` / mini-program storage). **Private keys
  never leave the endpoint.**
- The **routing key `rk`** is a 32-byte secret shared between the trusted endpoints
  (device + client). The relay never sees it. Distribute it the same way you would a
  password — e.g. as a hex string in the client config and the device env
  (`SBP_E2EE_RK=...`), or derive it from a passphrase both sides know via Argon2id
  (`derive_rk(passphrase)` → 32 bytes).
- The peer's **box_pk** (needed to *seal* to them) and **sign_pk** (needed to *verify*
  their frames) are pinned in each endpoint's config (`SBP_E2EE_PEERS=keyId:boxB64:signB64`).
  Pinning removes the relay from the key-distribution trust path: a relay that swaps a
  public key cannot impersonate a peer.

## 4. Handshake (auth, not key exchange)

The relay issues nothing; the client/device proves possession of `sign_sk` over a
server-supplied challenge.

```
client/device                                 relay
  hello {
    role, keyId, ts, nonce,
    sig = Ed25519(sign_sk, "sbp1|{keyId}|{ts}|{nonce}")
  } -----------------------------------------▶
                                                verify sig with sign_pk[keyId]
                                                reject (4401) if bad / ts off by >5min
  ◀-------------------------- hello.ok { kid, scope, caps }
```

`ts` is milliseconds; window ±5 min (replay protection). The relay records `kid` on the
connection; every later frame must carry the same `kid`.

No secret key crosses the wire — only the signature. The relay can *verify* but not
*derive* the key.

## 5. Envelope (what the relay sees)

After auth, every application frame is wrapped in an **envelope**. The relay only ever
sees these fields; the inner `payload` stays encrypted.

```json
{
  "v": "sbp/0.3e",
  "tag": "<base64url( HMAC-SHA256(rk, scope) )>",
  "kid": "<sender keyId — cleartext, for relay sig-verify>",
  "eph": "<base64url( X25519 ephemeral public key )>",
  "sig": "<base64url( Ed25519 over (tag || eph || cipher) )>",
  "cipher": "<base64url( nonce[24] || box_ct )>"
}
```

where `box_ct = crypto_box(inner_json, nonce, recipient_box_pk, eph_sk)`.

- **Confidentiality** comes from `crypto_box` sealing to the **pinned recipient box_pk**.
  Only the true recipient can open it — the relay lacks `box_sk`, and even a wrong
  subscriber who receives the envelope via tag routing cannot decrypt it.
- **Integrity / identity** comes from `sig`, verifiable by the relay (with `sign_pk[kid]`)
  and by the receiver (with its pinned peer `sign_pk`). A frame whose `sig` fails is
  dropped.
- **Forward secrecy**: a fresh `eph` keypair per frame means each frame's box key is
  independent; leaking a long-term key later does not decrypt past traffic.

### Inner frame (encrypted)

```json
{ "type": "req|res|evt", "name": "...", "id": "...",
  "sid": "...", "payload": { ... } }
```

This is exactly the existing SBP-0.1 frame minus routing — so device and client logic
reuse their current serialization; only the transport adds/removes the envelope.

## 6. Routing by opaque tag (blind pub/sub)

The relay routes **only by `tag`**; it never parses `sid`/`did`.

```
scope                tag input                   subscribed by
-----------------    ------------------------    -------------------------------
device tag           "dev|{did}"                 device + client (for session.list)
session tag          "sid|{did}|{sid}"           device + the client viewing it
```

- On connect, the device subscribes to its device tag and to every session tag it knows.
- The client subscribes to the device tag (to fetch the list) and, when it opens a
  session, to that session tag.
- Subscription is a cleartext-but-signed `ctl` frame:
  `{"v":"sbp/0.3e","ctl":"sub|unsub","tag":"...","kid":"...","sig":"..."}`.
  The relay verifies `sig` and updates `tag_subs[tag]`.

The relay delivers each envelope to **all connections subscribed to its `tag`** except
the sender. Because `tag` is `HMAC(rk, scope)` and `rk` is secret from the relay, the
relay cannot map a tag back to a `sid`/`did`.

## 7. Frame examples (E2EE)

Client → device, answer a prompt (tagged with the session tag):

```
envelope {
  tag = HMAC(rk, "sid|dev_abc|1cf0..."),
  kid = "cli",
  eph = <fresh>,
  cipher = crypto_box({type:"req",name:"session.answer",sid:"1cf0...",
                       payload:{pid:"a1",choice:2}}, nonce, dev_box_pk, eph_sk),
  sig = Ed25519(cli_sign_sk, tag||eph||cipher)
}
```

Device → client, a streamed message (same session tag):

```
envelope { tag = HMAC(rk,"sid|dev_abc|1cf0..."), kid="dev",
           cipher = crypto_box({type:"evt",name:"session.message",...}),
           sig = Ed25519(dev_sign_sk, ...) }
```

The relay forwards both by `tag` without ever seeing the prompt text or history.

## 8. Degradation paths

| Mode | When | Behaviour |
|---|---|---|
| **E2EE** (`SBP_E2EE=1`) | production, untrusted relay | full envelope crypto as above |
| **Plaintext / HMAC** | trusted LAN, legacy clients | existing SBP-0.1 HMAC + JSON frames on `/ws` (unchanged) |
| **Refuse** | E2EE configured but peer key missing | endpoint errors loudly instead of sending plaintext |

The E2EE endpoint (`/ws-e2ee`) and the legacy endpoint (`/ws`) coexist on the same
relay. Clients opt in by connecting to `/ws-e2ee` with a configured `rk` + keys. If a
peer's `box_pk`/`sign_pk` is not pinned, the endpoint MUST refuse to send (no silent
fallback to plaintext over an untrusted relay).

## 9. Residual risks (accepted)

- **Metadata**: the relay sees timing, sizes, and online status. Mitigate with
  cover-traffic later if needed.
- **Availability**: the relay can drop/delay frames. Endpoints detect missing `seq` and
  alert; the relay cannot forge content (no `sk`).
- **Key compromise**: if a device is fully compromised, the attacker reads that machine's
  local tmux (game over regardless of transport). E2EE protects the *transit* path, not
  the endpoints.

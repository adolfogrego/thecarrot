# thecarrot

**A receiver of ARIA credentials.** Live at [thecarrot.rest](https://thecarrot.rest).

thecarrot.rest is a small, playful site that checks an [ARIA](https://aria.bar) credential and decides what to allow. Bunny is stuck at the top of a path with two locked gates. A valid credential opens the first one. The right scope (permission to eat) opens the second one, and Bunny can reach the carrot.

Credentials are generated at **[thebunny.bar](https://thebunny.bar)**. The two sites are deliberately independent: different domains, different hosts, no shared accounts, no shared session, no shared backend, and no call to the ARIA registry when a credential is verified. The only thing thecarrot.rest knows about thebunny.bar is where to find its public keys.

## Try it

1. Open [thebunny.bar](https://thebunny.bar) and sign in as Bunny.
2. Pick what Bunny should carry: no credential, a credential without eat permission, a credential with eat permission, or a delegated sub-Bunny.
3. Follow the link to thecarrot.rest and try to drag Bunny along the path.
4. Tap **what's happening?** at the bottom for a plain-language explanation of the current state, the raw credential, and the technical verdict.

| What Bunny presents | Gate 1 (valid credential) | Gate 2 (Scope:eat) |
|---|---|---|
| Nothing | closed | closed |
| A credential that can't be read, has a bad signature, or whose issuer key can't be fetched | closed | closed |
| A credential that has expired (thebunny.bar issues them with a 3-minute lifetime) | closed | closed |
| A valid credential **without** `carrot:action:eat` | open | closed |
| A valid credential **with** `carrot:action:eat` | open | open |

With a closed gate Bunny can still be pushed against it, but only a little, and springs back when released.

When Bunny presents a credential it carries a small ID card (🪪) beside it: in color if the credential was accepted, greyed out if it was rejected or has expired.

## How it works

1. thebunny.bar signs an ARIA credential for Bunny in the browser. The credential is a W3C Verifiable Credential carrying a composite post-quantum signature (ML-DSA-65 + Ed25519). It is passed along in the link as `?credential=…`.
2. thecarrot.rest decodes it and fetches the issuer's public keys, fresh on every visit, from `https://thebunny.bar/.well-known/aria-identity.json`:
   ```json
   { "v": "aria1", "pq": "<ML-DSA-65 public key, hex>", "ed": "<Ed25519 public key, hex>", "status": "active" }
   ```
3. It verifies the credential with the official ARIA verification SDK, pointed at those keys:
   ```js
   import { verifyAgent, setRegistryKeys } from 'https://esm.sh/@aria-registry/verify@1.1.0';

   setRegistryKeys(hexToBytes(identity.pq), hexToBytes(identity.ed));
   const result = verifyAgent(credential);  // { valid, did, trustLevel, scopes, … }
   const canEat = result.valid && result.scopes.includes('carrot:action:eat');
   ```
4. Gate 1 opens if the credential is valid. Gate 2 opens only if the credential also declares the scope this receiver requires.

Verification itself is local: signatures, structure and expiry are checked in the browser. Altering a credential after it was signed, for example adding a scope, makes the signature fail.

The signature is verified once, when the page loads. After that, whenever Bunny walks up to a gate, the page only compares the credential's expiry date with the clock. If it has passed, the gates shut again and Bunny goes back to the start. This mirrors how real systems treat expiry: a cheap check on each request, not a timer per credential.

## Relationship to ARIA

[ARIA](https://aria.bar) is an open protocol for verifiable identity and authorization of AI agents, stewarded by [TrustLayer Foundation](https://trustlayer.foundation). This repo is a minimal receiver of its credential format; the protocol and reference material live in [trustlayer-foundation/aria-protocol](https://github.com/trustlayer-foundation/aria-protocol).

- **Issuer.** The SDK normally verifies against the ARIA registry's keys. It also lets a verifier point at a different signing authority with `setRegistryKeys()`, which is what this site does with thebunny.bar's keys. The credentials issued by thebunny.bar are self-issued demo credentials. They are **not** issued by the TrustLayer registry.
- **Trust level.** Everything here is L0, the lowest ARIA level: it shows that a key signed the credential, not who is behind it. Higher levels, which add verified identity, are outside this demo.
- **Scopes.** `carrot:action:eat` follows ARIA's scope grammar (three colon-separated segments, wildcard only at the end). ARIA does not define vocabularies, so `carrot`, `action` and `eat` are this demo's own words, the same way a real receiver would define its own.
- **Policy.** The policy shown in the panel uses the format of ARIA's Agent Trust Policy (`_aria-policy` DNS record) and lists what this receiver requires. It is displayed for reference: this version does not read it from DNS, the same rules are written into the page.

## What's inside

A single `index.html`. No build step, no backend, nothing to install.

- `@aria-registry/verify@1.1.0`, loaded in the browser from [esm.sh](https://esm.sh), is the only dependency.
- Plain HTML, CSS and JavaScript for the rest: an SVG path, pointer events for dragging, and the explanation panel.
- Hosted on GitHub Pages; the `CNAME` file maps it to thecarrot.rest.

To run it locally, serve the folder with any static server (`python3 -m http.server`) and open it. It still fetches thebunny.bar's keys, so credentials generated there will verify.

## Known simplifications

This is a demo, and some things are intentionally simple:

- The credential travels in the URL. It is about 8 KB because of the post-quantum signature, and URLs end up in browser history and server logs. A real deployment would send it in a request header and require proof that the sender holds the key.
- Revocation is not checked.
- The receiver trusts one hard-coded issuer, thebunny.bar. A real receiver decides whom to trust through policy.
- thebunny.bar's demo keys are public on purpose. It is a toy identity; don't reuse that pattern for anything real.

## Links

- Credential generator: [thebunny.bar](https://thebunny.bar)
- Protocol: [aria.bar](https://aria.bar) · [aria-protocol](https://github.com/trustlayer-foundation/aria-protocol)
- Verification SDK: [@aria-registry/verify](https://www.npmjs.com/package/@aria-registry/verify)
- Steward: [TrustLayer Foundation](https://trustlayer.foundation)

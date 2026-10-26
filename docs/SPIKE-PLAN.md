# Spike plan: a JavaCard that holds Fedimint spend keys

Written 2026-10-26. Fedimint source references are to `fedimint/fedimint` at `e486b46`
(main, 2026-10-26).

## The question

Can a JavaCard hold the spend key of a Fedimint note and sign for it, so that a real
federation accepts the spend while the key never leaves the card?

If yes, an NFC card for Fedimint needs no consensus change: every existing federation
already supports it, and the work is all on the card and the client.

## Why we think the answer is yes

- A Fedimint note is `(nonce, signature)`. The nonce is a secp256k1 public key, and the
  signature is a threshold BLS12-381 signature over a hash of it
  (`modules/fedimint-mintv2-common/src/lib.rs`, `Note` and `verify_note`).
- Spending a note returns `InputMeta { pub_key: input.note.nonce }`
  (`modules/fedimint-mintv2-server/src/lib.rs`), and the transaction must carry one
  BIP340 Schnorr signature per input over the transaction ID, checked against that key
  (`fedimint-core/src/transaction.rs`, `validate_signatures`, `NaiveMultisig`).
- Blinding is multiplicative on G1 (`crypto/tbs`, `blind_message`, `unblind_signature`).
  It hides notes from the federation; it does not decide who can spend. So the phone can
  do all of the BLS work while the card keeps the spend keys, and the phone still cannot
  spend the card's notes.
- The current mint module (`modules/fedimint-mint-*`) uses the same model
  (`pub_key: *input.note.spend_key()`), so it is a fallback if `mintv2` is hard to drive.
- The Cashu Card applet already makes BIP340 signatures with JavaCard hardware
  (`SchnorrHW.java` in cashucard/javacard), and has run on J3R180 silicon.

## Out of scope for the spike

BLS on the card, the change pool, parsing transactions on the card, PIN and owner rules,
and any Fedi integration. These go in the design note that follows the spike.

## Decisions

| Decision | Choice |
|---|---|
| Repo | `bobodread876/fedimint-card`, public, MIT |
| Commits | GPG-signed as Dread (`github@bobodread.com`, key `3139A2E774DAF7328533D0EC0A897098CF0A7E5B`) |
| Mint module | `mintv2`; fall back to the current mint module if `mintv2` costs more than half a day |
| Real card | Yes, step 7 is in: a spare NXP J3R180 and a PC/SC reader |
| Reuse | Copy `SchnorrHW.java` from cashucard/javacard with a credit line; everything else from scratch |

## Layout

```
applet/      JavaCard applet + jCardSim tests (Maven)
bridge/      Java HTTP bridge that serves the simulated applet on 127.0.0.1
host/        Rust workspace: card transport, federation tests
docs/        this plan, then SPIKE.md (results)
```

## Steps

| # | Step | Done when |
|---|---|---|
| 0 | Repo bootstrap: README, licence, `.gitignore`, CI running the Java and Rust tests | First signed commit, CI green |
| 1 | Applet with three commands: `GEN_KEY(slot)` makes a secp256k1 key on the card, `GET_PUB(slot)` returns the x-only public key, `SIGN(slot, 32 bytes)` returns a BIP340 signature. A few key slots, no PIN | jCardSim tests pass, BIP340 vectors pass, no command returns a private key |
| 2 | Card transport. The bridge wraps the applet in jCardSim behind `POST /apdu`; the Rust side has one `Card` trait with a bridge backend and a PC/SC backend (`pcsc` crate) | A Rust test generates a key on the card, gets a signature, and `secp256k1` verifies it |
| 3 | Test federation from `fedimint-testing` fixtures (`Fixtures::new_primary`, `new_fed_not_degraded`; see `modules/fedimint-mintv2-tests/tests/tests.rs`): 4 guardians, fake bitcoin, no devimint or nix | The federation starts inside `cargo test` and a test client is issued ecash |
| 4 | Issue a note to the card. The host takes the card's public key as the nonce, blinds `nonce_message(nonce)`, pays for the output from the test client, fetches shares from the `signature_shares` endpoint, combines and unblinds | `verify_note` passes and the note's nonce is the card's key |
| 5 | Spend with the card. The host builds a reissue transaction (card note in, host notes out), the card signs `tx_hash()`, the host submits it | The federation accepts it, and the new host notes are spendable |
| 6 | Negative checks | A signature from another key is rejected; a second spend of the note is rejected; changing an output after signing is rejected |
| 7 | Real card. Build the CAP, install it with GlobalPlatformPro, rerun steps 5 and 6 over PC/SC | Steps 5 and 6 pass on the J3R180; signing time measured |
| 8 | Write `docs/SPIKE.md` | Records what worked, timings, the size and encoding of a typical transaction (input for on-card parsing), and the questions for the design note |

## Risks

- **The client assumes it holds secret keys.** `mintv2` client inputs carry a `Keypair`
  (`modules/fedimint-mintv2-client/src/lib.rs`). Steps 4 and 5 will likely bypass the
  client's transaction builder: build the `Transaction` by hand, attach the card's
  signature as `TransactionSignature::NaiveMultisig`, and submit through the federation
  API. Time-boxed to half a day before the fallback.
- **Version.** Main is `0.14.0-alpha`; the newest tag seen was `v0.13.0-beta.1`. Pin the
  newest release that contains `mintv2`, else pin a main commit. Record the pin in
  `host/Cargo.toml` and in `SPIKE.md`.
- **Build time.** The first fedimint build is heavy (RocksDB and friends).
- **jCardSim is not silicon.** Its EC comes from BouncyCastle, so step 1 passing does not
  prove `ALG_EC_SVDP_DH_PLAIN_XY` behaves the same on the J3R180; step 7 settles it.
  `SchnorrHW.java` notes that the DH output framing differs across cards.

## Machine setup

- Rust (stable; follow fedimint's `rust-toolchain.toml` if it pins one)
- JDK 17 and Maven
- jCardSim `3.0.6.0` (the `ph4r05` fork on Maven Central, as cashucard/javacard uses)
- JavaCard SDK and `ant-javacard` for the CAP; GlobalPlatformPro (`gp.jar`) to install it
  (see cashucard/javacard `docs/HARDWARE_DEPLOYMENT.md`)
- A PC/SC reader; macOS ships the PC/SC framework
- The GPG key above, for signed commits

## After the spike

Draft `docs/DESIGN-NOTE.md`: the change pool (blinded change outputs prepared by the
owner's phone), the card checking the transaction for its own change before signing,
what a till must run, and which rules, if any, need a federation module. Share it with
Fedimint maintainers (GitHub Discussions) and Fedi, with the spike as the demo.

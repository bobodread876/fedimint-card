# fedimint-card

An experiment in NFC cards for [Fedimint](https://github.com/fedimint/fedimint) ecash.

The card holds the spend keys of Fedimint notes and signs spends with them; the phone or
till does the rest. Because Fedimint notes are spent with a Schnorr signature from each
note's own key, this needs no change to the Fedimint protocol and works with existing
federations.

Status: planning. The first spike is described in [docs/SPIKE-PLAN.md](docs/SPIKE-PLAN.md).

This is experimental software. Do not put real funds on a card.

## License

MIT

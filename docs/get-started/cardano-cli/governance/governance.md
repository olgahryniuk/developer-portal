---
title: Cardano governance
sidebar_position: 8
keywords: [governance, update proposals, cardano, cardano-node]
---
# Cardano Governance
## Update proposals

Currently, and until Conway era comes bringing decentralized governance, Cardano operates under a **federated governance** mechanism that allows updating protocol parameters, including the addition of new features. These updates are carried out through an update proposal process. Only the holders of the genesis delegate keys have the authority to submit and vote on proposals.

## Cardano ledger eras

Cardano has transitioned through different ledger eras: Byron, Shelley, Allegra, Mary, Alonzo, and the current era, Babbage. Each of these eras has brought forth a new set of functionalities. The next planned era is Conway, bringing decentralized governance implementing [CIP-1694](https://cips.cardano.org/cip/CIP-1694)

| Era     | Key features | Consensus | Protocol versions (major/minor) | Hard fork event   | Date |
| ------- | -------------| ----------| :-----------------------------: | ----------------- | --- |
| Byron   | Proof of stake | Ouroboros Classic<br/>PBFT | 0.0<br/>1.0  | Genesis Byron reboot |  | 
| Shelley | Decentralized block production | TPraos |  2.0 | Shelley HF | | 
| Allegra | Token locking | TPraos| 3.0 | Allegra HF |  |
| Mary    | Native tokens | TPraos| 4.0 | Mary HF    |  |
| Alonzo  | Plutus smart contracts| TPraos| 5.0<br/>6.0 | Alonzo HF<br/>Alonzo intra-era |  | 
| Babbage | PlutusV2<br/>Reference inputs<br/>Inline datums<br/>Reference scripts<br/>Removal of the "d" parameter | Praos | 7.0<br/>8.0 | Vasil<br/>Valentine intra-era (secp) |  |
| Conway  | Decentralized governance | Praos | 9.0<br/>10.0 | Chang HF Bootstrap<br/>Full governance | Planned |

Transitioning from one era to the next is triggered by an **update proposal** that updates the protocol version.

### Byron era update proposals
![Byron update proposals](/img/cli/upbyron.png)

The general mechanism for updating protocol parameters in Byron is as follows:

1. **Update proposal is registered.** An update proposal starts with a transaction that proposes new values for some protocol parameters or a new protocol version. This kind of transaction can only be initiated by a genesis key via its delegate.
2. **Accumulating votes.** Genesis key delegates **vote** for or against the proposal. The proposal must accumulate a sufficient number of votes before it can be confirmed. The threshold is determined by the **minThd** field of the [softforkRule protocol parameter](https://github.com/input-output-hk/cardano-ledger/blob/2a0abd500b9e01efe6dc47146fa8b805ef9ef307/eras/byron/ledger/impl/src/Cardano/Chain/Update/SoftforkRule.hs#L24).
3. **Confirmed (enough votes).** The system records the 'SlotNo' of the slot in which the required threshold of votes was met. At this point, 2k slots (two times the security parameter k) need to pass before the update is stably confirmed and can be _endorsed_. Endorsements for proposals that are not yet stably confirmed are not invalid but rather silently ignored.
4. **Stably-confirmed.** The last required vote is 2k slots deep. Ready to accumulate endorsements. A block whose header's protocol version number is that of the proposal is interpreted as an **endorsement**. In other words, the nodes are ready for the upgrade. Once the number of endorsers satisfies a threshold (same as for voting), the confirmed proposal becomes a **candidate proposal**.
5. **Candidate.** Enough nodes have endorsed the proposal. At this point, a further 2k slots need to pass before the update becomes a stable candidate and can be adopted.
6. **Stable candidate.** The last required endorsement is 2k slots deep.

If there is no stable candidate proposal, then no changes occur. Everything is retained, including a candidate proposal whose threshold-satisfying endorsement was not yet stable and will be adopted in the subsequent epoch unless it gets surpassed in the meantime.

A 'null' update proposal is one that neither increases the protocol version nor the software version. Such update proposals are considered invalid according to the Byron specification.

### Update proposals in Shelley and subsequent eras

The update mechanism in Shelley is simpler than in Byron. There is no distinction between votes and proposals. To 'vote' for a proposal, a genesis delegate submits the exact same proposal via a regular transaction signed by the delegate key. Additionally, there is no separate endorsement step:

![Shelley update proposals](/img/cli/upshelley.png)

The procedure is as follows:

1. **Register the proposal.** During each epoch, a genesis key can submit (via its delegates) zero, one, or many proposals; each submission overrides the previous one. Proposals can be explicitly marked for future epochs; in that case, they are simply not considered until that epoch is reached.
2. **Voting.** The window for submitting proposals ends 6k/f slots before the end of the epoch, where *k* is the security parameter and *f* is the _active slot coefficient_.
3. **Quorum.** At the end of the epoch, if the majority of nodes, determined by the **Quorum** specification constant (which must exceed half the total number of nodes), have most recently submitted the identical proposal, it becomes adopted.
4. **Update.** The update is applied at the epoch boundary.

Changing the values of the protocol parameters and global constants can be done via an update proposal.

Changing the values of the global constants always requires a software update, including an increase in the protocol version. An increase in the major version indicates a hard fork, while the minor version indicates a soft fork (meaning old software can validate but not produce new blocks).

On the other hand, updating protocol parameters can be done without a software update.

Until the Voltaire era is released, only the holders of the **genesis delegate keys** can submit and vote on proposals. From time to time, you will find it useful to deploy a local or private testnet. In that case, you will need to use the governance commands to upgrade your network to the desired era.

Example of a protocol parameter update proposal updating nOpt (the desired number of pools)

:::note
Only the holders of the **genesis delegate keys** can create this type of update proposals. 
This applies to Mainnet and Testnets running in any pre-Conway era.  
If you are running a private testnet, you can use this feature to update your testnet parameters.
:::

```bash
cardano-cli babbage governance action create-protocol-parameters-update \
--genesis-verification-key-file genesis-keys/non.extended.shelley.delegate.vkey \
--number-of-pools 1000 \
--out-file updateNOpt.proposal
```

Build, sign and submit the transaction: 

```bash
balance=$(cardano-cli query utxo --address $(cat payment.addr) --out-file /dev/stdout | jq '. | .[keys[0]].value.lovelace')
fee=1000000
change=$(($balance - $fee))

cardano-cli babbage transaction build-raw \
--fee $fee \
--tx-in $(cardano-cli query utxo --address $(cat payment.addr) --out-file /dev/stdout | jq -r 'keys[0]' \
--tx-out $(cat payment.addr)+$change \
--update-proposal-file updateNOpt.proposal \
--out-file updateNOpt.tx.raw

cardano-cli conway transaction sign \
--tx-body-file updateNOpt.tx.raw \
--signing-key-file payment.skey \
--signing-key-file genesis-keys/non.extended.shelley.delegate.skey \ 
--out-file updateNOpt.tx.signed

cardano-cli conway transaction submit --tx-file updateNOpt.tx.signed
```

## Update proposals in Conway era

Todo:
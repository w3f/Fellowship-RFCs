# RFC-xxxx: Extend BEEFY Signature to include Nugget BLS signature

|                 |                                                                                     |
|-----------------|-------------------------------------------------------------------------------------|
| **Start Date**  | 04.06.2024                                                                          |
| **Description** | BEEFY should support Nugget BLS signature making more efficient zk bridges possible |
| **Authors**     | Syed Hosseini                                                                      |

## Summary

Polkadot Validators MUST sign the BEEFY payload using Nugget BLS signature as described in [1] additionally to ECDSA signature that they are currently using.

## Motivation

The current bridging paradigm offered by BEEFY, verifying randomly sampled set of ECDSA signature is both complicated and inefficient compared to verifying aggregated BLS signatures of all validator. This will allow the bridged chain to be updated about the Polkadot state almost instantly after it gets finalized on Polkadot. In comparison, the bridged chain can only securely be informed of state of Polkadot after a  significant delay due to the security assumption of random sampling algorithm.

The additional Schnorr signature mandated by Nugget BLS protocol will help validators to more efficiently (of factor 4x) verify individual Nugget BLS signatures from  other validators before gossiping them making the possibility of DoS attack against BEEFY and the relay protocol less likely.

Mandating Polkadot Validators to keep signing BEEFY payload with ECDSA algorithm, ensure that current bridge infrastructure and applications depending on it function with minimal disturbance while transition to more efficient BLS based bridge is underway.

## Stakeholders

- Polkadot Node implementors.
- Polkadot validators.
- Polkadot Bridges.
- Applications on the bridged chains using Polkadot ecosystem.

## Explanation

This RFC mandates that the BEEFY vote message to migrate from current format of:

```
ScaleEncode(BEEFY-Commitment, ECDSA-PublicKey, ECDSA-Signature)
```

to:

```
ScaleEncode(BEEFY-Commitment, ECDSA-PublicKey|BLS-PublicKey-in-G1|BLS-Publickey-in-G2, ECDSA-Signature|BLS-Signature-in-G1|NuggetSchnorrSignature)
```

Accordingly BEEFY Public Keys, migrating from:

 33 Bytes ECDSA Public Key

To:

 33 Bytes ECDSA Public Key | 48 Bytes BLS-12-381 Public key G1 | 96 Bytes BLS12-381 Public key in G2

When validator rotate session keys the must provide proof of possession as described in [Nugget BLS paper](https://eprint.iacr.org/2022/1611). A reference implementation of the proof of possession can be found [here](https://github.com/w3f/bls/blob/master/src/double_pop.rs#L58). 

Runtime must reject session key rotation transaction whose proof of possession does not verify.

Accordingly, Polkadot node implementation must provide runtime host function `ecdsa_bls381_generate`with the following signature:
```
fn ecdsa_bls381_generate(
&mut self, 
id: KeyTypeId,                                                                         seed: Option<Vec<u8>>,                                                                 ) -> ecdsa_bls381::Public;
```
Where the four byte key type ID must be set to:
```
ecb8
```

Validators must verify the validity of both ECDSA and the Nugget BLS signature before gossiping the message.

Benefit of the proposed change:

- **Backward compatibility**: Keeping the ECDSA signature will let bridging approaches depending on ECDSA signature stay functioning while making more efficient ZK based bridging schemes possible.


- **Enable ZK Bridging**
Using BLS signature, allows to provide succinct prove of aggregation of the public keys using zk-SNARKs based algorithms such as [web3sum](https://eprint.iacr.org/2022/1205) or [hinTS](https://eprint.iacr.org/2023/567). This makes it possible to verify BLS signature on the bridge chain in constant time and space plus a bit vector identifying the validators whose signatures are present in the aggregation. 

- **Fast verification of individual signatures** Using Nugget Schnorr Signature allow the validators and provers to verify the validity of individual signatures significantly faster than verifying vanilla BLS Signatures in G1. 

- **Faster verification of aggregated BLS Public Key**: Providing the BLS Public Keys in G1 as well as in G2, allow verifier of the BLS signatures to verify the Aggregation of Public Keys in G2 by performing the aggregation of Public Keys G1 which is defined over a smaller finite field and offer faster Elliptic Curve operation. 


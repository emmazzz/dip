---
dip: 14
title: Multi-Agent Transactions
authors: Emma Zhong(@emmazzz), Tim Zakian, Sam Blackshear
Status: Draft
type: Standard
created: 04/05/2021
---


# Summary

This DIP describes a new scheme of user transaction: multi-agent transactions.

# Abstract

Currently in the Diem Framework, a transaction acts on behalf of a single on-chain account. However, there is no mechanism for multiple on-chain accounts to agree on a single atomic transaction. This DIP presents a new scheme of transactions--multi-agent transactions--which act on behalf of multiple on-chain accounts. Multi-agent transactions leverage Move’s [_`signer`_](https://developers.diem.com/docs/move/move-signer/) type to allow essentially any arbitrary atomic actions in one transaction involving multiple on-chain accounts.

## Terminology:

* Primary signer: This is the account that the transaction is sent from. This account’s sequence number is incremented, and is the account that pays gas. There is precisely one of these for every transaction.
* Secondary signer: This is any account that participates in a multi-agent transaction that isn’t the primary sender. There can be 0-N of these. The 0 case is a normal transaction today. 

# Motivation/Use Cases

## Minting Directly to VASPs

Today in the Diem Framework, we need two transactions in order to mint money through designated dealers to VASPs. The first transaction is a tiered-mint transaction sent by the treasury compliance account to mint money to a designated dealer’s account. The second transaction transfers the minted funds from the designed dealer’s account into the VASP’s account. This procedure requires two transactions due to the restriction that each transaction can only have one signer argument. In the multi-agent scheme, this restriction no longer exists and we can perform both steps in a single atomic transaction (see code sample below). This atomicity implies that designated dealers no longer need custody services for the money they temporarily hold in their accounts, making it more affordable and much quicker to operate as a designated dealer. This should make it much easier for more DDs to onboard in the Diem ecosystem, providing more options for VASPs. It is highly likely that in the future, DDs primarily mint directly on requests from VASPs.

```
fun mint<Coin>(
   tc: signer,
   dd: signer,
   vasp_addr: address,
   amount: u64,
   tier_index: u64
) {
   // First, TC mints to DD.
   tiered_mint<Coin>(
       tc, address_of(dd), amount, tier_index
   );

   // Then, DD distributes funds to VASP.
   pay<Coin>(
       dd, vasp_addr, amount
   );
}
```



## Atomic Swaps

In order to do a currency exchange in the current scheme between two on-chain entities Alice and Bob, we need two transactions and possibly an escrow. The time gap between the two transactions can lead to potential problems such as resource lockup. With the new multi-agent transaction scheme, we only need one atomic transaction signed by both Alice and Bob, in which payments are sent in both directions (see the code sample below). In this case, both Alice and Bob have the same control over the transaction contents including exchange rates and expiration time.


```
// alice and bob agree on the values of amount_usd and amount_eur 
// off-chain.
fun exchange(
    alice: signer, bob: signer, 
    amount_usd: u64, amount_eur: u64
) {
    // First, Alice pays Bob in currency USD.
    pay<USD>(alice, address_of(bob), amount_usd);

    // Then, Bob pays Alice in currency EUR.
    // Previously, this line has to be a separate transaction.     
    pay<EUR>(bob, address_of(alice), amount_eur);
}
```



## Actual Dual-attested Transactions 

Currently, we execute a [_dual-attested payment_](https://dip.diem.com/dip-1/) transaction by including a signature of the payee as an argument to the transaction, and checking the validity of the signature on-chain if the payment is between two different VASPs and the amount is over a certain threshold. In order to accomplish that, we have to run some fairly complex code to reconstruct and verify the signature ad-hoc on-chain. With the new scheme, we can simply require that the payee signs the entire transaction as well, and verify the signatures of both the payer and payee while validating the transaction. The code samples below show the before and after of what such a transaction might look like. 

Some benefits brought by the multi-agent scheme in this scenario are:

* Extensibility: In the current scheme, our dual attestation protocol is very specific to travel rule payments. And extending this protocol to dual attestation on a different type of action would require another ad-hoc data format and signature scheme. The multi-agent transaction scheme generalizes this by allowing dual attestation on any action encodable by a Move script without having to define new data formats or signature schemes.
* Security: Multi-agent scheme closes two security loopholes of our current protocol:
    * Replayability: Since the payee only signs over the payer and the amount of the payment, today the payer can replay the transaction using old payee approvals. In the new scheme, this wouldn’t be possible since the payee has to sign over the sequence number of the transaction as well, disallowing any replay of the transaction.
    * Currency: Today’s protocol does not let the payee commit to the currency type so the payer can reuse a payee approval for payment in one currency for payment in another with the same amount. Again, since the payee has to sign over the entire transaction including the currency type, this scenario would be impossible with the multi-agent scheme. 
* Earlier failure: Currently a bad signature in a travel rule transaction will be caught at execution time, and the transaction will abort. In the new scheme, the signature is checked at validation time and the transaction will be discarded instead.

```
/// Before:
fun dual_attested_payment<Coin>(
    payer: signer, payee: address, amount: u64,
    metadata: vector<u8>, 
    metadata_signature: vector<u8>
){
    let msg = message(payer, amount, metadata);
    verify_signature(compliance_key(payee), msg);
    pay<Coin>(payer, payee, amount, metadata);
}

/// After:
fun dual_attested_payment<Coin>(
    payer: signer, payee: signer, amount: u64,
    metadata: vector<u8>
){
    pay<Coin>(payer, address_of(payee), amount, metadata);
}
```



## Arbitrary Atomic Actions in One Transaction

Besides the use cases mentioned above, our new multi-agent transaction scheme allows the sender to encode any arbitrary combination of actions in a single atomic transaction. This added expressiveness opens up a lot more possibilities in the type of scenarios that we can implement. Some examples include [_delivery versus payment_](https://www.investopedia.com/terms/d/dvp.asp) (a crucial feature for securities settlement) and atomic administrative approval of sensitive actions (e.g., account creation). 




# Code Changes to Support Multi-agent

## DiemVM Code Changes

### Add New Enum `RawTransactionWithData`

In order to preserve backward compatibility, we can’t add more fields representing secondary signers to `RawTransaction` struct. Thus we add the following additional enum `RawTransactionWithData`, which the sender and secondary signers will sign over to authenticate the transaction.


```
pub enum RawTransactionWithData {
    MultiAgent {
        raw_txn: RawTransaction,
        secondary_signer_addresses: Vec<AccountAddress>,
    },
}
```



### Add New Enum `AccountAuthenticator`

Previously each transaction can only have one signer, which is the sender of the transaction. In the multi-agent scheme, with the ability to have multiple signers, the transaction can potentially have multiple authenticators from different accounts and of different schemes. An `AccountAuthenticator` serves as the authenticator for one account. And a `TransactionAuthenticator` can contain multiple `AccountAuthenticator`s, as shown in the next subsection.


```
pub enum AccountAuthenticator {
    /// Single signature
    Ed25519 {
        public_key: Ed25519PublicKey,
        signature: Ed25519Signature,
    },
    /// K-of-N multisignature
    MultiEd25519 {
        public_key: MultiEd25519PublicKey,
        signature: MultiEd25519Signature,
    },
}
```



### Add New Variants to `TransactionAuthenticator`

We have added two new variants to `TransactionAuthenticator` to differentiate single-agent and multi-agent transaction scheme. In the single-agent scheme, sender is the only signer and sender’s signature is verified against the `RawTransaction`. While in the multi-agent scheme, the sender and all the secondary signers have to sign over `RawTransactionWithData::MultiAgent{ raw_txn, secondary_signer_addresses }`, to make sure that all parties agree to transact with each other. 


```
pub enum TransactionAuthenticator {
    /// Single signature
    /// This variant is kept for backwards compatibility.
    Ed25519 {
        public_key: Ed25519PublicKey,
        signature: Ed25519Signature,
    },
    /// K-of-N multisignature
    /// This variant is kept for backwards compatibility.
    MultiEd25519 {
        public_key: MultiEd25519PublicKey,
        signature: MultiEd25519Signature,
    },
    /// Single-agent transaction.
    SingleAgent { sender: AccountAuthenticator },
    /// Multi-agent transaction.
    MultiAgent {
        sender: AccountAuthenticator,
        secondary_signer_addresses: Vec<AccountAddress>,
        secondary_signers: Vec<AccountAuthenticator>,
    },
    ..
}
```



### Additional Checks During Transaction Validation

In addition to signature checking, we also perform the following additional checks on signed multi-agent transactions during transaction validation:

* The number of secondary authenticators in the signed transaction is the same as the number of secondary signer addresses that all parties have signed over.
* There are no duplicates in the signers. In other words, sender and all the secondary signers have distinct account addresses.




## Diem Framework Changes

### Account Existence Checking in Prologue

Today we check in the prologue that the sender of the transaction has a `DiemAccount` resource under their address. With the ability to have multiple signers, we need to check that all of them have `DiemAccount` resources during the prologue.

```
let i = 0;
while (i < num_secondary_signers) {
    let secondary_address = *Vector::borrow(&secondary_signer_addresses, i);
    assert(
        exists_at(secondary_address), 
        Errors::invalid_argument(PROLOGUE_EACCOUNT_DNE)
    );
    i = i + 1;
};
```



### Authentication Key Checking in Prologue

Currently in the prologue, we check that the hash of the sender’s public key is equal to the authentication key that the sender has on chain. In the new scheme, we will need to perform the same check for public keys of all the secondary signers as well. Therefore, two new arguments are added to the `script_prologue` function. They are `secondary_signer_addresses: vector<address> `and `secondary_signer_public_key_hashes: vector<vector<u8>>`. In the prologue, we add a loop going through the two vectors to perform the aforementioned check. 


```
let i = 0;
while (i < num_secondary_signers) {
    let secondary_address = *Vector::borrow(&secondary_signer_addresses, i);
    let signer_account = borrow_global<DiemAccount>(secondary_address);
    let signer_public_key_hash = *Vector::borrow(&secondary_signer_public_key_hashes, i);
    assert(
        signer_public_key_hash == *&signer_account.authentication_key,
        Errors::invalid_argument(PROLOGUE_EINVALID_ACCOUNT_AUTH_KEY),
    );
    i = i + 1;
};
```





# SessionWallet: Streamlining Web3 Transactions with Delegated Authority and Site-Specific Wallets

**By Winton Nathan-Roberts (Sydney, Australia)**

Tired of the constant confirmation pop-ups every time you play an online game, or in the case of Algorand opt into an asset? Concerned about the security risks to your digital wallet with each transaction? SessionWallet is set to transform your Web3 experience, eliminating these hassles altogether.\
Web3 gaming and platforms often suffer from transactional delays and security headaches. Up until now, users had to confirm every transaction or worry about their main wallet's safety. SessionWallet changes the game with a secure, user-friendly approach: delegate the authority to site-specific wallets designed for low-friction Web3 interactions.\
Imagine effortlessly executing multiple transactions during your gaming or DeFi sessions, secure in the knowledge that your main wallet is safe. Ready to learn how SessionWallet can make your Web3 interactions more secure, efficient, and enjoyable? Dive into our Medium article and discover how this innovative solution is reshaping the future of digital asset management.\\
P.s. I'm not trying to sell you anything, other than just to show you a cool idea and an MVP, which may be turned into a product. Please do your own research to confirm any claims made.

## Introduction

SessionWallet introduces a mechanism for delegating authority from an existing wallet to a website-specific, reproducible wallet for short-term client-side sessions. This method enables the use of an existing wallet to sign transactions on behalf of the user without requiring constant user approval for each transaction. It's particularly beneficial for applications like Web3 games, offering portability with widely-used wallets like MetaMask or Pera on Mobile. The wallet doesn't need to support signing raw bytes to enable delegation, instead any wallet can be supported as it has powerful on-chain parsing to validate the delegating transaction signature. It's implemented on Algorand, however it can be extended to any blockchain that supports contract accounts.


## MVP Demo Video

[![SessionWallet Video](https://img.youtube.com/vi/66N7bFn19Ck/0.jpg)](https://www.youtube.com/watch?v=66N7bFn19Ck)

## Method

The method involves first authenticating a user to create a proof of session (POS) as a signed transaction, with claims embedded into the transaction like the expiration of access through the POS, and the key that is approved to sign transactions that will be sent from the derived SessionWallet LSIG contract account which is reproducible and specific to each origin (i.e. website domain) and the listed owners (e.g. a user's main access key, and any recovery keys) (see Section 2). This POS is conveyed as contract arguments that match up against the bound template variables to gate approval of the transaction that is being sent from the associated contract account (see Section 1). Finally, the LSIG is used to sign transactions sent from the SessionWallet contract account, following the procedure explained in see Section 3.

### 1. Session Wallet Logic Signature (LSIG) Contract Account

We have the following SessionWallet pseudocode, which approves transactions according to its logic.

```python
SessionWallet(delegateTx, delegateTxSig, delgTxIdSig):

    # Identification
    owners = TMPL("owners") # Array of owner X25519 keys
    origin = TMPL("origin")
    assert len(credpkBound) <= 64
    assert len(originBound) <= 64

    # Extract and process delegated transaction fields
    credpk = getTXField("snd", delegateTx) # to field
    exp = getTXField("lv", delegateTx) # lastValid field
    delegatedCSPK = getTXField("rcv", delegateTx) # from field
    note = getTXField("note", delegateTx) # note field
    delegatedOrigin = getOriginFromNote(note) # origin in the note field

    # Approval Conditions
    assert delegatedOrigin == originBound
    assert credpk in credpkBound
    assert currentBlockTime < exp
    assert signatureValid(delegateTx, delegateTxSig, credpk)
    assert signatureValid(Txn.id, delgTxIdSig, delegatedCSPK)

    return True
```

### 2. Authentication

- Generate Ephemeral Key: ED25519 keypair (`cspk`, `cssk`)
- Construct Delegating Transaction (`delegateTx`):
  ```
    from: Owner Wallet (owner)
    to: Delgated ED25519 PK (cspk)
    note: ZKEntry: [website origin (origin)] (note)
    lastRound (lastValditity): expiration time in Unix seconds (exp)
    fee: 0
    amount: 0
    type: pay
  ```
- Sign Transaction with the Owner Wallet using any wallet protocol and software, like WalletConnect (`delegateTxSig`)
- `delegateTx` doesn't need to be submitted to the blockchain, it's just a proof of authority from the owner to delegate access to a the delegated ED25519 key (`cspk`)
- Reproduce the TEAL code SessionWallet with values for the `owners` (consisting of Owner Wallet and a recovery key if any), and `origin` (origin) template variables of the SessionWallet TEAL denoted `SessionWallet(owners, origin)`
  - Values for `owners` and `origin` may need to be stored in a database for future use, or be retrieved if there's multiple owners (i.e. a recovery key)

### 3. Signing

- Follow the Authentication procedure to yield `SessionWallet(owners, origin)`
- Prepare a transaction called `TX` from `Addr(SessionWallet(owners, origin))` (hash of program code), which is the address of the SessionWallet's associated contract account (`sessionAddr`). 
- Sign TX.Id with the ephemeral key (`cssk`) to get ED25519 signature denoted `delgTxIdSig`. TX.Id a unique hash of the transaction and all its fields.
- Create logic signature `SessionWallet(owners, origin)(delegateTx, delegateTxSig, delgTxIdSig)` denoted `sessionTxSig`, which fills out the arguments `delegateTx`, `delegateTxSig`, `delgTxIdSig` of the SessionWallet TEAL
- Sign the transaction `TX` with `sessionTxSig`
- Submit the signed `TX` to the blockchain, where the `TX` from the Session Wallet will be approved according to the above TEAL Signature (i.e. that one of the bound owners signed the delegateTx, the specified `cspk` in the `to` field signed the TX id, the templated origin matches that of the note, and the delegateTx itself hasn't expired by comparing against the lastRound field `exp`)

### 4. Creation

- If it's your first time using the Session Wallet, it should be essentially identical to the authentication flow except you may need to store the `owners` and `origin` values for future use to reproduce the SessionWallet TEAL. E.g. insert a database entry.

## Why it works

The fact the delegating transaction was signed by the owner, means the owner gives permission for the TEAL to approve transactions sent from it if it meets the conditions of the teal: hasn't expired, the note field contains the origin, and the transaction id is signed by the delegated ED25519 key (cspk), and the relevant fields are contained in the delegating transaction. Essentially it just means a user gives the website permission to sign transactions on their behalf, in a site specific wallet through a kind of client side only JWT (JSON Web Token) with signed claims describing the nature of the session as above.

## Why it's useful

- **Many websites, One Account**: We can use the same main wallet account to authenticate with multiple services, apps and websites where each app can only access assets relevant to that app, following Principle of Least Privilege to the level of a website.
- **Low Friction Signing** An app has the ability to sign transactions on behalf of the user, without the need for constant user approval for each transaction. This would be very useful for a *Web3 game*.
- **Portability** It's portable if you use a portable owner wallet, such as MetaMask, or Pera on Mobile.

## Security Analysis

The following breaks down relevant security features.

- Phishing attacks are mitigated, because of the inclusion of the note field where a user can clearly see the origin of the website they are authenticating with. If it looks different to the website they are expecting, they can cancel the transaction.
- The lifetime of a session is finite, and is site controlled. Meaning if a user doesn't use the website for a while, the session will expire and the website will need to request a new session. This is done through the expiration field in the delegating transaction (lastRound field).
- The owner wallet is entirely self-custodial and never exposed to the website. The website client only has access to the SessionWallet, which is a contract account that can only sign transactions according to the SessionWallet TEAL. In a sense, the SessionWallet is self-custodial because it isn't sent to a server and can be verified through client-side code integrity checks.
- Assuming the client-side is not malicious, which can be verified with open-source methods, the SessionWallet is as secure as a detached hardware or dedicated main wallet. This is provable.
- The wallet innately follows the principle of least privilege, as the Session Wallet is specific to each website, app or service. Meaning the worst case involves losing assets only for that site, say an NFT from a game, rather than all assets in the main wallet.

## Future Work

Many extensions and features are planned, particularly focusing on Web3 gaming, however they are withheld for now as they're developed internally.

## Contribution Statement

As far as I'm aware, the method and approach above is novel particularly its application in creating application specific contextual wallets through delegating transactions, where the signed claim is an entire transaction (not just a JWT) improving its compatibility. I'm grateful to the Algorand community for their invaluable support. 
All applicable rights reserved.

## Disclaimer

No warranties or liabilities to the fullest extent of the law. This work isn't a product yet, it still remains closed, not for public use and in R&D. The above method is not tested for security, and isn't ready for production use necessarily.

**Keywords (for SEO)**

SessionWallet, Authority delegation, Client-side, signed claims, Zero Knowledge Proof, User Session Specific Wallet, hyperlocalised and context specific wallets with a central access point, Algorand, Account Abstraction, Wallet Abstraction, Logic Signature, Contract Account, Web3 games, MetaMask, Pera Mobile, Ephemeral Key, Delegating Transaction, TEAL code, Signature validation, Blockchain, Least Privilege, Transaction approval

# Creating a new NFT

This guide was put together using Mac OS X. It's also a brain dump and mostly written for future me, so certain things are likely missing. But, perhaps it is still useful to others :)

## General Setup

### Set `CARDANO_NODE_SOCKET_PATH`

If running a node via Deadalus you can find the `--node-socket` param value via `ps ax | grep -v grep | grep cardano-wallet`

stdout will look something like this

```bash
42999   ??  S      9:16.46 cardano-wallet serve --shutdown-handler --port 50017 --database /Users/pc/Library/Application Support/Daedalus Flight/wallets --tls-ca-cert /Users/pc/Library/Application Support/Daedalus Flight/tls/server/ca.crt --tls-sv-cert /Users/pc/Library/Application Support/Daedalus Flight/tls/server/server.crt --tls-sv-key /Users/pc/Library/Application Support/Daedalus Flight/tls/server/server.key --token-metadata-server https://tokens.cardano.org --sync-tolerance 300s --mainnet --node-socket /Users/pc/Library/Application Support/Daedalus Flight/cardano-node.socket
```

Then set the environment variable:

```bash
# (beware that empty spaces will need to be escaped with a backslash on linux)
export CARDANO_NODE_SOCKET_PATH=/Users/pc/Library/Application\ Support/Daedalus\ Flight/cardano-node.socket
```

### Save local copy of the environment's **protocol.json**

```bash
cardano-cli query protocol-parameters \
--mainnet \
--out-file ./cardano/protocol.json

# for testnet
cardano-cli query protocol-parameters \
--testnet-magic 1097911063 \
--out-file ./cardano/testnet-protocol.json
```

---

## Create payment keys

The verfication and signing keys will be put in the root `keys/` directory. This directory is `gitignored` to reduce the chances of the keys being made public accidentally.

```bash
cd keys

cardano-cli address key-gen \
--verification-key-file payment.vkey \
--signing-key-file payment.skey
```

### Create a new address with the keys

**What's this address for?** We'll send a little bit of **ADA** here and use that UTxO to pay the minting fees. Additionally, we can send the new Asset Token to this address _(or it can be sent to your wallet!)_.

```bash
cd keys

cardano-cli address build \
--payment-verification-key-file payment.vkey \
--mainnet \
--out-file payment.addr
```

Now, use a wallet to send some funds to the new address. e.g. 10**ADA** will allow you to mint ~ 5 Native Assets when the minimum required UTxO tx is 1.4**ADA** for minting and a TX fee of about 0.5**ADA**. These values vary depending on the size of the metadata & the number of _different_ Asset Tokens minted in the transaction _(larger metadata & more Asset Token types = higher tx fee)_.

### Query to confirm the funds are received

This often takes at least 20 seconds.

```bash
cardano-cli query utxo --address $(cat payment.addr) --mainnet
```

## Minting Policies

The Minting Policy has a very special relationship with an Asset Token. From the Policy, you'll create a **PolicyID**, and the **PolicyID**+**AssetTokenName** makes up the unique token.

On Cardano, it's common to use the same **PolicyID** to mint multiple different Asset Tokens. **Why?**

1. _FIRSTLY_, it's more efficient _(less fees)_ to mint multiple tokens in the same transaction. It's also easier to mint multiple tokens in 1 transaction when they have the same **PolicyID**. Yes, you can mint multiple tokens in 1 transaction with different **PolicyID**s, but the `cli` command is slightly simpler when they share a **PolicyID** and policy signing keys, etc.

2. _SECONDLY_, a common **Policy ID** is a way of grouping a collection of related Asset Tokens together. All online Token browsers typically allow you to search by **PolicyID** which would return results for all Asset Tokens minted using that Policy. (TODO: insert example here)

**So what does a Minting Policy actually do though?** It describes the rules for minting and burning/destroying the Asset Token(s). Simples examples (that do not require code) are:

**A)** Allow minting/burning up until a certain point in time (determined by a slot number).

**B)** Allow minting/burning only _after_ a certain point in time (determined by a slot number).

**C)** Allow or require 1 or more signatures to mint/burn the Asset Token.

### Aside on NFTs...

If only **1** Asset Token has been minted, and based on the policy - no more can be minted (e.g. the policy has expired) then Congratulations! You've created an NFT!!

However, if there is still the ability to create additional Asset Tokens using the same policy - even if you do NOT actually create a 2nd, or 3rd - you arguably have **not** created an NFT!

**Why would I want to set the policy to expire in the future? Why not expire it immediately after minting the NFT?**

Because you might fuck it up! While a policy is active, you can burn (destroy) the token and re-create it with different metadata.

### Example "simple" policies

Simple policies are ones that do not require coding a contract.

```json
{
  // all of the scripts below must be met to use this policy
  //
  "type": "all",

  // any of the scripts below will enable using this policy
  //
  // "type": "any",

  // At least 2 of the scripts below must be satisfied
  // to use this policy
  //
  // "type": "atLeast",
  // "required": 2,


  "scripts":
  [

    // requires a particular signing key to use this policy.
    // Every policy should have at least 1 signature,
    // otherwise anyone can mint w/ the policy.
    {
      "type": "sig",
      "keyHash": "e09d36c79dec9bd1b3d9e152247701cd0bb860b5ebfd1de8abb6735a"
    },

    // policy is only usable  before the specified slot
    {
      "type": "before",
      "slot": <insert slot here>
    },

    // policy is only usable after the specified slot
    {
      "type": "after",
      "slot": <insert slot here>
    },
  ]
}
```

A common policy is a multi-signature policy, which is simply a list of `[{ "type": "sig", "keyHash": "abcdef" }, ...]` policies where policy `{ "type": "all" }`.

### Creating our Minting Policy

I'm going to go with a simple "before" policy that requires 1 signature.

create file **cardano/policies/datadog/policy.script**:

```json
{
  "type": "all",
  "scripts": [
    {
      "type": "before",
      // I think, 10 slots are created every 60s (4,320,000 every epoch (5 days))
      // `cardano-cli query tip` to get the most recent slot
      "slot": 46181890
    },
    {
      "type": "sig",
      "keyHash": "65c8bd1d93acf0f569c1f0d45c0f18df849295fbfff69c28cc6275a3"
    }
  ]
}
```

We will create a special signature, just for the minting policy. Alternatively you could use the verification signature from your wallet.

```bash
cardano-cli address key-gen \
    --verification-key-file cardano/policies/datadog/keys/policy.vkey \
    --signing-key-file cardano/policies/datadog/keys/policy.skey
```

```bash
// get the keyHash of verification key sig
cardano-cli address key-hash --payment-verification-key-file policy/policy.vkey
```

### getting the policy ID

The **policyID** will be baked into the Minting Transaction metadata. It

```bash
cardano-cli transaction policyid \
  --script-file ./cardano/policies/datadog/policy.script \
  >> ./cardano/policies/datadog/policyID
```

## Minting Transaction's Metadata

This is where the fun stuff lives. For NFTs there is no exact standard, but there is a common one that was created by the community and will allow the NFT to be easily displayed on NFT explorers/marketplaces. [See it here](https://github.com/cardano-foundation/CIPs/blob/master/CIP-0025/README.md).

```json
{
  "721": {
    "version": "1.0",
    "ea2f7bddb06f579c2127a64bff5920d1375457da7f43a5a6650abb43": {
      // DataDog1 must match the AssetToken name during the minting transaction
      // e.g. --tx-out <address-where-token-will-go>+140000+"1 <policyid>.DataDog1"
      "DataDog1": {
        "name": "DataDog #1",
        "description": "Testing with a simple image field to redirected asset",
        "mediaType": "image/png",
        "image": "https://github.com/joebartels/nft/raw/3ee476b/assets/data.png"
      }
    }
  }
}
```

## Minting Transaction

First we need to get the current minimum ADA amount. **Why?** Each UTxO has a minimum amount of ADA, so we need to satisfy that + the transaction fee. This should be about 1.5A / 1,500,000 lovelace.

### Estimate the minimum-required UTxO and fees

A couple things to note in the below TX:

- The token names must be base16 encoded (as of cardano-cli 1.31)
- There will be a minimum ADA UTxO fee. I'm guessing for it to be `140000` as shown in the command. The node will reject this and tell me what the current live minimum UTxo fee is. Once it does, re-run the command an replace `1400000` with that value.
- `<tx-hash>` is the hash to an UTxO
- `<tx-num>` is the tx # on the UTxO where the fees will be pulled from
- `<address>` is where the token(s) and minimum required UTxO will go
- `<address>+1400000` the 1400000 is an estimated minimum required UTxO. This will always be wrong, but the reason for including it, is b/c the stdout of this command will tell you the current live minimum required UTxO.
- RE: `<base16-encoded-TokenName>` the decoded TokenName should match Keys in the metadata.
- `--witness-override` TODO: look into purpose / reasonable defaults

```bash
cardano-cli transaction build \
--alonzo-era \
--testnet-magic 1097911063 \
--tx-in <tx-hash>#<tx-num> \
--tx-out <address>+1400000+"1 <policyID>.<base16-encoded-TokenName> + 1 <policyID>.<base16-encoded-TokenName2>" \
--change-address <address> \
--mint="1 <policyID>.<base16-encoded-TokenName> + 1 <policyID>.<base16-encoded-TokenName2>" \
--minting-script-file ./cardano/policies/datadog/policy.script \
--metadata-json-file ./cardano/policies/datadog/metadata/DataDog.json \
--invalid-hereafter 46181890 \
--witness-override 2 \
--out-file ./cardano/policies/datadog/tx/testnettx.raw
```

If command is successful, then the stdout will be the estimated UTxO fee (not to be confused with the minimum required UTxO fee). For example:

```bash
> Estimated transaction fee: Lovelace 1511549
```

### Signing the transaction

```bash
cardano-cli transaction sign \
--signing-key-file ./keys/testnet/payment.skey \
--signing-key-file ./cardano/policies/datadog/keys/policy.skey \
--testnet-magic 1097911063 \
--tx-body-file ./cardano/policies/datadog/tx/testnettx.raw \
--out-file ./cardano/policies/datadog/tx/testnettx.signed
```

### Submitting the signed transaction

```bash
cardano-cli transaction submit \
--mainnet \
--tx-file ./cardano/policies/datadog/tx/testnettx.signed
```

# Resources

- docs on simple policy scripts
  - https://web.archive.org/web/20210601112403/https://docs.cardano.org/projects/cardano-node/en/latest/reference/simple-scripts.html

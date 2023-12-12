# DogeRealCool Specification

## DRC-20 Concept

DRC-20 have three functions:

- Create a drc-20 with the deploy function
- Mint an amount of drc-20's with the mint function
- Transfer an amount of drc-20's with the transfer function.

drc-20 balance state can be found by aggregating all of these function's activity together.

- Deployments initialize the drc-20. Do not affect state
- Mints provide a balance to only the first owner of the mint function inscription
- Transfers deduct from the senders balance and add to the receivers balance, only upon the first transfer of the transfer function. (1. Inscribe transfer function to senders address 2. sender transfer's transfer function)
  
### Usage

#### Getting a balance
You can either deploy your own or 'first is first' mint from existing deployments
1. (Optional: Only do if you want to create your own drc-20. If not go to step 2) Inscribe the deploy function to you ordinal compatible wallet with the desired drc-20 parameters set.
2. Inscribe the mint function to your ordinal compatible wallet. Make sure the ticker matches a drc-20 that has yet to reach its fully diluted supply. Also, if the drc-20 has a mint limit, make sure not to surpass this. 

#### Transferring a balance
1. Inscribe the transfer function to your ordinal compatible wallet. Make sure the transfer function inscription information is valid before inscribing. 
2. Once inscribed, (and if valid) send the inscription to the desired destination to transfer the balance. 

##### What is valid?

A valid transfer function is required to transfer a balance. Validity can be determined by the following:

- Valid transfer functions are ones where the amount stated in the inscription does not exceed Available balance when inscribed.
- Available balance defined as: [Overall balance] - [valid/active transfer function inscriptions in wallet (Transferable balance)]. If there are no valid/active transfer functions held by an address Available balance and Overall balance are equivalent.
- Example: A wallet holds an Overall balance of 1000 "ordi", and . The holder then inscribes a transfer function of 700 "ordi". Once the inscription is confirmed, the following is true: Overall balance = 1000, Available balance = 300, Transferable balance = 700. If in the next block, the user tried to inscribe a transfer function of 500 "ordi", this would not be valid as the maximum amount that can be inscribed is 300 (Available balance). 

#### Balance Control

- If a user changes their mind and no longer wishes to transfer their transfer function, and wants to restore their Available balance to the Overall balance (invalidate Transferable balance), the user must simply transfer the transfer function inscription to themselves.

### Notes

- Do not send inscriptions to non ordinal compatible wallet taproot addresses
- Under no circumstances does the transfer of a mint function result in a balance change.
- Each transfer inscription can only be used once
- The first deployment of a ticker is the only one that has claim to the ticker. Tickers are not case sensitive (DOGE = doge). 
- If two events occur in the same block, prioritization is assigned via order they were confirmed in the block. (first to last).
- The mint function and the second step of the transfer function are the only events that cause changes in balances
- The first mint to exceed the maximum supply will receive the fraction that is valid. (ex. 21,000,000 maximum supply, 20,999,242 circulating supply, and 1000 mint inscription = 758 balance state applied)
- No valid action can occur via the spending of an ordinal via transaction fee. If it occurs during the inscription process then the resulting inscription is ignored. If it occurs during the second phase of the transfer process, the balance is returned to the senders available balance. 
- Number of decimals is always ZERO.
- Maximum supply cannot exceed uint64_max.
- Limit per mint cannot exceed uint64_max.

## Operations

DogeRealCool use Protobuf to transfer drc-20 information, the messages are defined below:
```
syntax = "proto3";
package drc;
option go_package = "drc/drc";

message drc {
    uint32 p = 1;
}

message drc20 {
    uint32 p = 1;
    uint32 op = 2;
    string tick = 3;
    uint64 max = 4;
    uint64 lim = 5;
    uint64 amt = 6;
}
```

### Deploy drc-20

| Key | Required? | Description |
| ----| ---- | ---- |
|p|Yes|Protocol, value is 1 (drc-20)|
|op|Yes|Operation: Type of event, value is 1 (deploy)|
|tick|Yes|Ticker: 4 letter identifier of the drc-20|
|max|Yes|Max supply: set max supply of the drc-20|
|lim|No|Mint limit: If letting users mint to themsleves, limit per ordinal|

### Mint drc-20
| Key | Required? | Description |
| ----| ---- | ---- |
|p|Yes|Protocol, value is 1 (drc-20)|
|op|Yes|Operation: Type of event, value is 2 (mint)|
|tick|Yes|Ticker: 4 letter identifier of the drc-20|
|amt|Yes|Amount to mint: Has to be less than "lim" above if stated|

### Transfer drc-20
| Key | Required? | Description |
| ----| ---- | ---- |
|p|Yes|Protocol, value is 1 (drc-20)|
|op|Yes|Operation: Type of event, value is 2 (transfer)|
|tick|Yes|Ticker: 4 letter identifier of the drc-20|
|amt|Yes|Amount to transfer: States the amount of the drc-20 to transfer.|

## Index Rules
Primarily, the indexing guidelines are pertinent to users who intend to independently index drc-20 state data. Nevertheless, individuals encountering challenges may find this information useful for troubleshooting.

- The "amt" "max" and "lim" must be unsigned integer.
- DogeRealCool does NOT support decimals.
- Inscriptions must have a MIME Type of "application/x-protobuf".
- You can use protobuf message "drc" to decode data first, when you checked p == 1(drc-20), you can use "drc20" to decode data again. There will be more "p" in the future.
- Leading or trailing spaces/tabs/newlines are allowed (and stripped/trimmed).
- If op is deploy, message must have a "max" field. "lim" fields are optional. If "lim" is not set it will be equal to "max".
- If op is mint or transfer, message must have an "amt" field.
- 0 for numeric fields is invalid except for the "dec" field.
- Max value of any numeric field is uint64_max.
- "tick" must be 4 bytes wide (UTF-8 is accepted). "tick" is case insensitive, we use lowercase letters to track tickers (convert tick to lowercase before processing).
- If a deploy, mint or transfer is sent as fee to miner while inscribing, it must be ignored
- If a transfer is sent as fee in its first transfer, its amount must be returned to the sender immediately (instead of after all events in the block).
- If a mint has been deployed with more amt than lim, it will be ignored.
- If a transfer has been deployed with more amt than the available balance of that wallet, it will be ignored.
- All balances are followed using scriptPubKey since some wallets may not have an address attached to blockchain.
- First a deploy inscription is inscribed. This will set the rules for this drc-20 ticker. If the same ticker (case insensitive) has already been deployed, the second deployment will be invalid.
- Then anyone can inscribe mint inscriptions with the limits set in deploy inscription until the minted balance reaches to "max" set in deploy inscription.
- When a wallet mints a drc-20 token (inscribes a mint inscription to its address), its overall balance and available balance will increase.
- Wallets can inscribe transfer inscriptions with an amount up to their available balance.
- If a user inscribes a transfer inscription but does not transfer it, its overall balance will stay the same but its available balance will decrease.
- When this transfer inscription is transferred (not sent as fee to miner) the original wallet's overall balance will decrease but its available balance will stay the same. The receiver's overall balance and available balance will increase.
- If you transfer the transfer inscription to your own wallet, your available balance will increase and the transfer inscription will become used.
- A transfer inscription will become invalid/used after its first transfer.
- Buying, transferring mint and deploy inscriptions will not change anyone's balance.
- Balances sent to unspendable outputs are not returned to sender like with the fee instance. They can practically be considered burnt (notwithstanding a bitcoin update that enables transactions to be created with these keys in the future) 

### Points to consider
- JS integer is actually a double floating point number and it can only hold 53-bit integers with full-precision. If you want to make an indexer with JS, use BigInt or Long.js.
- Sent as fee rule is very important and needs to be handled carefully.
- Do not use string length for ticker length comparison. Some UTF-8 characters like most of the emojis use 4 bytes but their string length is 1 on most programming languages.
- Do not trim/strip any field.
- Beware of the reorgs. Since the order of transactions is very important in this protocol, a reorg will probably change the balances.


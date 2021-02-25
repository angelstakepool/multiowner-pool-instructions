# Instructions (General workflow) on setting up a multi-owner pool (from scratch), having funds secured in hw wallet devices

*Both owners use hw wallets for secured PLEDGE*

Note: A single owner pool with pledge secured in hw wallet, is a **particular** case of this

## Requirements

Both owners would need to install cardano-hw-cli in a desktop with connected hw wallet<BR>
https://github.com/vacuumlabs/cardano-hw-cli

## STEP 1 Create CLI keys and address

Generate cli-payment and cli-stake-reward keys, derive address and fund it for pool deposit
```
cardano-cli address key-gen ...
cardano-cli stake-address key-gen ...
cardano-cli address build ...
cardano-cli stake-address build ...

Fund cli-payment with ~510 ADA
```
## STEP 2 Create node-cold, vrf and kes keys
```
cardano-cli node key-gen ...
cardano-cli node key-gen-VRF ...
cardano-cli node issue-op-cert ...
```
## STEP 3 Register cli-stake-reward on chain
```
cardano-cli stake-address registration-certificate ...
```
## STEP 4 Register single-Owner pool and delegation certificate

Single owner Pool (1st iteration - Using CLI keys for rewards & Owner)
```
cardano-cli stake-pool registration-certificate \
  --cold-verification-key-file node-cold.vkey \
  --vrf-verification-key-file vrf.vkey \
  --pool-pledge 0 \
  --pool-cost 340000000 \
  --pool-margin <pool fee in fraction ie 0.011 for 1.1%> \
  --pool-reward-account-verification-key-file cli-stake-rewards.vkey \
  --pool-owner-stake-verification-key-file cli-stake-rewards.vkey \
  --mainnet \
  --single-host-pool-relay <IP of public relay> --pool-relay-port <port> \
  --metadata-url <domain>/pool-name.json \
  --metadata-hash <hash of metadata> \
  --out-file pool.cert
```
Create delegation certificate
```
cardano-cli stake-address delegation-certificate \
  --staking-verification-key-file cli-stake-rewards.vkey \
  --stake-pool-verification-key-file node-cold.vkey \
  --out-file delegation.cert
```
Build transaction including
```
  --certificate-file pool.cert \
  --certificate-file delegation.cert \
```
Sign it with following private keys
```
  --signing-key-file cli-payment.skey \
  --signing-key-file cli-stake-rewards.skey \
  --signing-key-file node-cold.skey \
 ```
and submit it to the chain
```
cardano-cli shelley transaction submit
```

## STEP 5 Delegate hw wallets to registered pool using Daedalus or Yoroi

Both owner and co-owner would delegate to this pool using interactive delegation in either Daedalus or Yoroi (~2.18 ADA)

## STEP 6 Export hw wallets public keys

Owner:
```
cardano-hw-cli address key-gen
  --path 1852H/1815H/0H/2/0
  --verification-key-file hw-stake1.vkey
  --hw-signing-file stake1.hwsfile
```
Co-owner:
```
cardano-hw-cli address key-gen
  --path 1852H/1815H/0H/2/0
  --verification-key-file hw-stake2.vkey
  --hw-signing-file stake2.hwsfile
 ```
## STEP 7 Register multi-owner pool certificate

Get **hw-stake2.vkey** from co-owner

Create another pool certificate (iteration 2) using public keys from both hw wallet owners (hw-stake1.vkey & hw-stake2.vkey), specifying cli-stake-rewards.vkey for rewards (rewards can only go to a single account, so they would need to be distributed manually after each epoch); cli-stake-rewards.vkey could also be left [optionally] as an owner, which might be useful in some scenarios. Effective pledge would be the sum of all the balances in all the owners
```
cardano-cli stake-pool registration-certificate \
  --cold-verification-key-file node-cold.vkey \
  --vrf-verification-key-file vrf.vkey \
  --pool-pledge <agreed-pledge> \
  --pool-cost <FixedFee> \
  --pool-margin <pool fee in fraction ie 0.011 for 1.1%> \
  --pool-reward-account-verification-key-file cli-stake-rewards.vkey \
  --pool-owner-stake-verification-key-file hw-stake1.vkey \
  --pool-owner-stake-verification-key-file hw-stake2.vkey \
  --pool-owner-stake-verification-key-file cli-stake-rewards.vkey \   ->  [ optional]
  --mainnet \
  --single-host-pool-relay <IP of public relay> --pool-relay-port <port> \
  --metadata-url <domain>/pool-name.json \
  --metadata-hash <hash of metadata> \
  --out-file pool2.cert
```

Then create a transaction tx-pool.raw that includes this pool certificate:
```
cardano-cli transaction build-raw \
     --tx-in <utxo> \
     --tx-out $(cat cli-payment.addr)+returnChange \
     --invalid-hereafter <ttl> \
     --fee <fee> \
     --certificate-file pool2.cert \
     --out-file tx-pool.raw
```

This transaction must be signed using witnesses (multi-sig)

we require 4 witnesses
  - node-cold.vkey
  - hw-stake1.vkey
  - hw-stake2.vkey
  - cli-payment (that I use to pay for tx fees and the 500 ADA deposit)


In offline server:

*node-cold*
```
cardano-cli transaction witness \
  --tx-body-file tx-pool.raw \
  --signing-key-file node-cold.skey \
  --mainnet \
  --out-file node-cold.witness
```
*cli-payment*
```
cardano-cli transaction witness \
  --tx-body-file tx-pool.raw \
  --signing-key-file cli-payment.skey \
  --mainnet \
  --out-file cli-payment.witness
```
then we need to copy tx-pool.raw to desktop

First create a witness using my hw-stake1.vkey in desktop (where ledger is connected)

*My hw-stake1 key*
```
cardano-hw-cli transaction witness
  --tx-body-file tx-pool.raw
  --hw-signing-file stake1.hwsfile
  --mainnet
  --out-file hw-stake1.witness
```
and then send **tx-pool.raw** to co-owner, who will create a witness as:

*Co-owner hw-stake2 key*
```
cardano-hw-cli transaction witness
  --tx-body-file tx-pool.raw
  --hw-signing-file stake2.hwsfile
  --mainnet
  --out-file hw-stake2.witness
```
then co-owner would send back **hw-stake2.witness**, which will be used to assemble the witnesses and submit the multi-sig transaction, something like this:
```
cardano-cli transaction assemble \
  --tx-body-file tx-pool.raw \
  --witness-file node-cold.witness \
  --witness-file cli-payment.witness \
  --witness-file hw-stake1.witness \
  --witness-file hw-stake2.witness \
  --out-file tx-pool.multisign 
```
then submit tx-pool.multisign in a hot node
```
cardano-cli transaction submit
```

... and Done !!! :+1:

---
description: Here we are adding a HW wallet as second owner so you can pledge from it.
---

# Add HW wallet owner for pool pledge

Make sure you can see your HW wallet on your air-gapped offline machine.

{% tabs %}
{% tab title=" Air-gapped offline machine" %}
```bash
lusb
# If not then visit :
# https://support.ledger.com/hc/en-us/articles/115005165269-Fix-connection-issues
# Linux tab at the bottom.
```
{% endtab %}
{% endtabs %}

Delegate HW wallet to your pool from either Daedalus or Yoroi.

Export your HW wallet public keys.

{% tabs %}
{% tab title="Air-gapped offline machine" %}
```bash
#Install cardano-hw-cli: https://github.com/vacuumlabs/cardano-hw-cli
cardano-hw-cli address key-gen
  --path 1852H/1815H/0H/2/0
  --verification-key-file hw-stake.vkey
  --hw-signing-file hw-stake.hwsfile
```
{% endtab %}
{% endtabs %}

If you are changing your pool metadata json file, remember to calculate the hash of your metadata file and re-upload the updated metadata json file.

{% tabs %}
{% tab title="Block Producer" %}
```bash
cardano-cli stake-pool metadata-hash --pool-metadata-file poolMetaData.json > poolMetaDataHash.txt
```
{% endtab %}
{% endtabs %}

Find the minimum pool cost value.

{% tabs %}
{% tab title="Block Producer" %}
```bash
minPoolCost=$(cat $NODE_HOME/params.json | jq -r .minPoolCost)
echo minPoolCost: ${minPoolCost}
```
{% endtab %}
{% endtabs %}

Create stake-pool registration certificate including HW wallet as second owner and also making it default reward account.

{% hint style="info" %}
Edit this to your own settings!
{% endhint %}

```text
cardano-cli stake-pool registration-certificate \
    --cold-verification-key-file node.vkey \
    --vrf-verification-key-file vrf.vkey \
    --pool-pledge 150000000000 \ ------your pledge in lovelaces
    --pool-cost 340000000 \ ------minimum pool cost value found before
    --pool-margin 0.01 \ ------pool fee in fraction ie 0.01 for 1%
    --pool-reward-account-verification-key-file hw-stake.vkey \ ------HW wallet key
    --pool-owner-stake-verification-key-file stake.vkey \ ------previous CLI key
    --pool-owner-stake-verification-key-file hw-stake.vkey \ ------HW wallet key
    --mainnet \
    --pool-relay-port 6000 \ ------your relay port
    --pool-relay-ipv4 IP \ ------your relay IP
    --metadata-url <url where you uploaded poolMetaData.json> \
    --metadata-hash $(cat poolMetaDataHash.txt) \
    --out-file pool.cert
```

Copy pool.cert to Block Producer.



Find current tip.

{% tabs %}
{% tab title="Block Producer" %}
```text
currentSlot=$(cardano-cli query tip --mainnet | jq -r '.slotNo')
echo Current Slot: $currentSlot
```
{% endtab %}
{% endtabs %}

Calculate payment.addr balance.

{% tabs %}
{% tab title="Block Producer" %}
```text
cardano-cli query utxo \
    --address $(cat payment.addr) \
    --allegra-era \
    --mainnet > fullUtxo.out

tail -n +3 fullUtxo.out | sort -k3 -nr > balance.out

cat balance.out

tx_in=""
total_balance=0
while read -r utxo; do
    in_addr=$(awk '{ print $1 }' <<< "${utxo}")
    idx=$(awk '{ print $2 }' <<< "${utxo}")
    utxo_balance=$(awk '{ print $3 }' <<< "${utxo}")
    total_balance=$((${total_balance}+${utxo_balance}))
    echo TxHash: ${in_addr}#${idx}
    echo ADA: ${utxo_balance}
    tx_in="${tx_in} --tx-in ${in_addr}#${idx}"
done < balance.out
txcnt=$(cat balance.out | wc -l)
echo Total ADA balance: ${total_balance}
echo Number of UTXOs: ${txcnt}
```
{% endtab %}
{% endtabs %}



Block Producer: Calculate payment.addr balance.

cardano-cli query utxo  --address $\(cat payment.addr\)  --allegra-era  --mainnet &gt; fullUtxo.out

tail -n +3 fullUtxo.out \| sort -k3 -nr &gt; balance.out

cat balance.out

tx\_in="" total\_balance=0 while read -r utxo; do in\_addr=$\(awk '{ print $1 }' &lt;&lt;&lt; "${utxo}"\) idx=$\(awk '{ print $2 }' &lt;&lt;&lt; "${utxo}"\) utxo\_balance=$\(awk '{ print $3 }' &lt;&lt;&lt; "${utxo}"\) total\_balance=$\(\(${total\_balance}+${utxo\_balance}\)\) echo TxHash: ${in\_addr}\#${idx} echo ADA: ${utxo\_balance} tx\_in="${tx\_in} --tx-in ${in\_addr}\#${idx}" done &lt; balance.out txcnt=$\(cat balance.out \| wc -l\) echo Total ADA balance: ${total\_balance} echo Number of UTXOs: ${txcnt}

Block Producer: Build raw transaction.

cardano-cli transaction build-raw  ${tx\_in}  --tx-out $\(cat payment.addr\)+${total\_balance}  --invalid-hereafter $\(\( ${currentSlot} + 10000\)\)  --fee 0  --certificate-file pool.cert  --allegra-era  --out-file tx.tmp

Block Producer: Calculate transaction fee

fee=$\(cardano-cli transaction calculate-min-fee  --tx-body-file tx.tmp  --tx-in-count ${txcnt}  --tx-out-count 1  --mainnet  --witness-count 4  --byron-witness-count 0  --protocol-params-file params.json \| awk '{ print $1 }'\) echo fee: $fee

Block Producer: Calculate final txOut

txOut=$\(\(${total\_balance}-${fee}\)\) echo txOut: ${txOut}

Block Producer: cardano-cli transaction build-raw  ${tx\_in}  --tx-out $\(cat payment.addr\)+${txOut}  --invalid-hereafter $\(\( ${currentSlot} + 10000\)\)  --fee ${fee}  --certificate-file pool.cert  --allegra-era  --out-file tx.raw

Copy tx.raw to air-gapped machine for signing.

Air-gapped machine: Create transaction witnesses from all used signing-keys.

cardano-cli transaction witness --tx-body-file tx.raw --signing-key-file node.skey --mainnet --out-file node-cold.witness cardano-cli transaction witness --tx-body-file tx.raw --signing-key-file stake.skey --mainnet --out-file cli-stake.witness cardano-cli transaction witness --tx-body-file tx.raw --signing-key-file payment.skey --mainnet --out-file cli-payment.witness HW witness creation using cardano-hw-cli: cardano-hw-cli transaction witness --tx-body-file tx.raw --hw-signing-file stake.hwsfile --mainnet --out-file hw-stake.witness

Air-gapped machine: Assemble final transaction with all witnesses. cardano-cli transaction assemble --tx-body-file tx.raw --witness-file node-cold.witness --witness-file cli-stake.witness --witness-file cli-payment.witness --witness-file hw-stake.witness --out-file tx-pool.multisign

Copy tx-pool.multisign to your Block Producer.

Block Producer: Submit final transaction. cardano-cli transaction submit --tx-file tx-pool.multisign --mainnet

Block Producer: Verification on next epoch start. cardano-cli query ledger-state --mainnet --allegra-era --out-file ledger-state.json jq -r '.esLState.\_delegationState.\_pstate.\_pParams."'"$\(cat stakepoolid.txt\)"'" // empty' ledger-state.json


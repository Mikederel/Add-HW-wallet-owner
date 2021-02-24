# README

Delegate your HW wallet to your pool from either Daedalus or Yoroi.

Air-gapped: Install cardano-hw-cli [https://github.com/vacuumlabs/cardano-hw-cli](https://github.com/vacuumlabs/cardano-hw-cli) Air-gapped: Export HW wallet public keys.

```bash
cardano-hw-cli address key-gen
  --path 1852H/1815H/0H/2/0
  --verification-key-file hw-stake.vkey
  --hw-signing-file hw-stake.hwsfile
```

Block Producer: If you're changing your pool metadata json file, remember to calculate the hash of your metadata file and re-upload the updated metadata json file.

```bash
cardano-cli stake-pool metadata-hash --pool-metadata-file poolMetaData.json > poolMetaDataHash.txt
```

Block Producer: Find the minimum pool cost value.

\`\`\`bash

minPoolCost=$\(cat $NODE\_HOME/params.json \| jq -r .minPoolCost\) echo minPoolCost: ${minPoolCost}

\`\`\`

Air-gapped machine: Create stake-pool registration certificate including HW wallet as second owner and reward account.

cardano-cli stake-pool registration-certificate  --cold-verification-key-file /opt/cardano/cnode/node.vkey  --vrf-verification-key-file /opt/cardano/cnode/vrf.vkey  --pool-pledge 150000000000  --pool-cost 340000000  --pool-margin 0.01  --pool-reward-account-verification-key-file /opt/cardano/cnode/priv/wallet/owner2/stake.vkey  --pool-owner-stake-verification-key-file /opt/cardano/cnode/stake.vkey  --pool-owner-stake-verification-key-file /opt/cardano/cnode/priv/wallet/owner2/stake.vkey  --mainnet  --pool-relay-port 6000  --pool-relay-ipv4 35.203.77.113  --metadata-url [https://raw.githubusercontent.com/Mikederel/c/main/f.json](https://raw.githubusercontent.com/Mikederel/c/main/f.json)  --metadata-hash ddebec9861159c63e9d3a8fa13cad142e2dbd3c960c0bf5d3ff1fbaeb8052954  --out-file pool.cert

Copy pool.cert to Block Producer.

Block Producer: Find current tip.

currentSlot=$\(cardano-cli query tip --mainnet \| jq -r '.slotNo'\) echo Current Slot: $currentSlot

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


# Arbitrum One RPC: Reads, Writes, and L2 Confirmation

Arbitrum One is EVM-compatible, but an RPC client benefits from understanding one Arbitrum-specific distinction immediately: **a general-purpose RPC endpoint and the sequencer submission endpoint do not have the same role**.

For normal application reads and writes, this guide uses a provider endpoint:

```bash
export ARB_RPC=https://arbitrum.api.onfinality.io/public
```

Arbitrum One uses chain ID `42161` (`0xa4b1`).

## Which endpoint are you talking to?

Arbitrum's documentation describes two important endpoint types:

| Endpoint type | What it is for |
| --- | --- |
| General RPC | Normal Ethereum-compatible reads and writes |
| Direct sequencer endpoint | Transaction submission only |

The public sequencer endpoint accepts only `eth_sendRawTransaction` and `eth_sendRawTransactionConditional`. It is not a replacement for a normal RPC provider.

That means calls such as `eth_getBalance`, `eth_call`, `eth_getLogs`, and receipt queries belong on a general RPC endpoint such as the one used in this tutorial.

## 1. Verify Arbitrum One

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

Expected result:

```json
{"jsonrpc":"2.0","id":1,"result":"0xa4b1"}
```

Do this check when a script can be configured for several EVM chains. Sending an otherwise valid transaction to the wrong chain is an application error that JSON-RPC compatibility cannot prevent.

## 2. Read the current L2 head

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

Arbitrum L2 block numbers are their own sequence. Do not compare an Arbitrum block number directly with an Ethereum L1 block number.

Fetch the full L2 block header and transaction hashes:

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBlockByNumber",
    "params":["latest",false]
  }'
```

## 3. Read state exactly like an EVM application

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBalance",
    "params":["0xYOUR_ADDRESS","latest"]
  }'
```

Contract reads use ordinary `eth_call`:

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_call",
    "params":[
      {"to":"0xCONTRACT_ADDRESS","data":"0xABI_ENCODED_CALLDATA"},
      "latest"
    ]
  }'
```

This is why most Ethereum libraries can be pointed at Arbitrum by changing the chain configuration.

## 4. Treat the receipt as L2 execution evidence

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getTransactionReceipt",
    "params":["0xTRANSACTION_HASH"]
  }'
```

Once a receipt exists, the transaction has executed in an Arbitrum L2 block. Check `status` before considering the application action successful.

But this is not the end of the settlement story.

## L2 inclusion is not the same thing as Ethereum finality

Arbitrum's sequencer orders L2 transactions quickly. The Arbitrum documentation explicitly distinguishes that soft confirmation from later parent-chain posting and finality.

If your application only needs responsive UX, an L2 receipt may be enough to update the interface. If it protects high-value withdrawals, accounting, or cross-chain actions, track the stronger settlement condition your risk model actually requires.

A good data model often stores separate timestamps/states for:

- transaction submitted;
- included and executed on Arbitrum;
- batch/data posted to the parent chain, if relevant to the workflow;
- finality condition satisfied.

## 5. Estimate an Arbitrum transaction

The normal RPC entry point is still `eth_estimateGas`:

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_estimateGas",
    "params":[{
      "from":"0xYOUR_ADDRESS",
      "to":"0xDESTINATION",
      "value":"0x0",
      "data":"0x"
    }]
  }'
```

Do not assume the final fee behaves exactly like Ethereum Mainnet just because the RPC method name is the same. Arbitrum fees include L2 execution economics and costs related to posting data to the parent chain.

For advanced fee inspection, Arbitrum exposes chain-specific precompiles such as `ArbGasInfo`; use the official precompile documentation or an Arbitrum-aware SDK rather than hard-coding undocumented assumptions.

## 6. Query logs for an L2 indexer

```bash
curl -s "$ARB_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getLogs",
    "params":[{
      "fromBlock":"0xSTART_BLOCK",
      "toBlock":"0xEND_BLOCK",
      "address":"0xCONTRACT_ADDRESS"
    }]
  }'
```

Checkpoint Arbitrum block numbers, not Ethereum block numbers. If you also need to correlate an L2 action with L1 bridge or rollup contracts, store those references separately.

## 7. Use viem as an Arbitrum-aware client

```bash
npm install viem
```

```js
import {createPublicClient, http} from 'viem';
import {arbitrum} from 'viem/chains';

const client = createPublicClient({
  chain: arbitrum,
  transport: http('https://arbitrum.api.onfinality.io/public'),
});

console.log('chain id:', await client.getChainId());
console.log('block:', await client.getBlockNumber());
```

The library handles hex conversion and standard EVM RPC response shapes while the chain object prevents accidental mainnet/Arbitrum configuration mixing.

## When the direct sequencer endpoint matters

The direct sequencer endpoint is a specialized submission path. According to Arbitrum's docs, it accepts raw transaction submission methods rather than general reads. A successful `eth_sendRawTransaction` response there means the sequencer has ordered/executed the transaction in an L2 block, but it still does not represent parent-chain finality.

For most applications, a normal provider endpoint is simpler because the same endpoint can serve reads, simulations, logs, receipts, and transaction submission.

## References

- [Arbitrum chain information](https://docs.arbitrum.io/for-devs/dev-tools-and-resources/chain-info)
- [Arbitrum documentation](https://docs.arbitrum.io/)
- [OnFinality Arbitrum RPC](https://onfinality.io/en/networks/arbitrum)
- [OnFinality network directory](https://onfinality.io/en/networks)

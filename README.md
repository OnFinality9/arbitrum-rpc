# Arbitrum RPC Getting Started

A practical guide to working with Arbitrum One through its Ethereum-compatible JSON-RPC interface.

## RPC endpoint

```text
https://arbitrum.api.onfinality.io/public
```

Arbitrum One is an Ethereum Layer 2. Standard EVM methods work, but it is a separate chain with its own chain ID and L1/L2 settlement model.

## 1. Verify Arbitrum One

Arbitrum One uses chain ID `42161` (`0xa4b1`).

```bash
curl -s https://arbitrum.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

## 2. Read the latest L2 block

```bash
curl -s https://arbitrum.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

Arbitrum block numbers and Ethereum L1 block numbers are separate sequences.

## 3. Read an ETH balance

```bash
curl -s https://arbitrum.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBalance",
    "params":["0xYOUR_ADDRESS","latest"]
  }'
```

## 4. Call a contract

```bash
curl -s https://arbitrum.api.onfinality.io/public \
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

## 5. Inspect a transaction receipt

```bash
curl -s https://arbitrum.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getTransactionReceipt",
    "params":["0xTRANSACTION_HASH"]
  }'
```

A `null` result generally means the transaction is not yet visible in the node's canonical view or the hash is wrong.

## 6. Estimate gas

```bash
curl -s https://arbitrum.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_estimateGas",
    "params":[{"from":"0xYOUR_ADDRESS","to":"0xDESTINATION","value":"0x0","data":"0x"}]
  }'
```

## 7. JavaScript example

```js
const RPC_URL = 'https://arbitrum.api.onfinality.io/public';

async function rpc(method, params = []) {
  const res = await fetch(RPC_URL, {
    method: 'POST',
    headers: {'content-type': 'application/json'},
    body: JSON.stringify({jsonrpc: '2.0', id: 1, method, params}),
  });

  const body = await res.json();
  if (body.error) throw new Error(body.error.message);
  return body.result;
}

const chainId = Number.parseInt(await rpc('eth_chainId'), 16);
if (chainId !== 42161) throw new Error(`Unexpected chain: ${chainId}`);

console.log(Number.parseInt(await rpc('eth_blockNumber'), 16));
```

## Arbitrum-specific notes

### L2 confirmation and L1 finality are different concepts

A transaction can be visible on Arbitrum before every L1 settlement assumption relevant to your application is satisfied. Choose confirmation rules according to your risk model.

### Bridging is not a normal same-chain transfer

Deposits and withdrawals involve bridge contracts and cross-chain messages.

### Explorer mismatch often means network mismatch

Verify that you are using Arbitrum One rather than Ethereum, Arbitrum Nova, or a testnet.

## Mainnet settings

| Setting | Value |
| --- | --- |
| Network | Arbitrum One |
| Chain ID | `42161` |
| Native token | ETH |
| RPC | `https://arbitrum.api.onfinality.io/public` |
| Explorer | `https://arbiscan.io` |

## Resources

- [Arbitrum documentation](https://docs.arbitrum.io/)
- [OnFinality Arbitrum RPC](https://onfinality.io/en/networks/arbitrum)
- [OnFinality RPC network directory](https://onfinality.io/en/networks)

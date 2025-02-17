---
sidebar_position: 2
description: Reference page of the squid-evm-typegen command line tool
---

# Squid EVM typegen

**Since `@subsquid/evm-typegen@2.0.0`**

The `evm-typegen(1)` tool generates TypeScript facades for EVM transactions, logs and `eth_call` queries.

The generated facades are assumed to be used by squids indexing EVM data.

The ABI file can be specified in three ways:

- as a plain JSON file:

```bash
npx squid-evm-typegen src/abi erc20.json
```

- as a contract address (to fetch ABI from the Etherscan API). Once can pass multiple addresses at once.

```bash
npx squid-evm-typegen src/abi 0xBB9bc244D798123fDe783fCc1C72d3Bb8C189413
```

- as an arbitrary URL:
```bash
npx squid-evm-typegen src/abi https://example.com/erc721.json
```

In all cases typegen will use ABI's `basename` as the root name for the generated files. You can overwrite the basename of generated files using the fragment `(#)` suffix.

```bash
squid-evm-typegen src/abi 0xBB9bc244D798123fDe783fCc1C72d3Bb8C189413#contract   
```

**Arguments:**

|                        |                                                           |
|------------------------|-----------------------------------------------------------|
|  `output-dir`          | output directory for generated definitions                |
|  `abi`                 | A contract address, a URL or local path to the ABI file.  Accepts multiple contracts. |


**Options:**

|                           |                                                          |
|---------------------------|----------------------------------------------------------| 
|  `--multicall`            | generate a facade for the MakerDAO multicall contract. May significantly improve the performance of contract state calls by batching RPC requests (see below)   |
|  `--etherscan-api <url>`  | etherscan API to fetch contract ABI by a known address. By default, `https://api.etherscan.io/`   |
|  `--clean`                | delete output directory before run                       |
|  `-h, --help`             | display help for command                                 |


## Usage 

### Events 

The EVM log data is provided by the `event` object with wrappers for each event defined in the ABI. 

**Example**

Subscribe to topics:

```ts
// generated by evm-typegen
import { events } from "./abi/weth";

const processor = new EvmBatchProcessor()
  .setDataSource({
    archive: 'https://eth.archive.subsquid.io',
  })
  .addLog([
    '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'
  ], {
    filter: [[
      events.Deposit.topic, events.Withdrawal.topic
    ]],
    data: {
      evmLog: {
          topics: true,
          data: true,
      },
    } as const,
  })
```

Decode event data:
```ts
processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
      if (i.kind === 'evmLog' && i.evmLog.topics[0] == events.Deposit.topic) {
        // type-safe decoding of the Deposit event data
        const amt = events.Deposit.decode(i.evmLog).wad
      }
    }
});
```

### Transactions

Similar to `events`, transaction access is provided by the `functions` object for each contract method defined in the ABI. 

### Contract state calls

The typegen creates a wrapper contract classes for each contract Create a contract instance using the processor context and the block height at which the state should be queried.

**Example**

```ts
// generated by evm-typegen
import { Contract } from "./abi/weth";

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
      // call totalSupply() at the current block
      let supply = await (new Contract(ctx, c, '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2').totalSupply())
    }
  }
});
```

### Batching contract state calls using the Multicall contract

The [MakerDAO Multicall contract](https://github.com/makerdao/multicall) was designed to reduce the number of roundtrips to a JSON RPC node. In the context of indexing, it normally significantly improves the indexing speed since JSON RPC calls is typically a bottleneck.

Multicall contracts are deployed in many EVM chains, see the [contract repo](https://github.com/makerdao/multicall) for the addresses. The `Multicall` facade exposes the method
```ts
tryAggregate<Args extends any[], R>(
        func: Func<Args, {}, R>,
        calls: [address: string, args: Args][],
        paging?: number
    ): Promise<MulticallResult<R>[]>
```
The arguments are as follows:
- `func`: the contract function to be called
- `calls`: an array of tuples `[contractAddress: string, args]`. Each specified contract will be called with the specified arguments.
- `paging` an (optional) argument for the maximal number of calls to be batched into a single JSON PRC request. Note that large page sizes may cause timeouts.

A typical usage is as follows:
```ts
// generated by evm-typegen
import { functions } from "./abi/mycontract";
import { Multicall } from "./abi/multicall";

const MY_CONTRACT='0xac5c7493036de60e63eb81c5e9a440b42f47ebf5'

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
        // some logic
    }
  }
  const lastBlock = ctx.blocks[ctx.blocks.length - 1];
  // Multicall address for Ethereum is 0x5ba1e12693dc8f9c48aad8770482f4739beed696
  const multicall = new Multicall(ctx, lastBlock, '0x5ba1e12693dc8f9c48aad8770482f4739beed696')
  // call MY_CONTRACT.myContractMethod("foo") and MY_CONTRACT.myContractMethod("bar")
  const args = ["foo", "bar"]
  const results = await multicall.tryAggregate(functions.myContractMethod, args.map(a => [MY_CONTRACT, a]) as [string, any[]], 100);

  results.forEach((res, i) => {
    if (res.success) {
      ctx.log.info(`Result for argument ${args[i]} is ${res.value}`);
    }
  }) 
});
```

## Migration from `evm-typegen@1.x`

1. In `evm-typegen@2.0`, the `--abi`, and `--output` flags are replaced by the positional arguments. For example, the command
```sh
npx squid-evm-typegen --abi src/abi/ERC721.json --output src/abi/erc721.ts
```
is replaced with
```sh
npx squid-evm-typegen src/abi src/abi/ERC721.json
```

2. The `events` object generated by `evm-typegen@2.0` exposes events by name, not by the full signature. For example,
```ts
events['UpdatedGravatar(uint256,address,string,string)'].decode(evmLog)
```
should be replaced with a more readable
```ts
events.UpdatedGravatar.decode(evmLog)
```
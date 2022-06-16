# Starknet Indexer

**Starknet Indexer** and **starknet-archive** are working titles for the software that aims to solve the problem most DApp 
developers face: the data their smart contracts produce lie in transaction inputs and events scattered in blocks. 
These data need to be gathered, decoded and interpreted for analysis (think an up to date TVL) 
and by end users (like a daily balance). 

This problem is solved by services called **indexers** that usually listen 
to blockchain events specified by developers then run code over them to parse and interpret them. 
The code to interpret events is usually written in high level languages by the DApp developers themselves, 
run by third parties and sometimes in a decentralized manner.

While this multi step approach gets the job done it involves effort better spend on developing the DApp and creates friction between the many parts of the process.

Our approach is a centralised service offering already decoded and normalized data ready for interpretation, 
so that the DApp developers can start using the data right away without the need to write extra code or run several processes and involve third party indexers.

We invite you to a sneak preview of this service available in a GraphQL query studio at 



Below are example queries demonstrating its basic capabilities. 

Much more to come: custom functions, triggers, events, remote schemas. Stay tuned. 

# GraphQL Queries

Below are some of the example queries that show 

## Input and event data decoded as per contract abi

Let's start with a query for a block with its transactions and events. It may be familiar to you as a common query to the chain API. 
This is the raw block 100000 as we received it.
```graphql
query raw_block {
  raw_block_by_pk(block_number: 100000) {
    raw
  }
}
```

The raw block has transaction inputs and event payloads in binary form which is hard to interpret.
```json
...
            {
                "type": "INVOKE_FUNCTION",
                "max_fee": "0x0",
                "calldata": [
                  "0x4bc8ac16658025bff4a3bd0760e84fcf075417a4c55c6fae716efdd8f1ed26c",
                  "0x219209e083275171774dab1df80982e9df2096516f06319c5c6d71ae0a8480c",
                  "0x3",
                  "0x263acca23357479031157e30053fe10598077f24f427ac1b1de85487f5cd124",
                  "0x204fce5e3e25026110000000",
                  "0x0",
                  "0x64"
                ]
}
...

"events": [
              {
                "data": [
                  "0x0",
                  "0x1778c6596d715a8613d0abcbe4fc08c052d208dce3b43eeb6b4dc24ddd62ed9",
                  "0x3d5b",
                  "0x0"
                ],
                "keys": [
                  "0x99cd8bde557814842a3121e8ddfd433a539b8c9f14bf31ebf108d12e6196e9"
                ],
                "from_address": "0x4e34321e0bce0e4ff8ff0bcb3a9a030d423bca29a9d99cbcdd60edb9a2bf03a"
              }
            ],
...
```

Now let's look at the same block parsed and decoded (most fields omitted for brevity).
```graphql
query block {
  block(where: {block_number: {_eq: 100000}}) {
    transactions {
      function
      entry_point_selector
      inputs {
        name
        type
        value
      }
      events {
        name
        transmitter_contract
        arguments {
          name
          type
          value
          decimal
        }
      }
    }
  }
}
```

See the function and its inputs decoded using the contract's abi: `execute`, `to`, `calldata` etc.
```json
          {
            "function": "execute",
            "entry_point_selector": "0x240060cdb34fcc260f41eac7474ee1d7c80b7e3607daff9ac67c7ea2ebb1c44",
            "inputs": [
              {
                "name": "to",
                "type": "felt",
                "value": "0x4bc8ac16658025bff4a3bd0760e84fcf075417a4c55c6fae716efdd8f1ed26c"
              },
              {
                "name": "selector",
                "type": "felt",
                "value": "0x219209e083275171774dab1df80982e9df2096516f06319c5c6d71ae0a8480c"
              },
              {
                "name": "calldata",
                "type": "felt[3]",
                "value": [
                  "0x263acca23357479031157e30053fe10598077f24f427ac1b1de85487f5cd124",
                  "0x204fce5e3e25026110000000",
                  "0x0"
                ]
              },
              {
                "name": "nonce",
                "type": "felt",
                "value": "0x64"
              }
            ]
            }

``` 

Events are also decoded, see `Transfer` and its payload `tokenId` as `Uint256`:

```json
"events": [
          {
            "name": "Transfer",
            "transmitter_contract": "0x4e34321e0bce0e4ff8ff0bcb3a9a030d423bca29a9d99cbcdd60edb9a2bf03a",
            "arguments": [
              {
                "name": "from_",
                "type": "felt",
                "value": "0x0",
                "decimal": "0"
              },
              {
                "name": "to",
                "type": "felt",
                "value": "0x1778c6596d715a8613d0abcbe4fc08c052d208dce3b43eeb6b4dc24ddd62ed9",
                "decimal": "663536632620382607614490239145922341009321511960837718021901264100395462361"
              },
              {
                "name": "tokenId",
                "type": "Uint256",
                "value": {
                  "low": "0x3d5b",
                  "high": "0x0"
                },
                "decimal": "15707"
              }
            ]
          }
        ]
```

But you are probably interested in events emitted by your own contract. 
Let's narrow down to this query for `Mint` events of contract `0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733`:
```graphql
query events {
  event(where: {name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}, limit: 10, order_by: {id: desc}) {
    name
    arguments {
      name
      type
      value
      decimal
    }
    transaction_hash
  }
}
```

## Handling proxy events

What if you use proxy contracts? Their implementation contracts change and so do their abi. 
Your data are still decoded, but by the implementation contract's abi. 

This queries for 3 transactions sent to a proxy contract  `0x47495c732aa419dfecb43a2a78b4df926fddb251c7de0e88eab90d8a0399cd8`.
You see the first `DEPLOY` transaction setting the implementation contract address to `0x90aa7a9203bff78bfb24f0753c180a33d4bad95b1f4f510b36b00993815704`.
The same query gets abi for both proxy and implementation contracts.
```graphql
query proxy_inputs {
  transaction(limit: 3, where: {contract_address: {_eq: "0x47495c732aa419dfecb43a2a78b4df926fddb251c7de0e88eab90d8a0399cd8"}}) {
    inputs {
      type
      value
      name
    }
    function
  }
  raw_abi(where: {contract_address: {_in: ["0x47495c732aa419dfecb43a2a78b4df926fddb251c7de0e88eab90d8a0399cd8", "0x90aa7a9203bff78bfb24f0753c180a33d4bad95b1f4f510b36b00993815704"]}}) {
    contract_address
    raw
  }
}
```

Note for example the input `call_array` of type `CallArray` is defined in the implementation not proxy abi, and is decoded properly. 
```json
...

"inputs": [
          {
            "type": "CallArray[1]",
            "value": [
              {
                "to": "0x4c5327da1f289477951750208c9f97ca0f53afcd256d4363060268750b07f92",
                "data_len": "0x3",
                "selector": "0x219209e083275171774dab1df80982e9df2096516f06319c5c6d71ae0a8480c",
                "data_offset": "0x0"
              }
            ],
            "name": "call_array"
          },
          {
            "type": "felt[3]",
            "value": [
              "0x30295374333e5b9fc34de3ef3822867eaa99af4c856ecf624d34574f8d7d8ea",
              "0xffffffffffffffffffffffffffffffff",
              "0xffffffffffffffffffffffffffffffff"
            ],
            "name": "calldata"
          },
          {
            "type": "felt",
            "value": "0x0",
            "name": "nonce"
          }
        ],
        "function": "__execute__"
...

"contract_address": "0x90aa7a9203bff78bfb24f0753c180a33d4bad95b1f4f510b36b00993815704",
        "raw": [
          {
            "name": "CallArray",
            "size": 4,
            "type": "struct",
            "members": [
              {
                "name": "to",
                "type": "felt",
                "offset": 0
              },
              {
                "name": "selector",
                "type": "felt",
                "offset": 1
              },
              {
                "name": "data_offset",
                "type": "felt",
                "offset": 2
              },
              {
                "name": "data_len",
                "type": "felt",
                "offset": 3
              }
            ]
          },
          
...
```

## Aggregation queries

So you can query for all your events and function calls, but how do you interpret these data? Let's say you want to derive a number from some of your events, 
for example, calculate Total Value Locked, which is a sum of arguments `amount0` of all `Mint` events. You can certainly query for the values:
```graphql
query Mint_amount0 {
  argument(where: {name: {_eq: "amount0"}, event: {name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}}, limit: 10) {
    type
    value
    name
    decimal
  }
}
```

See the values as `Uint256` struct and also conveniently converted into decimals.
```json
"argument": [
      {
        "type": "Uint256",
        "value": {
          "low": "0x52b7d2dcc80cd2e4000000",
          "high": "0x0"
        },
        "name": "amount0",
        "decimal": "100000000000000000000000000"
      },
```

You can consume this query's results by your software and sum it up, like some other indexers let you do.
But here, you can run an **aggregation** query that sums over the values and returns the result, without much effort.
That's why the values were converted into decimals: `numeric 78` database types large enough to support Uint256 and arithmetic operations over them.
```graphql
query TVL {
  argument_aggregate(where: {name: {_eq: "amount0"}, event: {name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}}) {
    aggregate {
      sum {
        decimal
      }
      avg {
        decimal
      }
      min {
        decimal
      }
      max {
        decimal
      }
    }
  }
}
```
This query returns the total sum (TVL) as well as results of other aggregation functions: min, max, avg.
```json
{
  "data": {
    "argument_aggregate": {
      "aggregate": {
        "sum": {
          "decimal": "7312519852578770281612328156"
        },
        "avg": {
          "decimal": "148767543894266393001838"
        },
        "min": {
          "decimal": "25047631971864"
        },
        "max": {
          "decimal": "5000000000000000000000000000"
        }
      }
    }
  }
}
```

## Complex queries

What if filters over chain data and aggregation queries still don't give you the desired results? 
Then use the full power and flexibility of **SQL**: create custom views and functions and query them.
Let's say you want to calculate daily `Mint` volume of your contract. 
This requires summing over your events in all blocks per day derived from the block's timestamp.
```graphql
query daily_mint {
  daily_mint {
    dt
    mint_amount0
  }
}
```

Returns daily sums:
```json
{
  "data": {
    "daily_mint": [
      {
        "dt": "2022-05-26",
        "mint_amount0": "996943538933314245382368"
      },
      {
        "dt": "2022-05-25",
        "mint_amount0": "15916292569039569873451454"
      },
      {
        "dt": "2022-05-24",
        "mint_amount0": "1012580148979965943271803"
      },
      {
        "dt": "2022-05-23",
        "mint_amount0": "847127202523304463244865"
      },
    
```

The query was created from this database view that sums over `Mint` event arguments grouped per day.
```sql
CREATE OR REPLACE VIEW "public"."daily_mint" AS 
 WITH RECURSIVE daily_mint(mint_amount0, dt) AS (
         SELECT sum(a."decimal") AS sum,
            (to_timestamp((b."timestamp")::double precision))::date AS dt
           FROM (((argument a
             LEFT JOIN event e ON ((a.event_id = e.id)))
             LEFT JOIN transaction t ON (((e.transaction_hash)::text = (t.transaction_hash)::text)))
             LEFT JOIN block b ON ((t.block_number = b.block_number)))
          WHERE (((e.transmitter_contract)::text = '0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733'::text) AND ((e.name)::text = 'Mint'::text) AND ((a.name)::text = 'amount0'::text))
          GROUP BY ((to_timestamp((b."timestamp")::double precision))::date)
          ORDER BY ((to_timestamp((b."timestamp")::double precision))::date) DESC
        )
 SELECT daily_mint.mint_amount0,
    daily_mint.dt
   FROM daily_mint;
```

Here's another interesting query
```graphql
query MyQuery {
  daily_transactions {
    count
    date
  }
}
```
calculating total transactions per day:
```json
{
  "data": {
    "daily_transactions": [
      {
        "count": "6778",
        "date": "2022-05-27"
      },
      {
        "count": "39393",
        "date": "2022-05-26"
      },
      {
        "count": "37364",
        "date": "2022-05-25"
      },
 
```
Created from this view:
```sql
create recursive view daily_transactions (count, date) as
  select count(t.transaction_hash), to_timestamp(b.timestamp)::date as dt from transaction as t
    left join block b on t.block_number = b.block_number
  group by dt
  order by dt desc;
```

Another one for the mix
```graphql
query MyQuery {
  top_functions {
    count
    name
  }
}
```
returning functions called the most:
```json
{
  "data": {
    "top_functions": [
      {
        "count": "2031870",
        "name": "__execute__"
      },
      {
        "count": "1408924",
        "name": "execute"
      },
      {
        "count": "468926",
        "name": "constructor"
      },
      {
        "count": "275052",
        "name": "initialize"
      },

```
Created from this view:
```sql
CREATE
OR REPLACE VIEW "public"."top_functions" AS WITH RECURSIVE top_functions(name, count) AS (
  SELECT
    t.function,
    count(t.function) AS ct
  FROM
    transaction t
  GROUP BY
    t.function
  ORDER BY
    (count(t.function)) DESC
)
SELECT
  top_functions.name,
  top_functions.count
FROM
  top_functions;
```

```bash
gq https://starknet-archive.hasura.app/v1/graphql -H 'x-hasura-admin-secret: 6Knsw4W8DgfxqJcipnf5r1SVmdg2I3PU4EBm1tU4ZVnVFD5S7nmry2A9vnbizFbS' -i
```

```bash
curl https://starknet-archive.hasura.app/api/rest/my_events2 -H 'x-hasura-admin-secret: 6Knsw4W8DgfxqJcipnf5r1SVmdg2I3PU4EBm1tU4ZVnVFD5S7nmry2A9vnbizFbS' | jq
```

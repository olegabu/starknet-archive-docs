# Starknet Indexer

**Starknet Indexer** and **starknet-archive** are working titles for the software that aims to solve the problem most DApp 
developers face: the data their smart contracts produce is buried in transaction inputs and events scattered in blocks. 
These data need to be gathered, decoded and interpreted for analysis (think an up to date TVL) 
and by to be presented to the end users. 

This problem is solved by services called **indexers** that listen to blockchain events specified by DApp developers 
then run code over them to parse and interpret. The code to parse events is usually written by the DApp developers themselves, 
run by third parties and sometimes in a decentralized manner.

While this multi step approach gets the job done it requires development effort better spend on the DApp itself, 
and creates friction between the many parts of the process.

Our approach is a centralised service offering already decoded and normalized data ready for consumption interpretation. 
The DApp developers can start using the data right away without the need to write extra code or run several processes 
and involve third party indexers.

We invite you to a sneak preview of this service available in a GraphQL query console at 
[http://54.80.141.84](http://54.80.141.84)

Below are example queries demonstrating its basic capabilities. 

Stay tuned as there's much more to come: direct sql queries, custom views and functions, triggers, events.

## Quick start 

A [GraphQL console](http://54.80.141.84) is open to developers to try out GraphQL queries for events, transactions and their decoded inputs.

![Screenshot-graphiql](https://github.com/olegabu/starknet-archive-docs/blob/main/Screenshot-graphiql.png?raw=true "GraphQL console")

Use the Explorer pane on the left to put together a query by selecting fields and filter parameters.
Or write queries directly into the middle pane.

You can combine queries to return all the data you're looking for in one shot. 
This query requests three `Mint` events and all `DEPLOY` transactions together with their inputs in block 100000.
```graphql
query mint_and_deploy_100000 {
  event(where: {name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}, limit: 3) {
    name
    arguments {
      name
      type
      value
      decimal
    }
    transaction_hash
  }
  block(where: {block_number: {_eq: 100000}}) {
    transactions(where: {type: {_eq: "DEPLOY"}}) {
      function
      entry_point_selector
      inputs {
        name
        type
        value
      }
    }
  }
}
```

You can also get data directly from the http endpoint. Using the query above:
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"query mint_and_deploy_100000 { event(where: {name: {_eq: \"Mint\"}, transmitter_contract: {_eq: \"0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733\"}}, limit: 3) { name arguments { name type value decimal } transaction_hash } block(where: {block_number: {_eq: 100000}}) { transactions(where: {type: {_eq: \"DEPLOY\"}}) { function entry_point_selector inputs { name type value } } }}"}'
```  

## Input and event data decoded as per contract abi

Let's get the whole block with its transactions and events. It may be familiar to you as a common query to the chain API. 
Paste this into the GraphQL query window.
```graphql
query raw_block {
  raw_block_by_pk(block_number: 100000) {
    raw
  }
}
```
You'll see block 100000 as we received it from the sequencer API.

The raw block has transaction inputs in bulk binary form which is hard to interpret.
```json
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
```

Event payload is in bulk as well. 
```json
{
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
    ]
}
```

Now let's look at the same block parsed and decoded. Try this query (most fields omitted for brevity).
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

Transaction functions and their inputs are decoded using the contract's abi: see the function name `execute`, inputs `to`, `calldata` etc.
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

Events are also decoded: see `Transfer` and its argument `tokenId` as `Uint256`, note it is represented in hex and in decimal.
```json
{
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
}
```

While this is useful to explore, can you build analytics tools or a front end with these data? 
Yes, you can make a direct http call and consume query results by other applications. 
Your development process may start with you designing queries in the console, combining and refining them. 
Once you figured out how to collect all the data you need, you can incorporate these query calls into your DApp frontend. 
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"query { block(where: {block_number: {_eq: 100000}}) { transactions { function entry_point_selector inputs { name type value } events { name transmitter_contract arguments { name type value decimal } } } }}"}'
```

## Querying your own events

You are probably interested not in whole blocks but in events emitted by your own contract. 
Let's narrow down to this query for `Mint` events of contract `0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733`, and limit it to one result for brevity.
```graphql
query events {
  event(where: {name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}, limit: 1) {
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

The query returns your event decoded.
```json
{
  "data": {
    "event": [
      {
        "name": "Mint",
        "arguments": [
          {
            "name": "sender",
            "type": "felt",
            "value": "0x1ea2f12a70ad6a052f99a49dace349996a8e968a0d6d4e9ec34e0991e6d5e5e",
            "decimal": "866079946690358847859985129991514658898248253189226492476287621475869744734"
          },
          {
            "name": "amount0",
            "type": "Uint256",
            "value": {
              "low": "0x52b7d2dcc80cd2e4000000",
              "high": "0x0"
            },
            "decimal": "100000000000000000000000000"
          },
          {
            "name": "amount1",
            "type": "Uint256",
            "value": {
              "low": "0x2d79883d2000",
              "high": "0x0"
            },
            "decimal": "50000000000000"
          }
        ],
        "transaction_hash": "0x521e56da1f33412f2f5e81dc585683c47b19783995aa3ebdcd84f5739cea489"
      }
    ]
  }
}
```

Query all for all `Mint` events by a direct http call:
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"query { event(where: {name: {_eq: \"Mint\"}, transmitter_contract: {_eq: \"0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733\"}}) { name arguments { name type value decimal } transaction_hash }}"}'
```

## Query for fields in JSON payloads

If the data you're interested in lies inside json, you can get to it by specifying a path to this field.

This queries for transaction inputs `index_and_x` defined as a tuple.
```graphql
{
  input(where: {name: {_eq: "index_and_x"}, transaction: {contract_address: {_eq: "0x579f32b8090d8d789d4b907a8935a75f6de583c9d60893586e24a83d173b6d5"}}}, limit: 1) {
    value
  }
}
```

Returns json values of `index_and_x`.
```json
{
  "data": {
    "input": [
      {
        "value": {
          "index": "0x39103d23f38a0c91d4eebbc347a5170d00f4022cbb10bfa1def9ad49df782d6",
          "values": [
            "0x586dbbbd0ba18ce0974f88a19489cca5fcd5ce29e723ad9b7d70e2ad9998a81",
            "0x6fefcb8a0e36b801fe98d66dc1513cce456970913b77b8058fea640a69daaa9"
          ]
        }
      }
    ]
  }
}
```

This query digs deeper into json by specifying the path within.
```graphql
{
  input(where: {name: {_eq: "index_and_x"}, transaction: {contract_address: {_eq: "0x579f32b8090d8d789d4b907a8935a75f6de583c9d60893586e24a83d173b6d5"}}}, limit: 1) {
    value(path: "values[1]")
  }
}
```

Returns bare `y` values of `index_and_x`.
```json
{
  "data": {
    "input": [
      {
        "value": "0x6fefcb8a0e36b801fe98d66dc1513cce456970913b77b8058fea640a69daaa9"
      }
    ]
  }
}
```

To demonstrate, let's request this contract's abi by this query.
```graphql
{
  raw_abi_by_pk(contract_address: "0x579f32b8090d8d789d4b907a8935a75f6de583c9d60893586e24a83d173b6d5") {
    raw(path: "[0]")
  }
}
```

Here's the definition of `index_and_x` that shows how to get the second part of tuple `(x : felt, y : felt)` by `path: "values[1]"`
```json
{
  "data": {
    "raw_abi_by_pk": {
      "raw": {
        "name": "IndexAndValues",
        "size": 3,
        "type": "struct",
        "members": [
          {
            "name": "index",
            "type": "felt",
            "offset": 0
          },
          {
            "name": "values",
            "type": "(x : felt, y : felt)",
            "offset": 1
          }
        ]
      }
    }
  }
}
```

## Handling proxy contracts

What if you use proxy contracts? Their implementation contracts change and so do their abi. 
While this may be challenging, the data can still be decoded, by the implementation contract's abi. 

This query requests three transactions sent to a proxy contract  `0x47495c732aa419dfecb43a2a78b4df926fddb251c7de0e88eab90d8a0399cd8`.
You see the first `DEPLOY` transaction setting the implementation contract address to `0x90aa7a9203bff78bfb24f0753c180a33d4bad95b1f4f510b36b00993815704`.
Note the same query gets the abi for both proxy and implementation contracts, for demonstration.
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

See the input `call_array` of type `CallArray` is defined in the implementation, not proxy abi. 
```json
{
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
      }
    ]
}
```

Yet `call_array` is still decoded properly as the function's input. 
```json
{
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
}
```

## Aggregation queries

Now you can query for all your events and function calls, but how do you interpret them? 
Let's say you want to derive a number from some of your events, for example, to calculate Total Value Locked, 
which is a sum of arguments `amount0` of all `Mint` events. 

You can certainly query for the values.
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
But here, you can run an **aggregation** query that sums over the values and returns the final result, without much effort.
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

What if filters over chain data and aggregation queries still don't give you the desired data? 
Then use the full power and flexibility of **SQL**: create custom views and functions and query them.
Let's say you want to calculate the daily `Mint` volume of your contract. 
This requires summing over your events in all blocks per day, which is derived from the block's timestamp.
```graphql
query {
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
      }
    ]
  }
} 
```

The query was created from this database view that sums over `Mint` event arguments grouped per day.
```sql
create recursive view daily_mint(amount0, dt) as
select sum(a.decimal) as sum, (to_timestamp((b."timestamp")))::date AS dt
from argument a left join event e on a.event_id = e.id left join transaction t on e.transaction_hash = t.transaction_hash left join block b on t.block_number = b.block_number
where e.transmitter_contract = '0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733' and e.name = 'Mint' and a.name = 'amount0'
group by dt order by dt desc;
```

Here's another example query that calculates total transactions per day.
```graphql
query {
  daily_transactions(limit: 3) {
    count
    date
  }
}
```

We limited its output to the three last days:
```json
{
  "data": {
    "daily_transactions": [
      {
        "count": "13370",
        "date": "2022-06-08"
      },
      {
        "count": "22068",
        "date": "2022-06-07"
      },
      {
        "count": "47647",
        "date": "2022-06-06"
      }
    ]
  }
}
```

The query selects data from this database view:
```sql
create recursive view daily_transactions (count, date) as
select count(t.transaction_hash), to_timestamp(b.timestamp)::date as dt from transaction as t
left join block b on t.block_number = b.block_number
group by dt order by dt desc;
```
Request all transactions count to date by a direct http call:
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"query {daily_transactions {count date}}"}'
```

Another one for the mix
```graphql
query {
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
      }
    ]
  }
}
```
Created from this view:
```sql
create recursive view top_functions (function, ct) as
select t.function, count(t.function) ct from transaction t group by t.function order by ct desc;
```

The above examples are rather simple but show that you can use full fledged SQL queries which can be rather complex 
and can return, aggregate and calculate over any data you're interested in.

In most cases no separate indexer process is needed to interpret your data. If however you want to do something that SQL, 
even with custom views and functions cannot, query for your specific data and consume the results by a your own process 
and deal with it.  

# Starknet Indexer

**Starknet Indexer** and **starknet-archive** are working titles for the
software that gathers blockchain data, decodes, persists and makes it
available for analysis with SQL, GraphQL and http queries.

* [Approach: write once](#approach--write-once)
* [Preview and road map](#preview-and-road-map)
* [Quick start](#quick-start)
* [Input and event data decoded as per contract ABI](#input-and-event-data-decoded-as-per-contract-abi)
* [Query with http calls](#query-with-http-calls)
* [Query for your contract's events](#query-for-your-contract-s-events)
* [Query for values in JSON payloads](#query-for-values-in-json-payloads)
* [Handling proxy contracts](#handling-proxy-contracts)
* [Aggregation queries](#aggregation-queries)
* [Complex queries from database views](#complex-queries-from-database-views)

## Approach: write once 

We aim to solve the problem most DApp developers face: the data their
smart contracts produce is buried in transaction inputs and events
scattered in blocks. These data need to be gathered, parsed and
interpreted for analysis (think an up-to-date TVL) and, finally,
presented to the end users.

This problem is often solved by an **indexer**, a service that listens
to blockchain events, decodes and persists the emitted data. The code to
interpret events is usually written by the DApp developers themselves
and run by third parties, sometimes in a decentralized manner.

While this multi-step approach gets the job done, it requires
development effort better spent on the DApp itself, and creates friction
between the many parts of the process.

Our approach is a centralised service offering already **decoded and
normalized data** ready for consumption and interpretation. We run one
process to gather data from blockchains, decode it and persist in a
relational database; there is no other secondary indexing or parsing.
Once in the database, the data are already indexed and available for
querying with SQL and GraphQL. Developers can use the up-to-date data 
right away without the need to write extra code, run multiple processes
or involve third party indexers.

## Preview and road map

We invite you to a sneak preview of our indexing service available in a
GraphQL query console at
[http://starknetindex.com/console](http://starknetindex.com/console).

Please see below example queries demonstrating its basic
capabilities.

Stay tuned as there's more to come: 
- [x] subscriptions to updates to your query results for alerting
- [x] direct access to data with sql queries
- [x] custom views and functions
- [ ] charts and dashboards


## Quick start 

A [GraphQL console](http://starknetindex.com/console) is open to
developers to query blockchain data for events, transactions and their
inputs, as well as to filter, aggregate and sum up values. 

![Screenshot-graphiql](https://github.com/olegabu/starknet-archive-docs/blob/main/Screenshot-graphiql.png?raw=true "GraphQL console")

Use the Explorer pane on the left to put together a GraphQL query by
selecting fields and filter parameters, or write queries directly into
the middle pane. Read the results in json in the left pane.

You can combine queries to return all the data you're looking for in one
shot. This example query requests three `Mint` events and all `DEPLOY`
transactions together with their inputs in block 100000.
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

You can get results directly from our http endpoint. Send the query above
with `curl`:
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"query mint_and_deploy_100000 { event(where: {name: {_eq: \"Mint\"}, transmitter_contract: {_eq: \"0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733\"}}, limit: 3) { name arguments { name type value decimal } transaction_hash } block(where: {block_number: {_eq: 100000}}) { transactions(where: {type: {_eq: \"DEPLOY\"}}) { function entry_point_selector inputs { name type value } } }}"}'
```  

## Input and event data decoded as per contract ABI

Blockchain APIs return transaction inputs and events in bulk arrays of
binary data which are hard to interpret. We decode these for you using
appropriate ABIs; special care is taken for proxy contracts (more on
this below).

Take a look at the transactions and events of block 100000 parsed and
decoded. Try this query (which omits most fields for brevity).
```graphql
{
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

Transaction function and its inputs are decoded using the contract's
ABI. See the function name `execute` and inputs: `to` as `felt`,
`calldata` as a three element `felt` array.
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

Events are also decoded: see `Transfer` event and its argument `tokenId`
as struct `Uint256` with `low` and `high` hex, also converted into a
decimal number.
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

Let's get the raw undecoded block for comparison. This query may be
familiar to you as a common call to a blockchain API. Paste this into
the GraphQL query window -- you'll see block 100000 as we received it
from the API, with its transactions and events.
```graphql
{
  raw_block_by_pk(block_number: 100000) {
    raw
  }
}
```

The raw block has transaction inputs as `calldata` in a bulk array.
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

Event payload `data` is in bulk as well. 
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

## Query with http calls

While GraphQL web IDE is useful to explore blockchain data, can you
build analytics tools with these queries? Yes, cause you can send your
GraphQL queries to our http endpoint and consume query results by your
applications and front ends.

Your development process may start with you designing queries in the
GraphQL console, combining and refining them. Once you figured out how
to collect all the data you need, you can incorporate these query calls
into your DApp frontend.

Try this http call with queries for both the decoded and the raw block 100000.
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"{ block(where: {block_number: {_eq: 100000}}) { transactions { function entry_point_selector inputs { name type value } events { name transmitter_contract arguments { name type value decimal } } } } raw_block_by_pk(block_number: 100000) { raw }}"}'
```

## Query for your contract's events

You are probably interested not in whole blocks but in events emitted by
your own contract. Let's narrow down with this query for `Mint` events
of contract
`0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733`,
limited to one result for brevity.
```graphql
{
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

Request all `Mint` events with this http call.
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"query { event(where: {name: {_eq: \"Mint\"}, transmitter_contract: {_eq: \"0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733\"}}) { name arguments { name type value decimal } transaction_hash }}"}'
```

Obviously, you can add many conditions to the `where` clause selecting
your events. 

This query returns all `Mint` event whose `amount1` values are less than
10.
```graphql
query event_mint_argument_amount1_lte_10 {
  event(where: {arguments: {name: {_eq: "amount1"}, decimal: {_lt: "10"}}, name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}) {
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

This query accomplishes the same, but from the other end: it requests
all arguments satisfying the conditions `amount1` and `< 10` whose event
is `Mint`, and returns the results together with their event,
transaction and its block number. 
```graphql
query argument_amount1_lte_10_event_mint {
  argument(where: {decimal: {_lt: "10"}, name: {_eq: "amount1"}, event: {name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}}) {
    decimal
    name
    type
    value
    event {
      transaction_hash
      transaction {
        block_number
      }
    }
  }
}
```

Another example query requests all `Transfer` events with a given
destination address specified by the `to` event argument. Note these
events come from various contracts as seen in different
`transmitter_contract` fields, so you can narrow down further if needed.
```graphql
query event_transfer_to {
  event(where: {name: {_eq: "Transfer"}, arguments: {name: {_eq: "to"}, value: {_eq: "0x455eb02b7080a4ad5d2161cb94928acec81a4c9037b40bf106c4c797533c3e5"}}}) {
    name
    arguments {
      name
      type
      value
      decimal
    }
    transaction_hash
    transmitter_contract
  }
}
```

## Query for values in JSON payloads

Some data fields are atomic of type `felt` and are easily accessible by
queries, but some are members of structs and are stored in json values.

If the data you're interested in lies in a field inside json, you can
get to it by specifying a path to this field in your query.

Query for this transaction input `index_and_x` defined as a struct.
```graphql
{
  input(where: {name: {_eq: "index_and_x"}, transaction: {contract_address: {_eq: "0x579f32b8090d8d789d4b907a8935a75f6de583c9d60893586e24a83d173b6d5"}}}, limit: 1) {
    value
  }
}
```

Returns a value of `index_and_x` in a json payload with fields `index` 
and `values`.
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

This query digs into json by specifying the path to the second half of
the tuple stored in the `values` field.
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

For illustration, try this query to see our contract's ABI.
```graphql
{
  raw_abi_by_pk(contract_address: "0x579f32b8090d8d789d4b907a8935a75f6de583c9d60893586e24a83d173b6d5") {
    raw(path: "[0]")
  }
}
```

The type of `index_and_x` input is struct `IndexAndValues`. See its
definition in the ABI that shows how to get the second half of the tuple
`values(x : felt, y : felt)` by `path: "values[1]"`
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

Proxy contracts delegate transaction function calls to implementation
contracts. Transaction input and event data are encoded per
implementation contract's ABI. Implementation contracts change and so do
their ABIs. While interpreting proxy contract calls may be challenging,
the data can still be decoded, by finding the implementation contract
and its ABI.

This query requests three transactions sent to a proxy contract
`0x47495c732aa419dfecb43a2a78b4df926fddb251c7de0e88eab90d8a0399cd8`. You
see the first `DEPLOY` transaction setting the implementation contract
address to
`0x90aa7a9203bff78bfb24f0753c180a33d4bad95b1f4f510b36b00993815704`.
Let's add to the query a call to `raw_abi` to get ABIs for both proxy
and implementation contracts, for demonstration.
```graphql
{
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

See that the input `call_array` of type `CallArray` is defined in the
implementation, not the proxy contract's ABI.
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

Yet `call_array` is still decoded properly as `__execute__` function's 
input.
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

You know how to query for all your inputs and events, but how do you
interpret them? Let's say you want to derive a number from some of your
events, for example, to calculate Total Value Locked, which is a sum of
arguments `amount0` of all `Mint` events.

One approach is to query for all of the values of `amount0`.
```graphql
{
  argument(where: {name: {_eq: "amount0"}, event: {name: {_eq: "Mint"}, transmitter_contract: {_eq: "0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733"}}}, limit: 10) {
    type
    value
    name
    decimal
  }
}
```

See the values as `Uint256` struct and also conveniently converted into 
decimals.
```json
{
    "type": "Uint256",
    "value": {
      "low": "0x52b7d2dcc80cd2e4000000",
      "high": "0x0"
    },
    "name": "amount0",
    "decimal": "100000000000000000000000000"
}
```

You would consume this query's results by your software and sum up the
values of `amount0`, like some other indexers let you do. 

But since your data are already in a relational database's table, you
can run an **aggregation** query over the values, which sums them up and
returns the final result, without much effort.

That's why the values were converted into decimals when they were
persisted: GraphQL query `argument_aggregate` calls a SQL query with an
aggregation function `sum` over a numeric column. Database type `numeric
78` used for the `decimal` column is large enough to support Uint256 and
arithmetic operations with it.

This query aggregates decimal values of `amount0` arguments of all 
`Mint` events.
```graphql
{
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

Returns the total sum (TVL) as well as results of other aggregation 
functions: min, max, avg.
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

## Complex queries from database views

What if filters and aggregation queries still don't give you the desired
data? Then you can use the full power and flexibility of **SQL**: create
custom database views and functions and query them with GraphQL.

Let's say you want to calculate daily `Mint` volumes of your contract,
which requires summing over your events each day. The date can be
derived from `timestamp` column in the block containing the event. This
is not an easy thing to do by a GraphQL query yet trivial in a SQL
query. You can create a database **view** with the SQL `select`
statement returning the results desired. This view automatically becomes
available as a GraphQL query. Just like you can query database tables
`block`, `event` etc. with GraphQL, you can query the database views you
created.

This query calls a custom database view `daily_mint`.
```graphql
{
  daily_mint(limit: 3) {
    dt
    mint_amount0
  }
}
```

Returns sums of `amount0` arguments of `Mint` events per day:
```json
{
  "data": {
    "daily_mint": [
      {
        "dt": "2022-06-08",
        "mint_amount0": "1079024791522862986420035"
      },
      {
        "dt": "2022-06-07",
        "mint_amount0": "1406494987101656904988874"
      },
      {
        "dt": "2022-06-06",
        "mint_amount0": "1994302239023862329983776"
      }
    ]
  }
}
```

GraphQL query `daily_mint` was created from a database view with the
same name that sums over `Mint` event arguments grouped by day.
```sql
create view daily_mint(amount0, dt) as
select sum(a.decimal) as sum, (to_timestamp((b."timestamp")))::date AS dt
from argument a left join event e on a.event_id = e.id left join transaction t on e.transaction_hash = t.transaction_hash left join block b on t.block_number = b.block_number
where e.transmitter_contract = '0x4b05cce270364e2e4bf65bde3e9429b50c97ea3443b133442f838045f41e733' and e.name = 'Mint' and a.name = 'amount0'
group by dt order by dt desc;
```

Here's another example query that calculates total transactions per day.
```graphql
{
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

GraphQL query `daily_transactions` selects data from this database view:
```sql
create view daily_transactions (count, date) as
select count(t.transaction_hash), to_timestamp(b.timestamp)::date as dt from transaction as t
left join block b on t.block_number = b.block_number
group by dt order by dt desc;
```

These queries are available like all the others via http calls. Request
all daily transaction counts to date:
```bash
curl https://starknet-archive.hasura.app/v1/graphql --data-raw '{"query":"query {daily_transactions {count date}}"}'
```

Such statistical queries are useful for constructing charts and
dashboards. More on this later.

Try this GraphQL query selecting from `top_functions` database view.
```graphql
{
  top_functions(limit: 4) {
    count
    name
  }
}
```

Returns four functions called the most.
```json
{
  "data": {
    "top_functions": [
      {
        "count": "2388068",
        "name": "__execute__"
      },
      {
        "count": "1414978",
        "name": "execute"
      },
      {
        "count": "536120",
        "name": "constructor"
      },
      {
        "count": "322249",
        "name": "initialize"
      }
    ]
  }
}
```

The view was created with this SQL select statement:
```sql
create view top_functions (function, ct) as
select t.function, count(t.function) ct from transaction t group by t.function order by ct desc;
```

The above examples show that you can use SQL queries which
can be rather complex, to aggregate and calculate over any data you're
interested in.

In most cases no separate indexer process is needed to interpret your
data. If however you want to do something that SQL, even with custom
views and functions cannot, you can query for specific data with GraphQL
and consume the results by a your own software and deal with it.

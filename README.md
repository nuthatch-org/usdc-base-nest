# usdc-base-nest

A [nuthatch](https://github.com/nuthatch-org/nuthatch) nest: **Circle's native USDC on Base**.

Every transfer, approval, mint, burn and blacklisting of USDC on Base, decoded into SQL tables on your
own machine, with views for hourly volume and for Circle's issuance.

One binary, one config file, no Postgres, no Docker, no API key.

## What it indexes

**Chain:** `base`. **1 contract**, **17 tables**.

| alias | contract | address | from block |
|---|---|---|---|
| `usdc` | USDC (FiatToken proxy) | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 2,797,221 |

## Verified

Measured on 2026-11-17 with nuthatch 5.22.0 on an 18-core M5 Pro, against the two keyless public
endpoints in `nuthatch.toml`. One run, not a benchmark.

- `nuthatch dev --dir . --backfill 43200 --poll-interval 2s` caught up to the tip at block
  58,756,922 with **586,460 events in 2 m 56 s** (3,323 events a second). Base makes a block every
  200 ms now, so 43,200 blocks are about two and a half hours.
- Peak RSS over the run was 616 MB. Disk after it: 26 MB of sealed segments, 66 MB of hot store.
- Following at a 2 s poll, it read 12 to 17 blocks behind the tip (about three seconds) once a minute
  across 14 minutes and a restart.
- **Checked against `totalSupply`.** Between blocks 58,713,751 and 58,756,000 the `issuance` view
  holds 945 mints of 25,978,176.151430 USDC and 732 burns of 60,664,358.149583, a net of
  -34,686,181.998153. `totalSupply()` by `eth_call` on dRPC's public endpoint, which this nest does
  not use, fell by exactly 34,686,181.998153 between blocks 58,713,750 and 58,756,000.
- A restart after three minutes down was back at the tip in 2 s. The queries below answered in
  under half a second, the top-receivers one in a second.

## What it does not do

- **Full history is not attempted here.** At about 11 transfers a block and five blocks a second,
  the 56 million blocks since deployment are a job for your own archive node. Pass it with `--rpc`
  and drop `--backfill`.
- **No balances or supply.** A bounded backfill sees only movements, so there is no holder table;
  `issuance` gives the change in supply across a range, not the supply itself.
- **USDC's implementation has changed.** The current one was deployed at block 7,631,554; events
  before it may use an earlier ABI. `init` warns about this, and it does not affect a recent range.
- **Amounts are raw units.** USDC has 6 decimals; divide by 1e6.
- `base-pokt.nodies.app`, which `init` chose as the second endpoint, answered 403 and capped batches
  at ten during the run, so this nest uses publicnode in its place.
  publicnode refuses a `getLogs` with no address and archive requests without a token; neither
  stops a recent range.

## Run it

```sh
nuthatch init --from https://github.com/nuthatch-org/usdc-base-nest
cd usdc-base-nest
nuthatch dev --dir . --backfill 43200 --poll-interval 2s
```

## Views

- `transfers_hourly`: transfers and `value_raw` per hour. `hour` is the unix time the hour starts.
- `issuance`: one row per `Mint` or `Burn`, with `kind`, `actor` (the minter or burner) and
  `amount_raw`, negative for a burn, so its sum over a range is the change in `totalSupply`.

```sql
-- the five addresses that received the most USDC in the window
SELECT "to" AS receiver, count(*) AS transfers, sum(value_dec) / 1e6 AS usdc
FROM usdc__transfer GROUP BY "to" ORDER BY usdc DESC LIMIT 5;
```

```
 receiver                                   | transfers | usdc
--------------------------------------------+-----------+-----------------
 0x61040e143a77f165ba44543af4a079f2c809d14b | 2234      | 199476998.47385
 0x4a7af46bc8abc00048c6c7920a1488daf568366c | 3400      | 168708493.543169
 0x784e539f15ca2d19449a6311c5dab6cf37d8f1ba | 1773      | 133928549.422111
 0x2192bc3b4028acc1113f2cd9ac2cba70c36520db | 2726      | 100514789.92995
 0xd0acf293547457fb4fe9ee32ca93e89497626816 | 266       | 96476860.759351
```

```sql
-- Circle's mints and burns over a block range; the net is the change in totalSupply
SELECT kind, count(*) AS events, sum(amount_raw) / 1e6 AS usdc
FROM issuance WHERE block_number BETWEEN 58713751 AND 58756000 GROUP BY kind;
```

```
 kind | events | usdc
------+--------+------------------
 burn | 732    | -60664358.149583
 mint | 945    | 25978176.15143
```

```sql
-- hourly volume
SELECT hour, transfers, value_raw / 1e6 AS usdc FROM transfers_hourly ORDER BY hour DESC LIMIT 3;
```

`from` and `to` are reserved words in SQL, so quote them.

# Bitcoin Indexer

An experiment in creating a perfect Bitcoin Indexer, in Rust.

Query blocks using JsonRPC, dump them into Postgres in an append-only format,
suitable for querying, as much as event-sourcing-like handling. After reaching
chain-head, keep indexing in real-time and handle reorgs.

This started as an educational experiment, but is quite advanced already.

Goals:

* simplicity and small code:
    * easy to fork and customize
* versatile data model:
    * append-only log
    * support for event sourcing
    * support for efficient queries
    * correct by construction
    * always-coherent state view (i.e. atomic reorgs)
* top-notch performance, especially during initial indexing

Read [How to interact with a blockchain](https://dpc.pw/rust-bitcoin-indexer-how-to-interact-with-a-blockchain) for knowledge sharing, discoveries and design-decisions.

Status:

* The codebase is very simple, design quite clean and composable and performance really good.
* Some tests are there and everything seems quite robust, but not tested in production so far. 
* Indexing the blockchain works
* Indexing mempool works

# Support

If you like and/or use this project, you can pay for it by sending Bitcoin to
[33A9SwFHWEnwmFfgRfHu1GvSfCeDcABx93](bitcoin:33A9SwFHWEnwmFfgRfHu1GvSfCeDcABx93).

## Running

Install Rust with https://rustup.rs


### Bitcoind node

Setup Bitcoind full node, with a config similiar to this:

```
# [core]
# Run in the background as a daemon and accept commands.
daemon=0

# [rpc]
# Accept command line and JSON-RPC commands.
server=1
# Username for JSON-RPC connections
rpcuser=user
# Password for JSON-RPC connections
rpcpassword=password

# [wallet]
# Do not load the wallet and disable wallet RPC calls.
disablewallet=1
walletbroadcast=0
```

The only important part here is being able to access JSON-RPC interface.

### Postgresql

Setup Postgresql DB, with a db and user:pass that can access it. Example:

```
sudo su postgres
export PGPASSWORD=bitcoin-indexer
createuser bitcoin-indexer
createdb bitcoin-indexer bitcoin-indexer
```

### `.env` file

Setup `.env` file with Postgresql and Bitcoin Core connection data. Example:

```
DATABASE_URL=postgres://bitcoin-indexer:bitcoin-indexer@localhost/bitcoin-indexer
NODE_RPC_URL=http://someuser:somepassword@localhost:18443
```

#### Optimize DB performance for massive amount of inserts!

**This one is very important!!!**

Indexing from scratch will dump huge amounts of data into the DB.
If you don't want to wait for the initial indexing to complete for days or weeks,
you should carefully review this section.

On software level `pg.rs` already implements the following optimizations:

* inserts are made using multi-row value insert statements;
* multiple multi-row insert statements are batched into one transaction;
* initial sync starts with no indices and utxo set is cached in memory;
* once restarted, missing UTXOs are fetched from the db, but new ones
  are still being cached in memory;
* once restarted, only minimum indices are created (for UTXO fetching)
* all indices are created only after reach the chain-head

Tune your system for best performance too:

* [Consider tunning your PG instance][tune-psql] for such workloads.;
* [Make sure your DB is on a performant file-system][perf-fs]; generally COW filesystems perform poorly
  for databases, without providing any value; [on `btrfs` you can disable COW per directory][chattr];
  eg. `chattr -R +C /var/lib/postgresql/9.6/`; On other FSes: disable barriers, align to SSD; you can
  mount your fs in more risky-mode for initial sync, and revert back to safe settings
  aftewards.

[perf-fs]: https://www.slideshare.net/fuzzycz/postgresql-on-ext4-xfs-btrfs-and-zfs
[tune-psql]: https://stackoverflow.com/questions/12206600/how-to-speed-up-insertion-performance-in-postgresql
[chattr]: https://www.kossboss.com/btrfs-disabling-cow-on-a-file-or-directory-nodatacow/

Possibly ask an experienced db admin if anything more can be done. We're talking
about inserting around billion records into 3 tables each.

For reference -  on my system, I get around 30k txs indexed per second:

```
[2019-05-24T05:20:29Z INFO  bitcoin_indexer::db::pg] Block 194369H fully indexed and commited; 99block/s; 30231tx/s
```

which leads to around 5 hours initial blockchain indexing time (current block height is around 577k)...
and then just 4 additional 4 hours to build indices.

### Run

Now everything should be ready. Compile and run with:

```
cargo run --release --bin bitcoin-indexer
```

After the initial full sync, you can also start mempool indexer:

```
cargo run --release --bin mempool-indexer
```


in a directory containing the `.env` file.

#### More options

You can use `--wipe-whole-db` to wipe the db. (to be removed in the future)

For logging set env. var. `RUST_LOG` to `rust_bitcoin=info` or refer to https://docs.rs/env_logger/0.6.0/env_logger/.


### Some useful stuff that can be done already

Check current balance of an address:

```
bitcoin-indexer=> select * from address_balances where address = '14zV5ZCqYmgyCzoVEhRVsP7SpUDVsCBz5g';                                                                                                                                          
              address               |   value
------------------------------------+------------
 14zV5ZCqYmgyCzoVEhRVsP7SpUDVsCBz5g | 6138945213
```

Check balances at a given height:

```
bitcoin-indexer=> select * from address_balances_at_height WHERE address IN ('14zV5ZCqYmgyCzoVEhRVsP7SpUDVsCBz5g', '344tcgkKA97LpgzGtAprtqnNRDfo4VQQWT') AND height = 559834;
              address               | height |   value   
------------------------------------+--------+-----------
 14zV5ZCqYmgyCzoVEhRVsP7SpUDVsCBz5g | 559834 | 162209091
 344tcgkKA97LpgzGtAprtqnNRDfo4VQQWT | 559834 |         0
```

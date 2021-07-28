# How to build an Ethereum mining pool

Mining pools are major power players in the Ethereum ecosystem. With miner-extractable value ("MEV") growing exponentially, the passing of EIP-1559 and the upcoming merge, they have become louder and increasingly important actors in the ecosystem.

For the uninitiated: mining pools are software providers who enable many mining machines to pool together their mining power and share rewards. Mining pools are essential in PoW mining on two levels: first, because earnings for individual miners are highly volatile, and second, because setting up the software infrastructure around mining is increasingly complex. By pooling resources, individual miners can lower variance and have a more predictable business.

But with this power comes great responsibility, and mining pools hold a lot of power. This is because mining pools ultimately decide which blocks get worked on by their miners and which transactions are included in those blocks. Mining pools decide on what MEV gets extracted and who gets to extract it, they vote on the gas limit, and they take part in major political battles. That’s why it’s essential to Ethereum culture that the barrier to entry for mining pools be as low as possible, to maximize decentralization.

When MiningDAO set out to build our own independent pool, we were surprised to find that it was incredibly challenging! There’s very little open and publicly shared info on how to run a competitive mining pool, and a lot of the open-source software is out of date.  So we figured: let’s fix that by releasing an open-source, step-by-step guide.

Building a pool consists of two parts: (1) setting up a full node client with good peer-to-peer networking and fast processing speed, and (2) connecting the full node to pool software that manages hashrate and distributes workload across all the miners.
Here, we’ll cover both.

**This guide comes from our first-hand experience building the [MiningDAO.io](https://miningdao.io/) pool, and outlines how we brought our uncle rates from 10%-14% down to approximately 4%-5%, on par or better than some top-10 pools.**

## 1) Set up Ethereum full node client

Running a mining pool requires running an Ethereum full node client. This client will be responsible for receiving new blocks and pending transactions, as well as producing its own blocks and broadcasting them to other nodes. This section covers how to properly set up one's full node client.

### 1.1) Server hardware requirements

Running a fully synced node requires fairly good hardware. We recommend at least 32GB RAM, and at least 2TB SSD storage (syncing the chain with HDD will take forever).

Bandwidth is important as well. It is preferable to co-locate as close as possible to other nodes, to receive new blocks as soon as possible. We recommend cloud-hosted dedicated machines on services popular with other pools: [OVH](https://www.ovhcloud.com/en/bare-metal/) and [Hetzner](https://www.hetzner.com/dedicated-rootserver) in Europe, Alibaba and AWS in Asia.

### 1.2) Geth or OpenEthereum? Geth!

The next decision to make is which Ethereum client to use. The most popular and well-tested choices are Geth and OpenEthereum (née Parity).
Geth leads on protocol development and is always up-to-date.

For comparison, we did some small-scale experimentation with Parity-2.7.2 (latest stable branch before the OpenEthereum refactoring) and OpenEthereum, but both had poor results with block import times and block production times, leading to unacceptably high uncle rates.
We welcome anyone to perform a more thorough A/B test and [reach out to us](mailto:contact@miningdao.io) with more data, but at the current stage we simply recommend Geth.

Here are the flags we use:
```bash
geth --datadir=/ssd/gethdata --syncmode=fast --cache=21000
 --maxpeers=250 --txpool.globalslots=1000
 --http --http.api=eth
 --miner.etherbase='0xADDRESS' --mine --miner.threads=0
 --miner.extradata='MiningDAO'
 --miner.notify='http://127.0.0.1:8107' &>> ~/geth-log.txt
```
Here `--cache=21000` means to allocate 21GB for in-RAM state storage (the most Geth can handle), and the remaining flags will be explained below.

More importantly, the modifications to the Geth code we will describe below can be found [here](https://github.com/bogatyy/go-ethereum/) as a repo to download, or [here](https://gist.github.com/bogatyy/de048d747c73ca4069c142899c0071fb) as a patch to apply.

### 1.3) Minimize frequency of empty blocks

Two things destroy value for miners: mining uncle blocks, and mining empty blocks.
In fact, the two are almost equally bad: uncle block rewards are `1.75 ETH`, and empty block rewards are `2 ETH`, with no transaction fee surplus in both cases. For comparison, full blocks with transaction fees typically bring `3-4 ETH` in total rewards, and sometimes a lot more. So why do mining pools sometimes [produce](https://etherscan.io/block/12737189) empty blocks, and how can one minimize their frequency?

When another pool mines a fresh block (say at height `N`), any other blocks at height `N` are likely to become uncles. So whenever a new block is found, Geth instantly switches the miners' jobs to mine an empty block at height `N+1`. This empty block does not have transaction fees, but that is still better than mining blocks destined to become uncles.
Subsequently, geth constructs a "real" block at height `N+1`, and switches the miners' jobs once again. Constructing such a "real" block takes time (0.1-0.3 seconds), hence the two-step process. But in that interim 0.1-0.3 seconds-long period miners are working on an empty block.

It might be tempting to collect all the pending transactions to maximize fees, but getting greedy with `--txpool.globalslots` substantially increases the amount of processing Geth has to do to construct a "real" block (up to 1 second and more). We recommend values no larger than `1000` or `2000`.

For more details on this, check out https://github.com/ethereum/go-ethereum/issues/21899

### 1.4) Minimize frequency of uncle blocks

With empty blocks out of the way, we can get started on the hard part.
To minimize uncle rates, two things are key:
1. when other pools produce new blocks, learn about it as quickly as possible
1. when your pool produces a new block, propagate it as widely as possible (so others start mining on top of it)

The first step to good p2p is, as explained earlier, running your full node on a cloud server with good bandwidth next to other nodes.

Second, good bandwidth allows the node to handle more direct peers, thus reducing the number of p2p hops necessary to receive new information. The Geth flag for the number of peers is `--maxpeers`.

Below we will explain a few more nuanced and powerful tricks to maximize block import speed and block propagation speed. 

#### 1.4.1) Use bloXroute

[bloXroute](https://bloxroute.com/) is a service dedicated to improving connectivity between miners and lowering their uncle rates. Most pools are connected to bloXroute, and even major established pools [report](https://medium.com/bloxroute/the-secret-weapon-f2pool-used-to-tackle-its-uncle-rate-1ecb6fe47ef8) significant improvements from using bloXroute. [Measurements](https://medium.com/keeperdao/a-performance-benchmark-on-mempool-services-9e68bf070952) performed by KeeperDAO further confirm the massive advantage bloXroute holds over comparable services.

Our experiments showed significant improvements as well. On a freshly-synced node with default peer settings, approximately 90% of all new blocks come from bloXroute first (and only 10% from all other peers). Even after our node has been fine-tuned to connect to top peers, still 40%-60% of new blocks come from bloXroute first.

After following the bloXroute setup tutorial, don't forget to [add the bloXroute node into the "trusted peers"](https://docs.bloxroute.com/gateway/gateway-installation/adding-the-gateway-as-a-trusted-peer) set for your Geth, we will need that later. [Trusted](https://github.com/ethereum/go-ethereum/blob/4fcc93d9229e99b14d3d1500f05ada6ba1f789bb/p2p/server.go#L113) peers are pre-set nodes that Geth will always connect to, irrespective of the random peer initialization process. Trusted peers also do not count against the connections limit. Adding the bloXroute gateway to trusted peers ensures Geth will not accidentally drop that connection.

**We further recommend connecting to Taichi Network.** Taichi is a block propagation network developed by Sparkpool. Connecting to it can be done by adding the Taichi [endpoints](https://taichi.network/) to the same trusted peers file.

#### 1.4.2) Propagate your blocks aggressively

Whenever Geth successfully mines a new block, it sends the block to propagate across the network. By default, Geth only propagates it to a random subset of size `sqrt(n_peers)`, who then propagate the block to some of their peers, and so on.
This mechanism is not ideal even if all peers were equally useful, but it is especially detrimental when some peers are more powerful than others and these peers end up not being included in the subset.

In particular, the first thing to do upon mining a new block is to send it to bloXroute, so that it will be forwarded to all the other participating mining pools. If the bloXroute gateway doesn't end up in that random `sqrt(n_peers)` subset, your chances of getting uncled go way up!

Next, you'd want to send the block to the highest-quality peers, and then to all the remaining peers.

We have open-sourced the [following Geth patch](https://github.com/bogatyy/go-ethereum/commit/13dad64b82a2b488a10d345da511514219c52732) and recommend applying it to your client. It propagates all newly mined blocks to all trusted nodes (including bloXroute), and then to all remaining peers.

#### 1.4.3) Cultivate the most well-connected peers

Vanilla Geth is designed for maximum decentralization and a flat network structure. This choice works well for hobbyists and supports a robust ecosystem of [thousands](https://www.ethernodes.org/) of nodes. However, as we've already seen in the previous section, these defaults do not work as well for nodes that perform critical responsibilities and have significant dollar costs associated with failures.

In reality, not all peers are equally useful. Some have slow connections and will neither supply new blocks nor help your blocks propagate. Others, especially the nodes of other mining pools, will produce a constant stream of new block data.
Following advice from Sparkpool, we [tweaked our Geth](https://github.com/bogatyy/go-ethereum/commit/3506b916afb01cf9ed0622c62f38daa094ae2800) to log which peers were the first to send us a new block. Collecting that data for several months allowed us to figure out the best peers to always keep connected to (via the "static"/"trusted" node settings in Geth). [Here](https://gist.github.com/bogatyy/d4cafa354e0c2e04809ce527f616a476) is a Python script we used to process that data and convert it into a `trusted_nodes.json` list that Geth can ingest.

Because MiningDAO is present in each geozone (North America, Europe, Asia), we have data-mined the lists of top peers for each geozone. Unfortunately, we cannot share the node IPs publicly to avoid the risk that these nodes will be DDoSed. Can probably share privately on serious inquiries with good justification. Also happy to share our own nodes in each geo for other pools to connect!

## 2) Set up mining pool software

Once the full node is set up properly, the next step is to set up the mining pool software itself.
This software will be responsible for handling connections from all the individual miner rigs, keeping track of worker shares, and managing payouts.

### 2.1) Pick pool software

We briefly analyzed the following 4 options: [miningcore](https://github.com/coinfoundry/miningcore), [open-ethereum-pool](https://github.com/sammy007/open-ethereum-pool), [NOMP (node open mining portal)](https://github.com/zone117x/node-open-mining-portal), and [MPOS (mining portal open source)](https://github.com/MPOS/php-mpos).
We later learned about [Flexpool Solo](https://github.com/flexpool/solo) but did not experiment with it.

We had a great experience with Miningcore for two reasons. First, it keeps all past data on disk in an SQL database, unlike open-ethereum-pool, which keeps data only in RAM via Redis. Disk storage offers stability against reboots and ability to analyze historical data. Second, we enjoyed the highly readable, object-oriented code of Miningcore.

Ultimately, MiningDAO ended up implementing our own pool software, written in Go for speed and modeled after Miningcore. We expect to open-source our implementation soon, but in the meantime we recommend using Miningcore.

### 2.2) Fix pool software latency

A major hiccup we encountered while using Miningcore is the way it processes work updates.
By default, Miningcore pings the full node RPC every 0.5 seconds to see if the latest job has changed. This setup may work for other cryptocurrencies with longer block times (it would be a negligible problem for Bitcoin with 10-minute blocks), but for Ethereum such a setup leads to unacceptably high uncle rates.

For context, there is an easy way to calculate the increase in uncle rates from any processing delay. Block times are [Poisson-distributed](https://en.wikipedia.org/wiki/Poisson_distribution), which means that no matter how long it has been since the last mined block, the probability of finding the next block in the next second (or millisecond or whatever) is always the same. For example, Ethereum targets 13-second block times, which means the probability of a block being mined in the next second is always `1 sec / 13 sec ~= 7.7%`. So if your mining pool has a `0.1 sec` delay anywhere in the pipeline for any reason, it will have `0.1 sec / 13 sec ~= 0.77%` extra uncle blocks as a result of that delay. The uncle blocks will come from that `0.1 sec` period of time that your miners are working on an outdated job.

Back to Miningcore. Using the formula above, a 0.5 second delay in updating the miners' job will lead to `0.5 sec / 13 sec ~= 4%` extra uncle blocks (absolute, not relative percentage). Naturally, such a high rate of unforced errors is unacceptable. We have experimented extensively with lowering the update frequency from 0.5 seconds down to 50 milliseconds and below, but found that setting rather unreliable: the updates were still significantly delayed.

A better solution is to make use of Geth's `notifyWork` feature, so that Geth proactively sends job updates to the mining pool software as soon as they appear. We patched Miningcore to support this option, and [released](https://github.com/Mining-DAO/miningcore) the modification. After transitioning to `notifyWork`, we found the communication delays between Geth and Miningcore to become practically negligible, and thus our uncle rates significantly improved.

## Conclusion

Hopefully this post proves useful and leads to more people running Ethereum mining pools or solo-mining, helping keep Ethereum open and decentralized. To summarize, we started with vanilla Geth on default settings and vanilla Miningcore implementation. This default setup produced uncles at a rate of approximately 10%-14%. Progressively applying the modifications outlined here brought our uncle rate [down to 4%-5%](https://etherscan.io/address/0xbbbbbbbb49459e69878219f906e73aa325ff2f0c#uncle), comparable or better than some existing top-10 pools (the Etherscan uncle rate is a bit higher because we sometimes experiment in prod).

Our Geth modifications can be found [here](https://github.com/bogatyy/go-ethereum) as a repo and [here](https://gist.github.com/bogatyy/de048d747c73ca4069c142899c0071fb) as a patch. Our Miningcore modifications can be found [here](https://github.com/Mining-DAO/miningcore), and a corresponding pool config file can be found [here](https://gist.github.com/bogatyy/420e789b3c05e3983ceb9a3966cb305c).

If you have further ideas on how to improve this setup, please send us a [pull request](https://github.com/Mining-DAO/docs) or an [email](mailto:contact@miningdao.io)!

*Miningcore speedup developed by [Alexander Melnikov](https://twitter.com/themelnikov). Many thanks for the suggestions and ideas that came from [Alex Obadia](https://twitter.com/ObadiaAlex) ([Flashbots](https://github.com/flashbots/pm)), [Eyal Markovich](https://twitter.com/eyalmarkov) and [Shen Chen](https://twitter.com/sunlearnstorock) ([bloXroute](https://twitter.com/bloXrouteLabs)), [Xin Xu](https://twitter.com/xuxin5108) and Dr. Yang Ze ([Sparkpool](https://twitter.com/sparkpool_eth)), Chris ([Flexpool](https://twitter.com/flexpool_io)) and [Haseeb Qureshi]() ([Dragonfly Capital](https://twitter.com/dragonfly_cap)).*
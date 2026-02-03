The distinctions between blockchain data, mempool content, and on-disk storage must be addressed carefully. Allegations that the blockchain 
remains identical across nodes while the mempool and other data differ, alongside claims that disk-stored elements such as blocks, indexes, 
and chainstate are fully identical to those in Bitcoin Core, cannot all hold true simultaneously. These positions conflict with each other 
and overlook additional customizations like mempool size limits. In a decentralized, distributed network like Bitcoin, modal and 
chronological differences arise naturally because nodes receive, validate, and process different transactions at different times, experience 
varying network demands, and may have miners operating in parallel. Assertions that mempool limits can be applied without affecting disk 
identity, or that disk contents remain identical node-to-node or between different clients like Core and Knots, do not withstand scrutiny 
even at the individual node level, let alone across nodes with divergent configurations.

On-disk storage includes references tied to in-memory states, where variations stem from differing loads, connected peers, validation 
outcomes, memory usage, and the chronology of block processing. Given the same block height of already-mined blocks, respected chronological 
transfer order, and proper synchronization from memory to disk via flush operations, the blockchain file or its granular structures should 
present exactly the same content and checksum for the flushed chain up to that head. This holds only for the flushed blockchain state at 
matching heads and does not extend to inter-node comparisons in other cases. Mempool.dat files, whether persisted via persistmempool or saved 
explicitly, remain linked to in-memory content before flushing to disk during expiry or save events. Divergences between nodes or different 
clients emerge for distinctive reasons, including those separating Knots from Core, as well as Core versions before and after 0.30. Knots 
provides configuration options unavailable in Core, notably datacarriersize, maxscriptsize, datacarriercost, and datacarrierfullcount, among 
others.

The mempool.dat on disk reflects distinctive impacts from datacarriersize independently of broader distributed-network variances in memory 
distribution or processing. With persistmempool enabled, data exceeding the default 300 MB limit or any configured threshold is written to 
disk. Customizing maxmempool imposes memory cost limits upward or downward, yet datacarriersize acts as a prior filter, ensuring that RAM and 
.dat contents differ regardless of the maxmempool setting. The chainstate directory remains largely unaltered in core structure, though 
Ordinals and BRC-20 affect UTXOs through separate mechanisms untouched by datacarriersize. However, datacarriersize influences I/O frequency 
and disk structure. The following code illustrates this behavior:

```
            if (!CheckDiskSpace(m_chainman.m_options.datadir, 48 * 2 * 2 * CoinsTip().GetCacheSize())) {
                return FatalError(m_chainman.GetNotifications(), state, _("Disk space is too low!"));
            }
            // Flush the chainstate (which may refer to block index entries).
            const auto empty_cache{(mode == FlushStateMode::ALWAYS) || fCacheLarge || fCacheCritical};
            if (empty_cache ? !CoinsTip().Flush() : !CoinsTip().Sync()) {
                return FatalError(m_chainman.GetNotifications(), state, _("Failed to write to coin database."));
            }
            full_flush_completed = true;
            TRACEPOINT(utxocache, flush,
                    int64_t{Ticks<std::chrono::microseconds>(NodeClock::now() - nNow)},
                    (uint32_t)mode,
                    (uint64_t)coins_count,
                    (uint64_t)coins_mem_usage,
                    (bool)fFlushForPrune);
```

This logic notifies errors, logs them, and prevents state updates when necessary, resulting in reduced I/O compared to Knots defaults.

Block and transaction indexes stored in directories such as datadir/blocks/index or datadir/indexes/txindex map txid to block hash plus 
position and therefore diverge based on which transactions are accepted. Differences occur upward or downward between nodes, with Core versus 
Knots exhibiting particularities because datacarriersize selects transactions by different criteria, especially when Knots deviates from the 
30-byte uncapped limit or prior Core defaults of roughly 80 bytes. The primary impact arises when a Knots node mines a block: it filters out 
certain unspendable transactions, producing a smaller block than one including them. This does not disrupt the node itself but permanently 
alters the blockchain for every block mined solo or via pools using nodes that enforce datacarriersize filtering.

Parameters like maxscriptsize, datacarriercost, and datacarrierfullcount further restrict script sizes, for instance limiting Taproot witness 
data and thereby filtering inscriptions in witness fields associated with BRC-20 and Ordinals. Datacarriercost penalizes spam transactions in 
the mempool by adjusting fees relative to size, while datacarrierfullcount tallies full data outputs or scripts against overall limits, 
blocking multiples. The greatest effects manifest in memory and disk economies, as well as reduced swapping. Originally the emphasis was 
placed on memory, network, and processing demands rather than solely disk, since consensus-critical elements remain unaffected while 
processing-linked files and memory states introduce variances. Disk represents the cheapest resource, yet low-resource nodes encounter RAM 
and CPU bottlenecks first. Storage metrics matter less than I/O throughput, particularly for slow large-capacity media like SD cards versus 
high-performance options.

Knots defaults datacarriersize to 42 bytes while allowing configuration down to zero, yielding immediate impacts on acceptance and storage 
even at default settings. Claims that nothing changes are imprecise, as the parameter is not as radical as zero but still filters 
meaningfully. The broader family of options including maxscriptsize, datacarriercost, and datacarrierfullcount effectively disrupts Ordinals, 
Runes, and BRC-20 flows, particularly for solo miners and Knots-centric pools, though this may carry drawbacks for protocols like Rhunes.

Additional on-disk elements include RPC authentication cookies managed via wallet restrictions. The following code shows the relevant changes:

```
bool GenerateAuthCookie(std::string* cookie_out, const std::pair<std::optional<fs::perms>, bool>& cookie_perms);
/** Read the RPC authentication cookie from disk */
bool GetAuthCookie(std::string *cookie_out);
/** Delete RPC authentication cookie from disk */
...
    enum Mode { EXECUTE, GET_HELP, GET_ARGS } mode = EXECUTE;
    std::string URI;
    std::string authUser;
    std::string m_wallet_restriction{"-"};
```

Fuzz testing reduces unnecessary disk I/O as seen in these lines:

```
    "dumpwallet", // avoid writing to disk
    "importwallet", // avoid reading from disk
    "savefeeestimates",      // disabled as a precautionary measure: may take a file path argument in the future
```

Package validation safeguards against premature quits and backend issues:


```
        if (!std::all_of(child->vin.cbegin(), child->vin.cend(), package_or_confirmed)) {
            package_state_quit_early.Invalid(PackageValidationResult::PCKG_POLICY, "package-not-child-with-unconfirmed-parents");
            return PackageMempoolAcceptResult(package_state_quit_early, {});
        }
        // Protect against bugs where we pull more inputs from disk that miss being added to
        // coins_to_uncache. The backend will be connected again when needed in PreChecks.
        m_view.SetBackend(m_dummy);
```


--
Best,
Patri Freimann <patrifreimann@gmx.at>












TESTE.txt:

FUP from An update on OP_RETURN, Bitcoin Core v30, and the Core-Knots war 
https://oakresearch.io/en/analyses/fundamentals/update-op-return-bitcoin-core-v30-core-knots-war

This techn tests show product differences between Bitcoin Knots and Bitcoin Core.

We download Bitcoin Core v30 which  was moved to this subdirectory /insecure-wallet-deletion/.
You can see that it accepts the parameter -1 today. This is why it is important to check the
source code or test the behavior empirically. [1.png]

Reproducing the tests on v30 the behavior is different from v27. Testing with -1 does
not filter. It does not become unlimited or infinite either. Looking at the source code it
applies the default maxcap. In v30 this was bumped to 100000. [2.png]

About the other parameters earlier explained but here are the results. This confirms
any in regards to doubt "what changes". It changes this that you can see in the print.
None of this works or exists in Core but everything is functional in Knots. [3.png]

In other words in my node I have OP_RETURN allowed but OP_RETURN limited to 83 bytes.
This was the default before the bump of the  disagreement in v30. Only 1 "full" OP_RETURN
per transaction and OP_RETURN penalized in the mempool via policy cost. Clearly this
is only  possible in Knots. Evidence is in the print. [3.png]

We could check the sources. But if You missed the comments and explanations about changes
and additions. I took the options that were highlighted in the source code and did a
simple grep. It was enough for distinction in the compiled product.[4.png]

The reference code changes both potential behavior, exposes additional configurations
and allow economics adjustment on a per-node base, punishing large OP_RETURN with
additional costs, without filtering em out or limiting application of the field to
avoid abuses while still allowing functional extensions made by clients writing into
the blockchain. [S3.png, S4.png, S5.png]

This answers in detail both what changes and what only exists in Knots. All that exist
in both have a behavior change from v29 to Knots. And from v30 they may change again. This
is seen by the deprecation of datacarriersize and behavior change plus bumping up
the maximum cap. [S1.png, S2.png]

This was in the sources even commented. So I will highlight the behavior changes.
See that "is set but are marked as deprecated" followed by  "They will be removed".
That is why its important noticing the deprecated warning. But you never responded.
So whatever the current behavior is when receiving 0 or -1 argument it seems that it will
no longer be exposed to the user and will remain the hardcoded limit max cap. [S1.png]

Finally see that when I applied those specific filters to allow some Ord and Runes.
Only possible in Knots. It took time for transactions that matched to appear but then 3
and 8 appeared respectively. See it closely. [5.png]

And to remind that the reason for the change in Knots was that before v30 the behavior
was different. Here it is on my old node. It was running for 2 years so I am not
going to update. The very same testes, different behavior from v27 to v30. Knots was
forked inbetween. [6.png, 7.png, 8.png]

For Knots supporters, Core v30's decision to allow up to 100,000 bytes per OP_RETURN
is a symbolic abandonment of this founding idea. As of this writting, Knots accounts
for ~23% of the total nodes while Bitcoin Core the remaining ~76% with no third 
competiting implementation in the free market of Bitcoin daemons.
--
Prenomnom, P. Freimann, and contributors.

# TSDB Internal Architecture

This document seeks to provide the reader with a comprehensive overview of the core concepts and processes that govern the TSDB.

The information presented here is correct as of Prometheus v2.19.0, but will likely change in future releases. Please bear this in mind while reading.


## Data Ingestion

Each block of scrape configuration creates a scrape pool 

Each scrape pool creates and operates one scrape loop per target discovered. 

Each target exposes  

Prometheus operates a 'pull' based system of data ingestion - that means that Prometheus requests data from targets rather than hav

How does Prometheus obtain its data?

Prometheus operates one scrape loop per target. Each target exposes the metrics data in the metrics exposition format.

Once the raw metrics page has been obtained from the target, it is converted into native data structures by the text parser.

The raw samples are then added to the scrape loop's appender

Once the raw metrics page has been [scraped](https://github.com/ianbillett/prometheus/blob/190addffd824be62b590360b894b183a8c074f6f/scrape/scrape.go#L950) from the target, it is [parsed](https://github.com/ianbillett/prometheus/blob/190addffd824be62b590360b894b183a8c074f6f/scrape/scrape.go#L1080) into native Golang types by the [textparse.Parser](https://github.com/ianbillett/prometheus/blob/e825282dd1b7a62999ed50d5c8039c5fdda89bea/pkg/textparse/interface.go#L25).

These raw samples are [added](https://github.com/ianbillett/prometheus/blob/190addffd824be62b590360b894b183a8c074f6f/scrape/scrape.go#L1171) to the scrape loop's [storage.Appender](https://github.com/ianbillett/prometheus/blob/b788986717e1597452ca25e5219510bb787165c7/storage/interface.go#L134) which buffers that scrape's samples before being [commited](https://github.com/ianbillett/prometheus/blob/190addffd824be62b590360b894b183a8c074f6f/scrape/scrape.go#L1091) to the underlying storage engine.      
Scrape loops use a [`fanout`](https://github.com/ianbillett/prometheus/blob/b788986717e1597452ca25e5219510bb787165c7/storage/fanout.go#L34) storage type that can write samples to multiple storage backends. In the [default case](https://github.com/ianbillett/prometheus/blob/d17d88935c8d0ba3044fb1f73ee32b07d2d49f11/cmd/prometheus/main.go#L352), this is local storage as the primary, with any number of configured remote storage backends as the secondaries.  

This document will focus exclusively on the local storage.

A lot of things happen when the [`tsdb.DB`](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L538) is [opened](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L538). 

The local `storage.Appender` used by the scrape loops resolves to the [`tsdb.headAppender`](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1074).

When the scrape loop calls [`headAppender.Commit`](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1184) after all the target's samples have been parsed, a number of things happen:
* The samples and series are [written](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1185) to the WAL - more on that later.
*  Each of the samples batched by the appender are [appended](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1201) to the corresponding [`tsdb.memSeries`](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1920) object.

[`tsdb.memSeries`](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1920) is a data structure that holds all of the in-memory data for a given time series.

When [`memSeries.append`](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L2098) is called with the timestamp & value, these values are appended to the current head chunk.

Chunk: per-time series encoded sequence of samples.

The `memSeries.headChunk` is the in-memory chunk that is currently open for appends.

Samples are encoded when they are appended to a chunk. By [default](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1975) chunks are [XOR](https://github.com/ianbillett/prometheus/blob/f42ed03dc5be7290a913fe427cdf96f6d1bee792/tsdb/chunkenc/xor.go#L57) encoded. This is an encoding scheme that leverages the regularity of scraping intervals and linearly increasing scrape values to compress the raw 16 bytes of a sample (8 byte integer timestamp & 8 bytes float64 value) to 1.2 bytes on average. According to the Gorilla white paper, 120 samples per chunk offers the optimal compression ratio for this type of encoding. 
 
After [enough](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L2124) samples have been written to the head chunk, it is memory-mapped to a file and a new head chunk is opened. The memory-mapping of closed head chunks was a major optimisation introduced in Prometheus v2.19.0 that reduced memory usage by up to 40%. You can read more about this [here](https://grafana.com/blog/2020/06/10/new-in-prometheus-v2.19.0-memory-mapping-of-full-chunks-of-the-head-block-reduces-memory-usage-by-as-much-as-40/). To summarise, backing these in-memory objects with disk allows the operating system to freely deallocate objects that are not often used, such as the chunks that we have finished writing to. 

At this point, we have seen how Prometheus buffers the stream of incoming samples in memory, but how is this data persisted onto disk?

The overarching data structure governing the TSDB is [`tsdb.Head`](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L50), which is commonly referred to as the 'head block'. This object stores all of the `tsdb.memSeries` objects that our scrape loops have been writing to, and is [instantiated](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L608) when the TSDB is [opened](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L538.).

At [least](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L674) every minute, [`db.Compact`](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L733) is called. If the `tsdb.Head` is deemed to be [compactable](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1420), approximately containing two hours worth of data - the head is [compacted](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L781)

[`tsdb.Compactor`](https://github.com/ianbillett/prometheus/blob/823b218e1b2e992338188314c9307437fb075c93/tsdb/compact.go#L56) is an object that compacts one or more blocks into an on-disk block containing all of the samples contained in the input blocks. In the case of the 'head block', `tsdb.Compactor` [compacts](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L785) all of the in-memory chunks onto disk. 

When this data has been persisted to disk, the head is [`truncated`](https://github.com/ianbillett/prometheus/blob/e65e2e0dac2ab726dc8db600a3523ea37dcebf4d/tsdb/db.go#L802) which essentially removes all existing in-memory samples from the head, and prepares it to receive the ongoing stream of samples from the scrape loops.

After being written to disk, a block looks like the following:

```
01EC0Y2MH2ZE557VX019T52X2B
├── chunks
│   ├── 000001
│   └── 000002
├── index
├── meta.json
└── tombstones
```

The [name](https://github.com/ianbillett/prometheus/blob/823b218e1b2e992338188314c9307437fb075c93/tsdb/compact.go#L527) of the block directory is the string representation of ULID hash of the [timestamp](https://github.com/ianbillett/prometheus/blob/823b218e1b2e992338188314c9307437fb075c93/tsdb/compact.go#L470) that the block was created.

The `chunks` sub-directory contains the `chunk segment` files. These are **not** per-series (as was the case in Prometheus v1) because with high levels of series churn (the rate that new time series are created) your operating system can very easily run out of file descriptors. Instead, the  chunks contained within the block are written in sequential files in the [Chunks Disk Format](tsdb/docs/format/chunks.md) that roll-over at 512MB.   

The `index` file contains all of the information necessary to map a given label set (time series identifier) to a specific locations in the chunk segment files where the time series data for that label set resides. The [Index Disk Format](tsdb/docs/format/index.md) is heavily optimised to minimise disk usage,  

`meta.json` looks like 

```json
{
	"ulid": "01EDAZWVD3QQ6K0ZYEGM7G2S82",
	"minTime": 1594837606481,
	"maxTime": 1594857600000,
	"stats": {
		"numSamples": 39284720,
		"numSeries": 60018,
		"numChunks": 295964
	},
	"compaction": {
		"level": 2,
		"sources": [
			"01EDA5YVAEP1CNQ3MWNVVKCE8P",
			"01EDAB9J6D4CP3KQR12D9T2H8N",
			"01EDAJ59EBS0CQYV4HT9RCWMXE"
		],
		"parents": [
			{
				"ulid": "01EDA5YVAEP1CNQ3MWNVVKCE8P",
				"minTime": 1594837606481,
				"maxTime": 1594843200000
			},
			{
				"ulid": "01EDAB9J6D4CP3KQR12D9T2H8N",
				"minTime": 1594843200000,
				"maxTime": 1594850400000
			},
			{
				"ulid": "01EDAJ59EBS0CQYV4HT9RCWMXE",
				"minTime": 1594850400000,
				"maxTime": 1594857600000
			}
		]
	},
	"version": 1
}
```

It contains human-readable information about the data contained within the block: the time-range that it covers, and the blocks that were compacted to create this current block.

Finally, Tombstones represent the way that prometheus supports for the deletion of data. When data is deleted via the [Delete Series API](https://prometheus.io/docs/prometheus/latest/querying/api/#delete-series), the time series data referenced by label set and time range in the API request are not deleted immediately. Instead, a Tombstone entry is created in the corresponding block in the [Tombstone Disk Format](tsdb/docs/format/tombstones.md.). When the block containing the tombstones is compacted, the compactor does not include the data referenced by the tombstones. This is how prometheus supports the deletion of data. 

Why doesn't Prometheus delete data from the block? Given the highly optimised format of the on-disk blocks, the deletion of any time series would require an expensive operation rewrite the entireity of the index and corresponding chunk segment files.  


At this point, all of the samples ingested from our targets are being held in memory (or memory-mapped to disk, which is kind of the same thing). What happens if the process crashes while we have all of this data in memory? 

Prometheus protects us against this eventuality with the write ahead log (WAL). The WAL is an on-disk persistent log of all raw samples ingested from 

Let's revisit the WAL.

You will recall that when the when the scrape loop calls `headAppender.Commit` on the samples ingested in that scrape, the first thing that happens is that they are [written](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1185) to the [WAL](https://github.com/ianbillett/prometheus/blob/ea013343cacf8e6c27d5027fe2856aa51d8f57ad/tsdb/head.go#L1169).

Every time that a scrape loop's data is committed to the local storage or the delete API is called, the new samples, series, or tombstones are all written to the WAL.

How often is the content of the WAL fsynced to disk? Fsyncing is the process of 'flushing' the modifications of files in-memory to disk, which constitutes the majority of time when writing to disk. 

The WAL is written to disk in `segment files` - these are numbered files that rollover at 128MB and data in the [WAL Disk Format](https://github.com/ianbillett/prometheus/blob/7cf09b0395125280eb3e2d44b603349aeecacec1/tsdb/docs/format/wal.md).

WAL checkpoints

WAL truncated 











On-disk layout.

Why?

When is it called?

Encoded by default

### Checkpoints

## Blocks

Head block vs. On-disk blocks.

Directory layout


[tsdb.Head](https://github.com/prometheus/prometheus/blob/ff80690a6e1e52f9ef972fdc0c5b1a80ca7e1536/tsdb/head.go#L50)

### Compaction

### Truncation

### Index

#### Postings Table

#### Symbols

### Tombestones


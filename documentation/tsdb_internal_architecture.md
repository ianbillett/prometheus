# TSDB Internal Architecture

This document seeks to provide the reader with a comprehensive overview of the most important concepts and processes that govern the TSDB.

The information presented here is correct as of Prometheus v2.19.0, but will likely change in future releases. Please bear this in mind while reading.

## Chunks

Prometheus ingests data from targets in a 'pull' model, that is that Prometheus

Targets are

### Encoding

### Chunk Memory Mapping

### On-Disk Format

Segment files

## WAL

On-disk layout.

Why?

When is it called?

Encoded by default

### Checkpoints

## Blocks

Head block vs. On-disk blocks.

Directory layout

### Compaction

### Truncation

### Index

#### Postings Table

#### Symbols

### Tombestones


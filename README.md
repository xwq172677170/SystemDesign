# System Design Study Notes

These notes were converted from the draw.io diagrams in `../SysdDiagram`. The original diagram content is the baseline, and each design has been filled out with requirements, APIs, data models, deep dives and trade-offs, mostly as lists.

| # | Topic | Key ideas |
|---|---|---|
| 00 | [Fundamentals](00_Fundamentals.md) | Interview framework, SQL vs NoSQL, REST vs GraphQL, API design, retry/backoff/jitter |
| 01 | [Notification / Message Queue](01_Notification_MessageQueue.md) | Partitions, ISR, ack levels, push vs pull, DLQ, effectively-once |
| 02 | [Rate Limiter](02_RateLimiter.md) | Token/leaky bucket, sliding window, Redis + Lua, fail-open vs fail-close |
| 03 | [Distributed Cache / KV Store](03_Cache.md) | Consistent hashing, quorum, LSM vs B-Tree, Cassandra, eviction |
| 04 | [Top-K / Ad Click Aggregator](04_TopK_AdClickAggregator.md) | Count-Min Sketch, Kafka, Flink vs Spark, reconciliation |
| 05 | [TinyURL / PasteBin](05_TinyUrl_PasteBin.md) | Key generation (KGS, Snowflake), 301 vs 302, TTL cleanup |
| 06 | [Web Crawler + Search Engine](06_WebCrawler_SearchEngine.md) | URL frontier, politeness, dedup, inverted index |
| 07 | [Google Drive / Dropbox](07_GoogleDrive_CloudStorage.md) | Chunking, pre-signed URLs, delta sync, conflict policy |
| 08 | [YouTube / Netflix](08_Youtube_Netflix.md) | Transcoding DAG, ABR streaming, CDN |
| 09 | [News Feed](09_NewsFeed.md) | Fan-out on write vs read, hybrid for celebrities |
| 10 | [Chat](10_Chat.md) | WebSocket, presence, message IDs, offline inbox |
| 11 | [Search Autocomplete](11_SearchAutoComplete.md) | Trie with top-K per node, Levenshtein, sharding |
| 12 | [Ticket / Hotel Booking](12_TicketBooking.md) | Idempotency, locking, seat holds, Saga vs 2PC |
| 13 | [Stock Exchange](13_StockExchange.md) | Limit order book, sequencer, partition by ticker, UDP multicast |
| 14 | [Uber / Lyft](14_Uber_Lyft.md) | Geohash vs QuadTree, matching consistency, surge |
| 15 | [Amazon Locker](15_AmazonLocker.md) | Compartment allocation, edge offline validation |
| 16 | [Parking Lot](16_ParkingLot.md) | Min heap per entry, OOD, concurrency |

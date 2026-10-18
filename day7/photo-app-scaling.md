# SnapShare Scaling Plan

## Assumptions

- SnapShare lets users upload photos and scroll photos from people they follow.
- 10 million registered users; 10% active daily.
- Each active user uploads 1 photo and views 50 feed pages daily.
- Each original averages 2 MB; each photo gets one 50 KB thumbnail.
- Use 86,400 seconds/day, 365 days/year and 5 times average traffic for peak estimates.
- Use decimal units: 1 MB = 1,000 KB; 1 TB = 1,000,000 MB.
- Activity stays constant and every photo is retained throughout the year.
- One feed view is one page request; individual image requests are additional traffic.
- Storage totals exclude database metadata, backups, extra copies and other image sizes.

## Calculations

- Daily active users: 10,000,000 x 10% = **1,000,000**.
- Daily uploads: 1,000,000 x 1 = **1,000,000**.
- Average uploads: 1,000,000 / 86,400 = **11.57/second**.
- Peak uploads: (1,000,000 / 86,400) x 5 = **57.87/second** (plan for at least 58).
- Daily feed views: 1,000,000 x 50 = **50,000,000**.
- Average feed views: 50,000,000 / 86,400 = **578.70/second**.
- Peak feed views: (50,000,000 / 86,400) x 5 = **2,893.52/second** (plan for at least 2,894).
- Annual uploads: 1,000,000 x 365 = **365,000,000 photos**.
- Annual originals: 365,000,000 x 2 MB = **730 TB**.
- Annual thumbnails: 365,000,000 x 50 KB = **18.25 TB**.
- Combined new storage: **748.25 TB/year**, or **0.74825 PB/year**.

These are average and assumed peak rates, not guaranteed limits; backups and replication require additional storage capacity.

## Read-heavy design

SnapShare is **read-heavy by request count**, with 50 feed reads per upload before counting image downloads. Optimise reads using a CDN, cached feeds and a database read replica, and scale app servers horizontally. Uploads still consume substantial bandwidth despite their lower request count.

## Where photos go

Store original photos and thumbnails in **object storage**, not inside the database: large binary files would inflate database size, backups and replication traffic and compete with metadata queries. Keep accounts, follows and photo metadata (owner, object keys, caption, time and processing status) in the database. Deliver images through the CDN; private photos require access checks and short-lived signed URLs.

## Architecture diagram

```text
Clients -- API --> Load balancer --> App servers
                                      |-- feed lookup --> Cache
                                      |-- writes ------> Database primary
                                      |-- reads -------> Read replica
                                      |-- jobs --------> Queue --> Worker

Database primary -- replication --> Read replica
Worker -- read original / write thumbnail --> Object storage
Worker -- update photo status -------------> Database primary
Clients -- signed direct upload ----------> Object storage
Clients -- image reads --> CDN -- cache miss --> Object storage
App servers -- issue upload URL / verify object --> Object storage
```

### Component responsibilities

- **Clients:** Let users upload photos and request feed pages.
- **CDN:** Caches images near users to reduce latency and repeated object-storage reads.
- **Load balancer:** Distributes API requests across healthy app servers to avoid overloading one server.
- **App servers:** Check permissions, assemble feeds and coordinate uploads while allowing horizontal scaling.
- **Cache:** Keeps frequently requested feed data in memory to reduce database load.
- **Database primary:** Stores authoritative accounts, follow relationships and photo metadata with consistent writes.
- **Read replica:** Offloads eligible database reads from the primary, increasing read capacity.
- **Object storage:** Holds large originals and thumbnails separately from the metadata database.
- **Queue:** Buffers thumbnail jobs so uploads do not wait for image processing and workers can absorb bursts.
- **Worker:** Creates thumbnails in the background rather than slowing interactive requests.

## Upload flow

1. The signed-in client requests an upload through the load balancer.
2. An app server checks permissions and proposed file size/type, creates pending metadata with a unique object key, and issues a short-lived signed upload URL.
3. The client uploads the original directly to object storage, avoiding large transfers through app servers.
4. The client reports completion; the app server verifies the stored object and records a thumbnail job alongside the metadata update in a transactional outbox.
5. An app-server dispatcher sends the job to the queue, retrying delivery so a temporary queue failure does not lose it.
6. A worker reads and validates the original, creates a 50 KB thumbnail on average, and writes it to object storage under a deterministic key.
7. The worker updates thumbnail metadata and ready status in the primary, invalidates affected feed caches, and acknowledges the job after success.
8. Failed jobs retry with backoff; repeated failures go to a failed-job queue for inspection, and idempotent processing prevents duplicate results.
9. Ready photos appear in authorised followers' feeds; clients download originals or thumbnails through the CDN.
10. Read the primary when the uploader needs immediate visibility despite replica lag, and clean up abandoned uploads.

## Trade-offs

- **Caching versus freshness:** CDN and feed caches speed up reads but can serve stale content, so use bounded expiry and targeted invalidation at the cost of more complexity.
- **Replica capacity versus consistency:** Read replicas handle more reads but may lag behind writes, so use the primary for immediate post-upload reads.
- **Background processing versus availability:** Queued thumbnails keep uploads responsive but appear later and need retries, so show a processing state until ready.
- **Precomputed versus on-demand feeds:** Precomputed feeds make reads fast but multiply writes for popular accounts; on-demand feeds reduce write work but increase read cost, so begin with cached on-demand feeds and measure before adopting a hybrid.

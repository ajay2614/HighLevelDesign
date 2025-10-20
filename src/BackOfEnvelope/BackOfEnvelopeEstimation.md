“In this estimation, we are designing a backend for an Instagram-like application. We cover three main things:

Storage: How much disk/DB storage we need per year based on daily uploads and metadata.
Requests & Bandwidth: How many requests per second the system must handle, and the approximate network bandwidth required.
Servers (CPU/RAM): How many app and DB servers we need, based on requests/sec, and how much RAM is sufficient for hot posts and caching.

All numbers are approximate/back-of-envelope and rounded to 0, 50, or 100 for simplicity.”

Total Users: 10,000,000
Active Users per Day: 1,000,000

Uploads:

Each active user posts ~2 times/day
Total uploads/day: 2,000,000

Image Storage:

Image size per upload: 2 MB
Total image/day: 2,000,000 × 2 MB = 4,000,000 MB ≈ 4,000 GB ≈ 4 TB/day
Yearly storage: 4 TB × 365 ≈ 1,500 TB ≈ 1.5 PB

Metadata Storage:

Metadata per post: 200 bytes
Total metadata/day: 2,000,000 × 200 bytes ≈ 0.5 GB/day
Yearly metadata: 0.5 × 365 ≈ 150 GB
Total Storage Requirement:

≈ 2 PB/year (approximate, rounded)

Requests Per Second (QPS):

Uploads/sec: 2,000,000 ÷ 100,000 sec ≈ 20 → round 50/sec
Reads/sec (each post viewed 20×): 2,000,000 × 20 ÷ 100,000 ≈ 400 → round 500/sec
Total QPS ≈ 550 → round 600/sec for safety

Bandwidth:

Upload bandwidth/day: 4 TB/day
Bandwidth/sec: 4,000 GB ÷ 100,000 sec ≈ 40 MB/sec → round 50 MB/sec
Upload bandwidth Gbps: 50 × 8 ≈ 400 Mbps → round 0.5 Gbps
Reads/downloads/day: 2,000,000 × 20 × 2 MB = 80,000,000 MB ≈ 80,000 GB ≈ 80 TB/day
Bandwidth/sec: 80 TB ÷ 100,000 sec ≈ 800 MB/sec → round 1,000 MB/sec
Reads bandwidth Gbps: 1,000 × 8 ≈ 8,000 Mbps → round 10 Gbps

Use CDN to reduce read bandwidth

CPU / App Server Sizing:

Total QPS: ~600/sec

Assume 1 CPU core handles ~50 req/sec
Total CPU cores needed: 600 ÷ 50 ≈ 12 → round 12 cores
App servers: 3 servers × 4 cores each (with redundancy)
RAM per app server: 16 GB → enough for hot posts + request handling

Database Sizing:

Metadata storage ~150 GB/year → small
DB servers: 3 nodes (1 primary + 2 replicas)

Each DB server: 16 vCPU, 64 GB RAM, 2 TB SSD

Image storage handled in object store/S3

Cache / CDN:

Cache for thumbnails / hot posts optional

Cache size: ~8 TB → 60 nodes × 128 GB RAM

CDN: for images to reduce egress bandwidth

Notes:

All numbers are approximate/back-of-envelope
Round numbers to nearest 0, 50, 100 for clarity
Focus flow: DAU → posts → storage → QPS → CPU/RAM → cache/CDN
Mention redundancy and scaling for interview explanation
RAM sized for hot posts; older posts stored on disk/object storage
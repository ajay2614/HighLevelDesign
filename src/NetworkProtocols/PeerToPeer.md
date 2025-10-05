Peer-to-Peer (P2P) Model

Definition: P2P is a decentralized network where each node (peer) can act as both client and server simultaneously.
No central server; peers share resources directly with each other.
Dynamic Roles: 
Client: Downloads missing chunks from other peers.
Server: Uploads chunks already downloaded to other peers.

Roles are fluid and simultaneous.

File Sharing Mechanism:
Files are split into small pieces/chunks.
Peers download different chunks from multiple sources.
As soon as a peer has a chunk, it shares it immediately.

Seeders and Leechers:
Seeders: Peers with the complete file, uploading it to others.
Leechers: Peers currently downloading the file; may have partial data and can share chunks they already have.
A leecher becomes a seeder after downloading the full file.
More seeders → faster, more reliable downloads.

BitTorrent Example:
Movie split into chunks.
Peer A downloads first 100MB → uploads that to Peer B.
Peer B downloads missing chunks from Seeder and Peer A simultaneously.
Peer A = client for remaining chunks, server for downloaded chunks.

Analogy:
Like a potluck dinner — everyone can bring food (serve) and eat food (request) at the same time.
Even a peer with only part of the resources can contribute immediately.
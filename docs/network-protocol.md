# Chiral Network Protocol Documentation

## Protocol Overview

The Chiral Network implements a multi-layered protocol stack combining blockchain consensus, distributed hash table routing, and peer-to-peer file transfer protocols. This document details the network protocols, message formats, and communication patterns.

## Protocol Stack

```
┌─────────────────────────────────────────┐
│         Application Layer               │
│   File Transfer | DHT | Mining          │
├─────────────────────────────────────────┤
│         Session Layer                   │
│   Authentication | Encryption | State   │
├─────────────────────────────────────────┤
│         Network Layer                   │
│   DHT Routing | Peer Discovery | NAT    │
├─────────────────────────────────────────┤
│         Transport Layer                 │
│   libp2p | TCP | UDP | QUIC | WebRTC    │
└─────────────────────────────────────────┘
```

## Core Protocols

### 1. Peer Discovery Protocol

#### Bootstrap Process (Current Implementation)

1. **Connect to the bootstrap DHT node**  
   Every node loads the configured `bootstrap_nodes` list (currently a single multiaddress such as `/ip4/bootstrap.chiral.network/tcp/4001/p2p/QmBootstrap`) and dials it on startup. That node runs the same libp2p/Kademlia behaviour as any other peer but is flagged with `is_bootstrap=true`, kept online permanently, and is configured not to publish provider records so it acts purely as a router.

2. **Seed the local routing table**  
   Once the connection succeeds, we add the bootstrap peer ID and address into the local Kademlia table. No custom “peer list” RPC is issued—the bootstrap node simply shares the peers it already knows through standard Kademlia gossip.

3. **Run the initial Kademlia bootstrap walk**  
   After the connection is established, the client immediately invokes `kademlia.bootstrap()`. This kicks off the built-in random walk (successive `FIND_NODE` queries) using the bootstrap node as the starting point, which populates the routing table with additional peers and providers.

4. **Keep the table fresh**  
   Non-bootstrap peers continue to run the periodic bootstrap loop (1 s interval) while the node is up, ensuring they resync with the network if entries age out. The dedicated bootstrap node disables this interval, but it remains available so new peers can repeat steps 1–3 at any time.

##### Operational Notes

- The bootstrap node exposes only the libp2p/DHT service (no extra REST endpoints) and listens on the same ports as any peer.
- Today the network relies on a single bootstrap address; adding secondary bootstrap nodes is recommended to avoid a single point of failure.

#### Message Format

```
PeerDiscovery Message {
  header: {
    version: u16,           // Protocol version (0x0001)
    message_type: u8,       // Message type enum
    request_id: u32,        // Request identifier
    timestamp: u64,         // Unix timestamp
  },
  sender: {
    node_id: [u8; 32],     // Node public key hash
    addresses: Vec<String>, // Multiaddresses
    capabilities: u32,      // Capability flags
  },
  payload: MessagePayload,  // Type-specific data
  signature: [u8; 64],     // Ed25519 signature
}
```

### 2. DHT Protocol (Kademlia)

#### Node ID Generation

```
Node ID = SHA256(public_key || nonce)
Distance = XOR(NodeID_A, NodeID_B)
```

#### Routing Table Structure

```
K-Buckets (k=20, b=160):
┌────────────────────────────────┐
│ Bucket 0: Distance 2^0        │ → [20 nodes max]
│ Bucket 1: Distance 2^1        │ → [20 nodes max]
│ ...                            │
│ Bucket 159: Distance 2^159    │ → [20 nodes max]
└────────────────────────────────┘
```

#### DHT Operations

##### PING

```
Request:
{
  type: "PING",
  sender_id: [u8; 20],
  random: [u8; 20]
}

Response:
{
  type: "PONG",
  sender_id: [u8; 20],
  echo: [u8; 20]
}
```

##### FIND_NODE

```
Request:
{
  type: "FIND_NODE",
  sender_id: [u8; 20],
  target_id: [u8; 20]
}

Response:
{
  type: "NODES",
  sender_id: [u8; 20],
  nodes: [{
    id: [u8; 20],
    ip: [u8; 4] | [u8; 16],
    port: u16
  }]
}
```

##### STORE

```
Request:
{
  type: "STORE",
  sender_id: [u8; 20],
  key: [u8; 32],
  value: Vec<u8>,
  ttl: u32
}

Response:
{
  type: "STORED",
  sender_id: [u8; 20],
  key: [u8; 32],
  expires: u64
}
```

##### FIND_VALUE

```
Request:
{
  type: "FIND_VALUE",
  sender_id: [u8; 20],
  key: [u8; 32]
}

Response (if found):
{
  type: "VALUE",
  sender_id: [u8; 20],
  key: [u8; 32],
  value: Vec<u8>
}

Response (if not found):
{
  type: "NODES",
  sender_id: [u8; 20],
  nodes: [...]
}
```

### 3. File Transfer Protocol

#### Minimal DHT Record

The DHT stores only essential file information. All protocol-specific details are obtained through the messaging protocol.

```json
{
  "fileHash": "sha256_abc123def456...", // SHA-256 content hash (64 hex chars, no 0x prefix)
  "fileName": "document.pdf", // Original filename
  "fileSize": 1048576 // Total size in bytes
}
```

#### Messaging Protocol

File transfer uses a four-step query flow:

##### Step 1: INFO_REQUEST

Downloader requests transfer terms from a seeder.

```
Request:
{
  type: "INFO_REQUEST",
  file_hash: "sha256_abc123...",
  request_id: u32,
  sender_id: [u8; 20]
}

Response:
{
  type: "INFO_RESPONSE",
  file_hash: "sha256_abc123...",
  content_protocol: "http" | "ftp" | "bittorrent" | "webrtc" | "ed2k" | "bitswap",
  price_per_mb: "0.001",
  wallet_address: "0x742d35Cc6634C0532925a3b8D0C9e0c8b346b983",
  supported_versions: [1, 2],
  min_payment_increment_mb: 1,
  request_id: u32
}
```

##### Step 2: PROTOCOL_SPECIFIC_REQUEST

Downloader requests protocol-specific connection details.

```
Request:
{
  type: "PROTOCOL_SPECIFIC_REQUEST",
  protocol: "http" | "ftp" | "bittorrent" | "webrtc" | "ed2k" | "bitswap",
  file_hash: "sha256_abc123...",
  info_requested: "connection_details",
  request_id: u32
}
```

##### Step 3: PROTOCOL_SPECIFIC_RESPONSE

Seeder returns protocol-specific details.

```
// HTTP
{
  type: "PROTOCOL_SPECIFIC_RESPONSE",
  protocol: "http",
  endpoint: "http://192.168.1.100:8080/file/sha256_abc123...",
  timeout: 30
}

// FTP
{
  type: "PROTOCOL_SPECIFIC_RESPONSE",
  protocol: "ftp",
  host: "ftp.example.com",
  port: 21,
  username: "anonymous",
  path: "/pub/file.zip",
  passive: true,
  tls: true
}

// BitTorrent
{
  type: "PROTOCOL_SPECIFIC_RESPONSE",
  protocol: "bittorrent",
  info_hash: "abc123sha1...",
  trackers: ["udp://tracker.example.com:6969/announce"],
  magnet: "magnet:?xt=urn:btih:..."
}

// WebRTC
{
  type: "PROTOCOL_SPECIFIC_RESPONSE",
  protocol: "webrtc",
  signaling_url: "ws://signaling.example.com:8888",
  ice_servers: [
    { "urls": "stun:stun.l.google.com:19302" },
    { "urls": "turn:turn.example.com:3478", "username": "...", "credential": "..." }
  ]
}

// ed2k
{
  type: "PROTOCOL_SPECIFIC_RESPONSE",
  protocol: "ed2k",
  ed2k_link: "ed2k://|file|example.zip|12345678|..."
}

// BitSwap
{
  type: "PROTOCOL_SPECIFIC_RESPONSE",
  protocol: "bitswap",
  cids: ["bafybeifx...", "bafybeigy...", "bafybeihz..."],
  peer_id: "12D3KooW...",
  multiaddrs: ["/ip4/192.168.1.100/tcp/4001/p2p/12D3KooW..."]
}
```

##### Step 4: Transfer

After receiving protocol-specific details, the downloader initiates the transfer using the appropriate protocol and makes payments according to the terms received in INFO_RESPONSE.

#### Transfer Protocol

The actual file transfer protocol depends on the `content_protocol` returned in INFO_RESPONSE:

- **HTTP/FTP**: Standard range-based downloads
- **BitTorrent**: BitTorrent peer-wire protocol
- **WebRTC**: WebRTC data channels
- **ed2k**: eDonkey2000 chunk-based protocol
- **BitSwap**: IPFS Bitswap protocol with CID-based block exchange

Chunk verification is handled according to each protocol's native mechanism:

- HTTP/FTP: SHA-256 hash verification
- BitTorrent: SHA-1 piece hashes
- ed2k: MD4 chunk hashes
- BitSwap: CID verification
- WebRTC: Application-defined

#### Parallel Transfer Optimization

```
MaxParallelChunks = 10
WindowSize = 5

For chunks 0..n:
  While active_transfers < MaxParallelChunks:
    Request next chunk
    Track in flight

  On chunk received:
    Verify and store
    Request next chunk
    Update progress
```

### 4. Blockchain Protocol

#### Block Structure

```
EthereumBlock {
  header: {
    parentHash: [u8; 32],      // Previous block hash
    sha3Uncles: [u8; 32],      // Uncle blocks hash
    miner: [u8; 20],           // Coinbase address
    stateRoot: [u8; 32],       // State trie root
    transactionsRoot: [u8; 32], // Transaction trie root
    receiptsRoot: [u8; 32],    // Receipt trie root
    difficulty: U256,          // Block difficulty
    number: u64,               // Block number
    gasLimit: u64,             // Gas limit
    gasUsed: u64,              // Gas used
    timestamp: u64,            // Unix timestamp
    mixHash: [u8; 32],         // Ethash mix hash
    nonce: u64,                // Ethash nonce
  },
  transactions: Vec<Transaction>,
  uncles: Vec<BlockHeader>,
}
```

#### Transaction Format

```
EthereumTransaction {
  nonce: u64,
  gasPrice: U256,
  gasLimit: u64,
  to: Option<[u8; 20]>,       // Recipient address
  value: U256,                // Amount in wei
  data: Vec<u8>,              // Input data
  v: u64,                     // Recovery ID
  r: U256,                    // ECDSA signature r
  s: U256,                    // ECDSA signature s
}

TransactionTypes:
- Simple Transfer: to != null, data = []
- Contract Call: to != null, data != []
- Contract Creation: to = null, data = contract_code
- Storage Operation: encoded in data field
```

#### Consensus Messages

##### NewBlock

```
{
  type: "NEW_BLOCK",
  block: Block,
  total_difficulty: U256,
  sender: NodeId
}
```

##### GetBlockHeaders

```
{
  type: "GET_BLOCK_HEADERS",
  block: Either<u64, [u8; 32]>,  // Block number or hash
  maxHeaders: u32,
  skip: u32,
  reverse: bool
}
```

##### NewPooledTransactionHashes

```
{
  type: "NEW_POOLED_TRANSACTION_HASHES",
  hashes: Vec<[u8; 32]>
}
```

### 5. Provider Verification Protocol

#### On-Stream Chunk Validation

1. Requesters query DHT for file metadata (fileHash, fileName, fileSize).
2. Requesters send INFO_REQUEST to seeders to obtain transfer terms.
3. Requesters send PROTOCOL_SPECIFIC_REQUEST to obtain protocol-specific connection details.
4. Transfer proceeds using the agreed protocol (HTTP, FTP, BitTorrent, WebRTC, ed2k, BitSwap).
5. Chunk verification follows each protocol's native mechanism:
   - HTTP/FTP: SHA-256 verification
   - BitTorrent: SHA-1 piece hashes
   - ed2k: MD4 verification
   - BitSwap: CID verification
6. Any failed verification aborts the transfer, blacklists the provider locally, and emits a reputation penalty signal.
7. Providers that successfully deliver all requested chunks earn a positive reputation update.

### 6. Multi-Network Integration: Chiral and BitTorrent

To enhance file availability and download speed, the Chiral client integrates with the public BitTorrent network. This creates a hybrid system where the client can fetch file pieces from both the private, reputation-based Chiral P2P network and the massive, public BitTorrent swarm simultaneously.

#### Dual-Network Orchestration

The system does **not** merge the two networks. Instead, the `ProtocolManager` acts as an orchestrator, managing two parallel network handlers:

1.  **Chiral P2P Handler**: Interacts with the libp2p-based Kademlia DHT to find and communicate with other Chiral peers.
2.  **BitTorrent Handler**: Interacts with the public BitTorrent DHT (Mainline DHT) and public trackers to find and communicate with standard BitTorrent clients.

This allows the Chiral client to source file pieces from a high-reputation Chiral peer and a public torrent swarm at the same time, with the `multi_source_download` engine assembling the final file from all incoming pieces.

#### BitTorrent Integration via Messaging Protocol

To enable Chiral peers to discover BitTorrent sources, the messaging protocol is used:

1.  **DHT Lookup**: Client queries DHT for minimal file metadata (fileHash, fileName, fileSize).
2.  **INFO_REQUEST**: Client sends INFO_REQUEST to seeders.
3.  **INFO_RESPONSE**: Seeder indicates BitTorrent is available via `content_protocol: "bittorrent"`.
4.  **PROTOCOL_SPECIFIC_REQUEST**: Client requests BitTorrent connection details.
5.  **PROTOCOL_SPECIFIC_RESPONSE**: Seeder returns `info_hash`, `trackers`, and `magnet` link.

This approach keeps DHT metadata minimal while allowing protocol-specific details to be discovered through the query flow.

## Network Protocols

### 1. libp2p Integration

#### Protocol Multiplexing

```yaml
protocols:
  /chiral/kad/1.0.0: # Kademlia DHT
    handler: dht_handler
  /chiral/transfer/1.0.0: # File transfer
    handler: transfer_handler
  /chiral/dht/1.0.0: # DHT protocol
    handler: dht_handler
  /chiral/eth/1.0.0: # Ethereum-compatible sync
    handler: eth_handler
```

#### Stream Multiplexing

```
Connection
    ├── Stream 1: DHT queries
    ├── Stream 2: File transfer
    ├── Stream 3: Blockchain sync
    └── Stream 4: Control messages
```

### 2. NAT Traversal

#### STUN Protocol

```
STUN Request:
{
  type: "BINDING_REQUEST",
  transaction_id: [u8; 12],
  attributes: {
    USERNAME: "peer_id",
    MESSAGE_INTEGRITY: [u8; 20]
  }
}

STUN Response:
{
  type: "BINDING_RESPONSE",
  transaction_id: [u8; 12],
  attributes: {
    XOR_MAPPED_ADDRESS: "public_ip:port",
    SOFTWARE: "chiral/1.0.0"
  }
}
```

#### TURN Relay

```
Relay Protocol:
Client A → TURN Server → Client B

1. Allocate Relay
   → ALLOCATE_REQUEST
   ← ALLOCATE_RESPONSE(relay_address)

2. Create Permission
   → CREATE_PERMISSION(peer_address)
   ← PERMISSION_CREATED

3. Send Data
   → SEND_INDICATION(data, peer_address)
   Server → Peer: DATA_INDICATION
```

### 3. WebRTC Integration

#### Signaling Protocol

```javascript
// Offer
{
  type: "offer",
  sdp: "v=0\r\no=- ... ",
  ice_candidates: [
    {
      candidate: "candidate:1 1 UDP ...",
      sdpMLineIndex: 0
    }
  ]
}

// Answer
{
  type: "answer",
  sdp: "v=0\r\no=- ... ",
  ice_candidates: [...]
}
```

#### Data Channel Protocol

```
DataChannel Configuration:
{
  ordered: true,
  maxRetransmits: 3,
  maxPacketLifeTime: 5000,
  protocol: "chiral-transfer",
  negotiated: false
}
```

## Message Serialization

### Protocol Buffers Schema

```protobuf
syntax = "proto3";
package chiral;

message Envelope {
  uint32 version = 1;
  string message_type = 2;
  bytes payload = 3;
  uint64 timestamp = 4;
  bytes signature = 5;
}

message Node {
  bytes id = 1;
  repeated string addresses = 2;
  uint64 last_seen = 3;
  double reputation = 4;
}

message FileRequest {
  string hash = 1;
  uint32 chunk_index = 2;
  uint32 offset = 3;
  uint32 length = 4;
}

message FileResponse {
  string hash = 1;
  uint32 chunk_index = 2;
  bytes data = 3;
}
```

### MessagePack Format

```
Message Structure:
┌──────────┬──────────┬──────────┬──────────┐
│  Magic   │  Version │  Type    │  Length  │
│  2 bytes │  2 bytes │  2 bytes │  4 bytes │
├──────────┴──────────┴──────────┴──────────┤
│              Payload (variable)            │
├────────────────────────────────────────────┤
│            Checksum (4 bytes)              │
└────────────────────────────────────────────┘
```

## Protocol Negotiation

### Version Negotiation

```
Client: HELLO {
  versions: [0x0003, 0x0002, 0x0001],
  capabilities: ["serve", "relay", "mine"]
}

Server: HELLO_ACK {
  selected_version: 0x0002,
  capabilities: ["serve", "relay"],
  features: ["encryption", "compression"]
}
```

### Capability Discovery

```
Capabilities Bitmap:
Bit 0: Storage Node
Bit 1: Relay Node
Bit 2: Mining Node
Bit 3: DHT Node
Bit 4: Bootstrap Node
Bit 5: Archive Node
Bit 6-31: Reserved
```

## Network Topology

### Overlay Network Structure

```
Super Nodes (High Bandwidth/Storage)
    │
    ├── Regional Hubs
    │       │
    │       ├── Edge Nodes
    │               │
    │               └── Client Nodes
    │
    │
    │
    └── Relay Nodes
            │
            └── NAT-ed Clients
```

### Routing Strategies

#### Iterative Routing

```
1. Query α closest nodes
2. Wait for responses
3. Query α next closest
4. Repeat until target found
```

#### Recursive Routing

```
1. Query closest node
2. Node forwards query
3. Continues recursively
4. Response returns via path
```

## Quality of Service

### Priority Levels

```
enum Priority {
  Critical = 0,  // System messages
  High = 1,      // Financial transactions
  Normal = 2,    // File transfers
  Low = 3,       // Background sync
}
```

### Bandwidth Allocation

```
Total Bandwidth = 100 Mbps
- Critical: 10% reserved
- High: 30% guaranteed
- Normal: 50% shared
- Low: 10% best effort
```

### Flow Control

```
Window-based Flow Control:
- Initial window: 64 KB
- Maximum window: 1 MB
- Increment: 32 KB per RTT
- Backoff: 50% on congestion
```

## Protocol Security

### Message Authentication

```
HMAC-SHA256(key, message) where:
- key = shared_secret
- message = header || payload
```

### Replay Attack Prevention

```
Requirements:
1. Timestamp within 5 minutes
2. Nonce not in recent set
3. Sequence number increments
```

### Protocol Fuzzing

```yaml
fuzzing_targets:
  - message_parsing
  - state_transitions
  - error_handling
  - boundary_conditions
```

## Performance Metrics

### Latency Targets

| Operation         | Target | Maximum |
| ----------------- | ------ | ------- |
| Ping              | 50ms   | 200ms   |
| DHT Lookup        | 500ms  | 2s      |
| Chunk Request     | 100ms  | 1s      |
| Block Propagation | 1s     | 5s      |

### Throughput Targets

| Operation      | Target  | Minimum |
| -------------- | ------- | ------- |
| File Upload    | 10 MB/s | 1 MB/s  |
| File Download  | 20 MB/s | 2 MB/s  |
| DHT Operations | 100/s   | 10/s    |
| Transactions   | 100/s   | 10/s    |

## Protocol Extensions

### Custom Protocol Registration

```typescript
interface ProtocolHandler {
  name: string;
  version: string;
  handler: (stream: Stream) => Promise<void>;
}

network.registerProtocol({
  name: "/chiral/custom/1.0.0",
  version: "1.0.0",
  handler: async (stream) => {
    // Handle protocol
  },
});
```

### Protocol Upgrade Path

```
Version 1.0.0 → 1.1.0:
- Backward compatible
- New optional fields
- Deprecation warnings

Version 1.x → 2.0.0:
- Breaking changes
- Migration period
- Dual-stack support
```

## Debugging & Monitoring

### Protocol Tracing

```
TRACE [2024-01-01 00:00:00] DHT FIND_NODE
  → Target: 0x1234...
  ← Nodes: 20
  Duration: 150ms

DEBUG [2024-01-01 00:00:01] FILE_TRANSFER
  → Request chunk 5 of file 0xabcd...
  ← Received 262144 bytes
  Verification: OK
```

### Network Diagnostics

```bash
# Test connectivity
chiral-cli network ping <peer_id>

# Trace route
chiral-cli network trace <file_hash>

# Protocol statistics
chiral-cli network stats --protocol=dht
```

## Protocol Compliance

### Standards Compliance

- libp2p Specification v1.0
- Ethereum Wire Protocol (RLPx)
- Ethereum DevP2P Protocol
- WebRTC RFC 8825
- JSON-RPC 2.0 (Ethereum-compatible)

### Testing Suite

```yaml
test_categories:
  conformance:
    - message_format
    - state_machine
    - error_codes
  interoperability:
    - version_compatibility
    - cross_platform
    - network_conditions
  performance:
    - throughput
    - latency
    - scalability
```

## Future Protocol Enhancements

### Planned Features

1. **QUIC Transport:** Lower latency connections
2. **GraphSync:** Efficient graph synchronization
3. **Bitswap:** Content exchange protocol
4. **Gossipsub:** Pub/sub messaging
5. **Noise Protocol:** Modern crypto handshake

### Research Areas

- Quantum-resistant protocols
- Machine learning optimization
- Satellite communication
- Mesh networking
- Edge computing integration

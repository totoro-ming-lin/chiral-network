# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Chiral Network is a decentralized P2P file sharing platform combining blockchain technology with DHT-based file storage. It implements a BitTorrent-like continuous seeding model with privacy-first design and is built using:

- **Frontend**: Svelte 5 + TypeScript + Tailwind CSS
- **Desktop Runtime**: Tauri 2 (Rust backend)
- **P2P Stack**: libp2p v0.54 (Kademlia DHT, WebRTC, NAT traversal)
- **Blockchain**: Ethereum-compatible network via Geth integration

## Essential Build Commands

### Development
```bash
npm run dev              # Web dev server (127.0.0.1:1420)
npm run tauri:dev        # Desktop app with hot reload
npm run check            # TypeScript type checking
```

### Testing
```bash
# Frontend tests (vitest)
npm test                 # Run all unit/integration tests
npm run test:watch       # Watch mode
npm run test:e2e         # End-to-end tests (excludes real-network)
npm run test:e2e:real    # Real network E2E tests (spawns actual nodes)

# Rust tests
cd src-tauri
cargo test                              # All Rust tests
cargo test --test <test_name>           # Specific integration test
cargo test <test_name> -- --nocapture   # Show output during test
```

### Production Build
```bash
npm run build           # Web production build
npm run tauri:build     # Desktop production build (creates installers)
```

### Special Commands
```bash
npm run signaling-server              # Run WebRTC signaling server
npm run test:e2e:real:uploader        # E2E uploader node only
npm run test:e2e:real:downloader      # E2E downloader node only
```

## Core Architecture Concepts

### 1. Decoupled Payment and Data Transfer

**Critical Design Principle**: Payment layer is completely separate from data transfer protocols.

```
Application Layer (File Management, UI)
         │
    ┌────┴────┐
    │         │
Payment    Data Transfer
(Blockchain)  (HTTP/BitTorrent/WebTorrent/ed2k)
    │         │
    └────┬────┘
         │
   Settlement
```

This separation enables:
- Protocol flexibility without changing payment logic
- Adding new protocols (IPFS, custom) without touching blockchain code
- Using established protocols (BitTorrent, ed2k) unmodified
- Independent testing and evolution of each layer

### 2. Protocol Manager Architecture

The `ProtocolManager` (conceptual, see `docs/protocols.md`) provides a unified interface via `IContentProtocol`:

**Key Operations**:
- `getPeersServing()` - Find peers with content
- `getFileMetadata()` - DHT metadata lookup
- `getTransferTerms()` - Pricing/payment info
- `getProtocolDetails()` - Protocol-specific config
- `getContentFrom()` - Download from peer
- `startSeeding()` / `stopSeeding()` - Upload/share
- `pauseDownload()` / `resumeDownload()` / `cancelDownload()` - Transfer control

### 3. DHT Bootstrap Flow (libp2p/Kademlia)

Current implementation uses **single bootstrap node** architecture:

1. **Connect to bootstrap node** - Dial configured multiaddress (e.g., `/ip4/bootstrap.chiral.network/tcp/4001/p2p/QmBootstrap`)
2. **Seed routing table** - Add bootstrap peer to local Kademlia
3. **Bootstrap walk** - Execute `kademlia.bootstrap()` to populate routing table via `FIND_NODE` queries
4. **Periodic refresh** - Non-bootstrap peers run 1s interval loop to keep table fresh

**Note**: Bootstrap node is flagged `is_bootstrap=true`, kept online permanently, and configured NOT to publish provider records (acts as pure router).

### 4. Frontend-Backend Communication (Tauri)

**Rust → Frontend**: Use Tauri `invoke()` system
- Frontend calls: `await invoke('command_name', { params })`
- Backend exports via `#[tauri::command]` macro
- All commands registered in `src-tauri/src/main.rs`

**Key Services**:
- `src-tauri/src/dht.rs` - DHT operations (412KB file, core networking)
- `src-tauri/src/bittorrent_handler.rs` - BitTorrent protocol (98KB)
- `src-tauri/src/ed2k_client.rs` - eDonkey2000 protocol (64KB)
- `src-tauri/src/ethereum.rs` - Blockchain integration (118KB)

See `docs/tauri-commands.md` for full command reference.

### 5. State Management (Svelte Stores)

Central state in `src/lib/stores.ts`:

```typescript
// File management
files: FileItem[]                    // All files (states: downloading, seeding, paused, etc.)
downloadQueue: FileItem[]
activeDownloads: number

// Network
peers: PeerInfo[]                    // Connected peers with reputation metrics
networkStats: NetworkStats
networkStatus: NetworkStatus
peerGeoDistribution: Derived

// Wallet & Mining
wallet: WalletInfo
etcAccount: ETCAccount | null
transactions: Transaction[]
miningState: MiningState

// Privacy & Security
userLocation: string
blacklist: BlacklistEntry[]
suspiciousActivity: ActivityLog[]
settings: AppSettings
```

Additional stores:
- `src/lib/reputationStore.ts` - Peer reputation system
- `src/lib/stores/searchHistory.ts` - Download search history

### 6. Internationalization (i18n)

**Setup**: `src/i18n/i18n.ts` with `svelte-i18n` library
**Languages**: EN, ES, RU, ZH, KO
**Translation Files**: `src/locales/*.json`
**Usage**: `$t('key.path')` in components

**Flow**:
1. `setupI18n()` called in `App.svelte` onMount
2. Auto-detect language via geolocation
3. Persist preference in localStorage
4. All UI text must use translation keys

### 7. Multi-Protocol Download Support

**Supported Protocols**:
- **HTTP/HTTPS**: Baseline with pause/resume via `download_restart.rs`
- **BitTorrent**: Native integration via `librqbit` crate + custom Chiral extension
- **WebTorrent**: Browser-compatible WebRTC-based torrenting
- **ed2k/eDonkey2000**: Legacy P2P protocol support
- **FTP/FTPS**: Resume-capable downloads with bookmarks

**Multi-Source Download**: `multiSourceDownloadService.ts` orchestrates parallel chunk downloads from multiple peers with intelligent peer selection based on reputation.

### 8. Testing Architecture

**Frontend Tests** (Vitest):
- Unit tests in `tests/*.test.ts`
- Component tests in `tests/components/`
- E2E tests in `tests/e2e/`
- Real network tests spawn actual Tauri nodes: `tests/e2e/real-network.test.ts`

**Rust Tests**:
- Integration tests in `src-tauri/tests/`
- 81 Rust source files in `src-tauri/src/`
- Key test files:
  - `dht_integration-test.rs` (DHT functionality)
  - `bittorrent_integration_tests.rs` (BitTorrent protocol)
  - `nat_traversal_e2e_test.rs` (NAT/relay testing)
  - `download_restart_test.rs` (resume capability)

**Test Patterns**:
```bash
# Run specific Rust test with output
cargo test dht_integration -- --nocapture

# Run E2E test that spawns real nodes
npm run test:e2e:real

# Run cross-machine E2E test (uploader on one machine, downloader on another)
npm run test:e2e:real:uploader    # Machine A
npm run test:e2e:real:downloader  # Machine B
```

## Key Implementation Details

### NAT Traversal Stack
1. **AutoNAT v2**: Automatic reachability detection with confidence scoring
2. **Circuit Relay v2**: Relay reservation for NAT'd peers
3. **DCUtR**: Direct Connection Upgrade through Relay (hole punching)
4. **mDNS**: Local network peer discovery
5. **SOCKS5 Proxy**: Privacy-focused routing

### File Sharing Model
- Files are **instantly seeded** when added (no "pending" state)
- Each file gets SHA-256 hash
- Metadata published to Kademlia DHT
- Files show real-time seeder/leecher counts
- Continuous seeding until manually removed
- Support for AES-256-GCM encryption with PBKDF2 key derivation
- Multi-CID support for chunked files (256KB chunks)

### Blockchain Parameters
- **Network ID**: 98765
- **Chain ID**: 98765 (EIP-155)
- **Block Time**: ~15 seconds
- **Mining Algorithm**: Ethash (ASIC-resistant PoW)
- **Initial Reward**: 2 Chiral
- **Ports**: P2P (30304), RPC (8546), WebSocket (8547), File Transfer (8080), DHT (4001)

### Routing System
- **Router**: `@mateothegreat/svelte5-router` (Svelte 5 compatible)
- **Route Config**: Defined in `App.svelte`
- **Pages**: Download (default), Upload, Network, Relay, Mining, Proxy, Analytics, Reputation, Account, Settings
- **404 Handling**: NotFound page

## Common Development Tasks

### Adding a New Frontend Service

1. Create file in `src/lib/services/`
2. Export as singleton or factory pattern
3. Add TypeScript interfaces for all types
4. Integrate with Tauri backend via `invoke()` if needed
5. Update relevant stores in `src/lib/stores.ts`
6. Add error handling and logging

### Working with DHT

1. DHT operations are in Rust backend: `src-tauri/src/dht.rs`
2. Frontend calls via Tauri `invoke('dht_*', { params })`
3. File metadata uses `FileMetadata` interface
4. Monitor DHT health with `DhtHealth` interface
5. Handle NAT traversal states (public/private/unknown)

### Adding Translations

1. Add keys to ALL locale files in `src/locales/`: `en.json`, `es.json`, `ru.json`, `zh.json`, `ko.json`
2. Use descriptive key paths: `reputation.filters.trustLevel`
3. Test with multiple languages
4. Maintain consistency across translations

### Running System Diagnostics

Built-in diagnostics available in Settings → Diagnostics:
- 13 comprehensive health checks across 5 categories
- Environment, network, storage, security, system tests
- Real-time DHT, AutoNAT, Circuit Relay status
- Exportable text reports for troubleshooting

See `src/lib/services/diagnosticsService.ts`

## Important Constraints

### What NOT to Implement

**Commercial & Piracy Features** (NEVER add):
- Global file search/discovery (could enable piracy)
- Price fields or payment systems for file trading
- File marketplace or trading features
- Content recommendations
- Social features (comments, likes, reviews)
- Analytics that could track users

**VPN/Anonymity Network Features** (We are NOT building a VPN):
- ❌ VPN service functionality
- ❌ General internet traffic routing
- ❌ Exit node functionality
- ❌ Anonymous browsing capabilities
- ❌ Traffic mixing/onion routing

**What we DO support** (limited to file sharing):
- ✅ SOCKS5 proxy support (use existing proxies like Tor)
- ✅ Circuit Relay v2 (for NAT traversal, not anonymity)
- ✅ File encryption (protect file content)
- ✅ Anonymous mode (hide IP during P2P file transfers only)

### Development Guidelines

1. **No Commercial Elements**: Never add pricing, trading, or marketplace features
2. **Privacy First**: Always consider user privacy and anonymity
3. **Legitimate Use**: Design for legal file sharing use cases only
4. **Decentralized**: No centralized servers or intermediaries
5. **BitTorrent Model**: Files should seed continuously, not "upload once"
6. **i18n Support**: Add translation keys for all new UI text
7. **Type Safety**: Use TypeScript interfaces for all data structures

## Module Organization

### Rust Backend (`src-tauri/src/`)
- 81 Rust files (55 in root, 26 in subdirectories)
- Key modules exported in `lib.rs`:
  - `protocols` - Protocol implementations
  - `dht` - DHT/libp2p integration
  - `multi_source_download` - Multi-source orchestration
  - `download_restart` - Resume capability
  - `bittorrent_handler` - BitTorrent protocol
  - `ed2k_client` - eDonkey2000 protocol
  - `ftp_client` / `ftp_bookmarks` - FTP support
  - `ethereum` - Blockchain integration
  - `encryption` / `keystore` - Security
  - `reputation` - Peer reputation system
  - `payment_checkpoint` - Payment integration

### Frontend (`src/`)
```
src/
├── i18n/                    # Internationalization
├── lib/
│   ├── components/          # Reusable components
│   │   ├── download/        # Download-specific
│   │   ├── ui/              # UI primitives
│   │   └── wallet/          # Wallet components
│   ├── services/            # Frontend services (39 files)
│   ├── stores/              # Additional stores
│   ├── types/               # TypeScript types
│   ├── utils/               # Utility functions
│   ├── wallet/              # HD wallet (BIP32/BIP39)
│   ├── dht.ts               # DHT config
│   └── stores.ts            # Main state
├── locales/                 # Translation JSON files
├── pages/                   # Application pages (10 pages)
├── routes/                  # Special routes
├── App.svelte               # Main app + routing
└── main.ts                  # Entry point
```

## Documentation

- `docs/architecture.md` - Decoupled architecture design
- `docs/protocols.md` - Protocol Manager and IContentProtocol interface
- `docs/network-protocol.md` - DHT, bootstrap, and message formats
- `docs/technical-specifications.md` - Network parameters and specs
- `docs/implementation-guide.md` - Development workflows
- `docs/tauri-commands.md` - Backend command reference
- `docs/user-guide.md` - End-user documentation

Full documentation index: `docs/index.md`

## Troubleshooting

### Common Issues

1. **DHT not connecting**: Run diagnostics (Settings → Diagnostics), verify bootstrap nodes
2. **Mining not starting**: Check Geth service initialization
3. **Tauri invoke errors**: Ensure backend commands are registered in `main.rs`
4. **Storage path errors**: Diagnostics will show if directory is missing/inaccessible
5. **NAT/Relay issues**: Check diagnostics for AutoNAT and Circuit Relay status

### Debug Commands
```bash
# Clean rebuild
rm -rf node_modules dist
npm install
npm run build

# Type checking
npm run check

# Verbose test output
npm test -- --reporter=verbose
cargo test -- --nocapture
```

## Repository Notes

- **Main Branch**: `main`
- **Current Version**: v0.1.0
- **License**: MIT
- **Support**: Zulip (https://brooknet.zulipchat.com/join/f3jj4k2okvlfpu5vykz5kkk5/)
- **Issues**: GitHub Issues

---

*Last Updated: January 2025*

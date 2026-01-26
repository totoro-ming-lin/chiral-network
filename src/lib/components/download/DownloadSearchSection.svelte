<script lang="ts">
  import { invoke } from '@tauri-apps/api/core';
  import { listen } from '@tauri-apps/api/event';
  import Card from '$lib/components/ui/card.svelte';
  import Input from '$lib/components/ui/input.svelte';
  import Label from '$lib/components/ui/label.svelte';
  import Button from '$lib/components/ui/button.svelte';
  import { Search, X, History, RotateCcw, AlertCircle, CheckCircle2 } from 'lucide-svelte';
  import { createEventDispatcher, onDestroy, onMount } from 'svelte';
  import { get } from 'svelte/store';
  import { t } from 'svelte-i18n';
  import { dhtService } from '$lib/dht';
  import { paymentService } from '$lib/services/paymentService';
  import type { FileMetadata } from '$lib/dht';
  import SearchResultCard from './SearchResultCard.svelte';
  import SearchResultCardSkeleton from './SearchResultCardSkeleton.svelte';
  import { dhtSearchHistory, type SearchHistoryEntry, type SearchStatus } from '$lib/stores/searchHistory';
  import PeerSelectionModal, { type PeerInfo } from './PeerSelectionModal.svelte';
  import PeerSelectionService from '$lib/services/peerSelectionService';
  import { costFromPricePerMb, pickLowestPricePeer } from '$lib/utils/pricing';

  type ToastType = 'success' | 'error' | 'info' | 'warning';
  type ToastPayload = { message: string; type?: ToastType; duration?: number; };

  // Progressive search state
  interface SeederInfo {
    index: number;
    peerId: string;
    walletAddress?: string;
    pricePerMb?: number;
    protocols?: string[];
    protocolDetails?: any;
    hasGeneralInfo: boolean;
    hasFileInfo: boolean;
  }

  interface ProgressiveSearchState {
    status: 'idle' | 'searching' | 'complete' | 'timeout';
    fileHash: string | null;
    basicMetadata: {
      fileName: string;
      fileSize: number;
      createdAt: number;
      mimeType?: string;
    } | null;
    providers: string[];
    seeders: SeederInfo[];
  }

  const dispatch = createEventDispatcher<{ download: FileMetadata; message: ToastPayload }>();
  const tr = (key: string, params?: Record<string, unknown>) => (get(t) as any)(key, params);

  const DEV = import.meta.env.DEV;

  const SEARCH_TIMEOUT_MS = 10_000;

  let searchHash = $state('');
  let searchMode = $state<'merkle_hash' | 'magnet' | 'torrent' | 'ed2k' | 'ftp'>('merkle_hash');
  let isSearching = $state(false);
  let torrentFileInput = $state<HTMLInputElement>();
  let torrentFileName = $state<string | null>(null);
  let hasSearched = $state(false);
  let latestStatus = $state<SearchStatus>('pending');
  let latestMetadata = $state<FileMetadata | null>(null);
  let searchError = $state<string | null>(null);
  let lastSearchDuration = $state(0);
  let searchStartedAtMs = $state<number | null>(null);
  let searchCancelTimeoutId = $state<ReturnType<typeof setTimeout> | null>(null);
  let currentSearchId = $state(0);

  async function stopActiveSearch() {
    // Invalidate any in-flight async work
    currentSearchId += 1;

    if (searchCancelTimeoutId) {
      clearTimeout(searchCancelTimeoutId);
      searchCancelTimeoutId = null;
    }

    isSearching = false;

    // Stop consuming progressive events; backend may still finish its search.
    await cleanupProgressiveEventListeners();
  }

  export async function cancelSearch() {
    await stopActiveSearch();

    // Freeze the UI in whatever state we currently have.
    if (latestMetadata) {
      progressiveSearchState.status = 'timeout';
    } else {
      progressiveSearchState.status = 'idle';
      latestStatus = 'pending';
    }

    pushMessage('Search cancelled', 'info', 2000);
  }

  export async function handleFileNotFound(fileHash: string) {
    const expectedHash = progressiveSearchState.fileHash ?? searchHash.trim();
    if (!expectedHash || expectedHash !== fileHash) return;

    const startedAt = searchStartedAtMs;
    await stopActiveSearch();

    if (typeof startedAt === 'number') {
      lastSearchDuration = Math.round(performance.now() - startedAt);
    }

    progressiveSearchState.status = 'idle';
    latestMetadata = null;
    latestStatus = 'not_found';
    hasSearched = true;
    searchError = null;

    if (searchMode === 'merkle_hash' && activeHistoryId) {
      dhtSearchHistory.updateEntry(activeHistoryId, {
        status: 'not_found',
        errorMessage: tr('download.search.status.notFoundDetail'),
        elapsedMs: lastSearchDuration > 0 ? lastSearchDuration : undefined,
      });
    }

    pushMessage(tr('download.search.status.notFoundNotification'), 'warning', 6000);
  }
  let historyEntries = $state<SearchHistoryEntry[]>([]);
  let activeHistoryId = $state<string | null>(null);
  let showHistoryDropdown = $state(false);

  // Protocol selection state
  let availableProtocols = $state<Array<{id: string, name: string, description: string, available: boolean}>>([]);

  // Peer selection modal state
  let showPeerSelectionModal = $state(false);

  // Debug peer selection modal state
  $effect(() => {
    if (!DEV) return;
    console.log('[DownloadSearchSection] showPeerSelectionModal:', showPeerSelectionModal);
  });
  let selectedFile = $state<FileMetadata | null>(null);
  let selectedFileIsSeeding = $state(false);
  let peerSelectionMode = $state<'auto' | 'manual'>('auto');
  let selectedProtocol = $state<'http' | 'webrtc' | 'bitswap' | 'bittorrent' | 'ed2k' | 'ftp'>('http');
  let availablePeers = $state<PeerInfo[]>([]);
  let autoSelectionInfo = $state<Array<{peerId: string; score: number; metrics: any}> | null>(null);

  // Torrent confirmation state
  let pendingTorrentIdentifier = $state<string | null>(null);
  let pendingTorrentBytes = $state<number[] | null>(null);
  let pendingTorrentType = $state<'magnet' | 'file' | null>(null);

  // Progressive search state
  let progressiveSearchState = $state<ProgressiveSearchState>({
    status: 'idle',
    fileHash: null,
    basicMetadata: null,
    providers: [],
    seeders: []
  });

  function syncAvailablePeerOffer(peerId: string, data: { walletAddress?: string; pricePerMb?: number }) {
    if (!availablePeers || availablePeers.length === 0) return;
    const idx = availablePeers.findIndex((p) => p.peerId === peerId);
    if (idx < 0) return;

    const prev = availablePeers[idx];
    const next: PeerInfo = {
      ...prev,
      walletAddress: data.walletAddress ?? prev.walletAddress,
      price_per_mb:
        typeof data.pricePerMb === 'number' && Number.isFinite(data.pricePerMb)
          ? data.pricePerMb
          : prev.price_per_mb,
      offerSource:
        typeof data.pricePerMb === 'number' && Number.isFinite(data.pricePerMb)
          ? 'seeder'
          : prev.offerSource,
    };

    if (next === prev) return;
    availablePeers = [...availablePeers.slice(0, idx), next, ...availablePeers.slice(idx + 1)];
  }

  // Event listener cleanup functions
  let eventUnlisteners = $state<Array<() => void>>([]);

  const unsubscribe = dhtSearchHistory.subscribe((entries) => {
    historyEntries = entries;
    // if (!activeHistoryId && entries.length > 0) {
    //   activeHistoryId = entries[0].id;
    //   latestStatus = entries[0].status;
    //   latestMetadata = entries[0].metadata ?? null;
    //   searchError = entries[0].errorMessage ?? null;
    //   hasSearched = entries.length > 0;
    // }
    if (entries.length > 0) {
      // 1. Always set the active ID from the most recent entry for the history dropdown.
      activeHistoryId = entries[0].id;

      // 2. Control the main UI state based on whether a search has been initiated in this session.
      if (!hasSearched) {
        // If it's a fresh load (hasSearched is false):
        // Keep the input clear, and the result panel empty.
        searchHash = '';
        latestStatus = 'pending';
        latestMetadata = null;
        searchError = null;
      } else {
        // If the user has searched in this session, ensure the current search results are displayed.
        const entry = entries.find(e => e.id === activeHistoryId) || entries[0];
        if (entry) {
          latestStatus = entry.status;
          latestMetadata = entry.metadata ?? null;
          searchError = entry.errorMessage ?? null;
          searchHash = entry.hash;
        }
      }
    } else {
      activeHistoryId = null;
      // On empty history, ensure the main state is also reset.
      if (!hasSearched) {
        searchHash = '';
        latestStatus = 'pending';
        latestMetadata = null;
        searchError = null;
      }
    }
  });

  onMount(() => {
    document.addEventListener('click', handleClickOutside);
    if (DEV) {
      console.log('[DownloadSearchSection] mounted', { searchMode, isSearching, hasSearched, latestStatus });
    }
  });

  onDestroy(() => {
    document.removeEventListener('click', handleClickOutside);
    unsubscribe();
    cleanupProgressiveEventListeners();
  });

  function pushMessage(message: string, type: ToastType = 'info', duration = 4000) {
    dispatch('message', { message, type, duration });
  }

  function clearSearch() {
    searchHash = '';
    torrentFileName = null;
  }

  function handleTorrentFileSelect(event: Event) {
    const target = event.target as HTMLInputElement
    const file = target.files?.[0]
    if (file && file.name.endsWith('.torrent')) {
      // For Tauri, we'll handle this differently in the download function
      torrentFileName = file.name
    } else {
      torrentFileName = null
      pushMessage('Please select a valid .torrent file', 'warning')
    }
  }

  function hydrateFromEntry(entry: SearchHistoryEntry | undefined) {
    if (!entry) {
      latestStatus = 'pending';
      latestMetadata = null;
      searchError = null;
      return;
    }

    latestStatus = entry.status;
    latestMetadata = entry.metadata ?? null;
    searchError = entry.errorMessage ?? null;
    hasSearched = true;
    searchHash = entry.hash;
    lastSearchDuration = entry.elapsedMs ?? 0;
  }

  // Setup progressive search event listeners
  async function setupProgressiveEventListeners() {
    // Clean up any existing listeners
    await cleanupProgressiveEventListeners();

    const unlisteners: Array<() => void> = [];

    unlisteners.push(await listen('search_started', (event: any) => {
      console.log('🔍 Search started:', event.payload);
      progressiveSearchState.status = 'searching';
      pushMessage('Searching for file...', 'info', 2000);
    }));

    unlisteners.push(await listen('dht_metadata_found', (event: any) => {
      const { fileHash, fileName, fileSize, createdAt, mimeType } = event.payload;
      progressiveSearchState.basicMetadata = { fileName, fileSize, createdAt, mimeType };

      // Immediately populate latestMetadata with basic info so UI updates instantly
      latestMetadata = {
        merkleRoot: fileHash,
        fileHash: fileHash,
        fileName: fileName,
        fileSize: fileSize,
        seeders: progressiveSearchState.providers.length > 0 ? progressiveSearchState.providers : [],
        createdAt: createdAt,
        mimeType: mimeType,
        isEncrypted: false,
        encryptionMethod: undefined,
        keyFingerprint: undefined,
        cids: undefined,
        isRoot: true,
        downloadPath: undefined,
        price: 0,
        uploaderAddress: undefined,
        httpSources: undefined,
        ftpSources: undefined,
        ed2kSources: undefined,
        infoHash: undefined
      };

      latestStatus = 'found';
      hasSearched = true;

      console.log('✅ Basic metadata found and displayed:', fileName);
      pushMessage(`Found file: ${fileName}`, 'success', 3000);
    }));

    // Listen for full file metadata (includes CIDs, HTTP sources, etc.)
    // Backend emits this as 'found_file' event with complete FileMetadata
    unlisteners.push(await listen('found_file', (event: any) => {
      const metadata = event.payload as FileMetadata;
      console.log('📦 Full metadata discovered via found_file event:', metadata);
      console.log('📦 Metadata has CIDs:', metadata.cids?.length || 0);
      console.log('📦 Metadata has HTTP sources:', metadata.httpSources?.length || 0);
      console.log('📦 Metadata has seeders:', metadata.seeders?.length || 0);

      // Store full metadata directly instead of building it from basic info
      // This ensures we capture CIDs, HTTP sources, and other protocol-specific data
      latestMetadata = {
        ...metadata,
        fileHash: metadata.merkleRoot || metadata.fileHash,
        // Keep seeders from providers list (more up-to-date)
        seeders: progressiveSearchState.providers.length > 0 ? progressiveSearchState.providers : metadata.seeders
      };

      console.log('✅ Stored full metadata with CIDs:', latestMetadata.cids?.length || 0);
      console.log('✅ Stored full metadata with seeders:', latestMetadata.seeders?.length || 0);
    }));

    unlisteners.push(await listen('providers_found', (event: any) => {
      const { providers, count } = event.payload;

      // Only update if we have more providers than before (defensive against duplicate/stale events)
      if (count > progressiveSearchState.providers.length) {
        progressiveSearchState.providers = providers;

        // Initialize/update seeder slots (preserve previously loaded info by peerId)
        const prevByPeer = new Map(progressiveSearchState.seeders.map((s) => [s.peerId, s] as const));
        progressiveSearchState.seeders = providers.map((peerId: string, index: number) => {
          const prev = prevByPeer.get(peerId);
          if (prev) return { ...prev, index };
          return {
            index,
            peerId,
            hasGeneralInfo: false,
            hasFileInfo: false
          };
        });

        // If basic metadata is already shown, update seeders immediately (don't wait for seeder info)
        if (latestMetadata && progressiveSearchState.status === 'searching') {
          latestMetadata = {
            ...latestMetadata,
            seeders: providers,
          };
        }

        console.log(`📡 Found ${count} providers:`, providers);
        console.log('📡 Progressive state providers:', progressiveSearchState.providers);
        pushMessage(`Found ${count} seeders`, 'info', 2000);
      } else {
        console.log(`⏭️ Ignoring duplicate/stale providers_found event (current: ${progressiveSearchState.providers.length}, new: ${count})`);
      }
    }));

    unlisteners.push(await listen('seeder_general_info', (event: any) => {
      const { seederIndex, walletAddress, defaultPricePerMb } = event.payload;

      const seeder = progressiveSearchState.seeders[seederIndex];
      if (seeder) {
        seeder.walletAddress = walletAddress;
        seeder.pricePerMb = defaultPricePerMb;
        seeder.hasGeneralInfo = true;

        syncAvailablePeerOffer(seeder.peerId, { walletAddress, pricePerMb: defaultPricePerMb });
      }

      if (DEV) console.log(`[DownloadSearchSection] seeder_general_info #${seederIndex}`, { walletAddress });
    }));

    unlisteners.push(await listen('seeder_file_info', (event: any) => {
      const { seederIndex, pricePerMb, supportedProtocols, protocolDetails } = event.payload;

      const seeder = progressiveSearchState.seeders[seederIndex];
      if (seeder) {
        if (pricePerMb !== null) {
          seeder.pricePerMb = pricePerMb;
        }
        seeder.protocols = supportedProtocols;
        seeder.protocolDetails = protocolDetails;
        seeder.hasFileInfo = true;

        syncAvailablePeerOffer(seeder.peerId, { pricePerMb: pricePerMb ?? undefined });
      }

      if (DEV) console.log(`[DownloadSearchSection] seeder_file_info #${seederIndex}`);

      // If the DHT record didn't include Bitswap CIDs (common when metadata is built progressively),
      // hydrate them from seeder protocol details so downloads can start.
      if (latestMetadata && protocolDetails) {
        const next = { ...latestMetadata } as any;
        let changed = false;

        // Bitswap requires CIDs; seed them as soon as we learn them.
        if (
          (!next.cids || next.cids.length === 0) &&
          Array.isArray(protocolDetails.cids) &&
          protocolDetails.cids.length > 0
        ) {
          next.cids = protocolDetails.cids;
          changed = true;
          // Default to root=true when we only have a root CID list.
          if (typeof next.isRoot !== 'boolean') {
            next.isRoot = true;
            changed = true;
          }
        }

        // Fill in other protocol-specific fields opportunistically.
        if (!next.httpSources && Array.isArray(protocolDetails.httpSources) && protocolDetails.httpSources.length > 0) {
          next.httpSources = protocolDetails.httpSources;
          changed = true;
        }
        if (!next.ftpSources && Array.isArray(protocolDetails.ftpSources) && protocolDetails.ftpSources.length > 0) {
          next.ftpSources = protocolDetails.ftpSources;
          changed = true;
        }
        if (!next.ed2kSources && Array.isArray(protocolDetails.ed2kSources) && protocolDetails.ed2kSources.length > 0) {
          next.ed2kSources = protocolDetails.ed2kSources;
          changed = true;
        }
        if (!next.infoHash && typeof protocolDetails.infoHash === 'string' && protocolDetails.infoHash.trim().length > 0) {
          next.infoHash = protocolDetails.infoHash;
          changed = true;
        }
        if (!next.trackers && Array.isArray(protocolDetails.trackers) && protocolDetails.trackers.length > 0) {
          next.trackers = protocolDetails.trackers;
          changed = true;
        }

        if (changed) {
          latestMetadata = next;
          console.log('✅ Hydrated metadata from seeder_file_info:', {
            cids: next.cids?.length || 0,
            httpSources: next.httpSources?.length || 0,
            ftpSources: next.ftpSources?.length || 0,
            ed2kSources: next.ed2kSources?.length || 0,
            infoHash: next.infoHash ? 'set' : 'unset'
          });
        }
      }
    }));

    unlisteners.push(await listen('search_complete', (event: any) => {
      const { totalSeeders, durationMs } = event.payload;
      progressiveSearchState.status = 'complete';
      if (typeof durationMs === 'number' && Number.isFinite(durationMs)) {
        lastSearchDuration = durationMs;
      } else if (typeof searchStartedAtMs === 'number') {
        lastSearchDuration = Math.round(performance.now() - searchStartedAtMs);
      }

      console.log(`✅ Search complete: ${totalSeeders} seeders in ${durationMs}ms`);
      pushMessage(`Search complete! Found ${totalSeeders} seeders`, 'success');

      // Build final metadata from progressive state
      if (progressiveSearchState.basicMetadata || latestMetadata) {
        buildFinalMetadata();
      } else {
        console.warn('⚠️ Search complete but no metadata available');
      }
    }));

    unlisteners.push(await listen('search_timeout', (event: any) => {
      const { partialSeeders, missingCount } = event.payload;
      progressiveSearchState.status = 'timeout';
      if (typeof searchStartedAtMs === 'number') {
        lastSearchDuration = Math.round(performance.now() - searchStartedAtMs);
      }

      console.warn(`⚠️ Search timeout: ${partialSeeders} complete, ${missingCount} missing`);
      pushMessage(`Partial results: ${partialSeeders} seeders available`, 'warning');

      // Build metadata with partial results
      if (progressiveSearchState.basicMetadata || latestMetadata) {
        buildFinalMetadata();
      } else {
        console.warn('⚠️ Search timeout but no metadata available');
      }
    }));

    eventUnlisteners = unlisteners;
  }

  // Clean up progressive event listeners
  async function cleanupProgressiveEventListeners() {
    for (const unlisten of eventUnlisteners) {
      unlisten();
    }
    eventUnlisteners = [];
  }

  // Build final metadata from progressive state
  function buildFinalMetadata() {
    if (!progressiveSearchState.basicMetadata) return;

    console.log('🔧 Building final metadata from progressive state');
    console.log('🔧 Progressive providers:', progressiveSearchState.providers);
    console.log('🔧 Progressive seeders:', progressiveSearchState.seeders);
    console.log('🔧 Existing latestMetadata:', latestMetadata);

    // If we already have full metadata from file_discovered event, just update seeders
    if (latestMetadata && latestMetadata.merkleRoot === progressiveSearchState.fileHash) {
      console.log('✅ Using existing full metadata from file_discovered event');
      latestMetadata = {
        ...latestMetadata,
        seeders: progressiveSearchState.providers.length > 0 ? progressiveSearchState.providers : latestMetadata.seeders
      };
      console.log('✅ Updated seeders from providers:', latestMetadata.seeders);
    } else {
      // Fallback: Build minimal metadata from basicMetadata
      console.log('⚠️ No full metadata from file_discovered, building minimal metadata');
      latestMetadata = {
        merkleRoot: progressiveSearchState.fileHash || '',
        fileHash: progressiveSearchState.fileHash || '',
        fileName: progressiveSearchState.basicMetadata.fileName,
        fileSize: progressiveSearchState.basicMetadata.fileSize,
        createdAt: progressiveSearchState.basicMetadata.createdAt,
        mimeType: progressiveSearchState.basicMetadata.mimeType,
        seeders: progressiveSearchState.providers,
        isEncrypted: false,
        isRoot: true,
        price: 0
      };
    }

    // Final fallback: if the metadata still doesn't include Bitswap CIDs, attempt to hydrate them
    // from any seeder_file_info protocolDetails we received.
    if (latestMetadata && (!latestMetadata.cids || latestMetadata.cids.length === 0)) {
      const firstWithCids = progressiveSearchState.seeders.find((s) =>
        Array.isArray((s as any).protocolDetails?.cids) && (s as any).protocolDetails.cids.length > 0
      ) as any;

      if (firstWithCids?.protocolDetails?.cids?.length) {
        latestMetadata = {
          ...latestMetadata,
          cids: firstWithCids.protocolDetails.cids,
          isRoot: typeof latestMetadata.isRoot === 'boolean' ? latestMetadata.isRoot : true,
        };
        console.log('✅ Hydrated missing CIDs from progressive seeder_file_info');
      }
    }

    console.log('✅ Final metadata built with seeders:', latestMetadata.seeders);
    console.log('✅ Final metadata CIDs:', latestMetadata.cids?.length || 0);
    console.log('✅ Full metadata:', latestMetadata);

    latestStatus = 'found';
    isSearching = false;
  }

  async function searchForFile() {
    console.log('🔍 searchForFile() called with searchMode:', searchMode, 'searchHash:', searchHash, 'isSearching:', isSearching)
    if (isSearching) {
      console.warn('⚠️ Search already in progress, ignoring duplicate call')
      return
    }

    currentSearchId += 1;
    const searchId = currentSearchId;

    isSearching = true
    console.log('✅ Search started, isSearching now:', isSearching)

    // Handle BitTorrent downloads - show confirmation instead of immediately downloading
    if (searchMode === 'magnet' || searchMode === 'torrent' || searchMode === 'ed2k' || searchMode === 'ftp') {
      console.log('✅ Entering magnet/torrent/ed2k/ftp path')
      let identifier: string | null = null

      if (searchMode === 'magnet') {
        identifier = searchHash.trim()
        if (!identifier) {
          pushMessage('Please enter a magnet link', 'warning')
          isSearching = false
          return
        }

        // For magnet links, extract info_hash and search DHT directly
        console.log('🔍 Parsing magnet link:', identifier)
        const urlParams = new URLSearchParams(identifier.split('?')[1])
        const infoHash = urlParams.get('xt')?.replace('urn:btih:', '').toLowerCase()
        console.log('🔍 Extracted info_hash (normalized to lowercase):', infoHash)
        if (infoHash) {
          try {
            console.log('🔍 Searching DHT by info_hash:', infoHash)
            // Tauri converts parameters to camelCase, so we use infoHash here
            const params = { infoHash }
            console.log('🔍 Calling search_by_infohash with params:', JSON.stringify(params))
            // Search DHT by info_hash (uses two-step lookup: info_hash -> merkle_root -> metadata)
            const metadata = await invoke('search_by_infohash', params) as FileMetadata | null
            console.log('🔍 DHT search result:', metadata)
            if (metadata) {
              // Found the file! Show it instead of the placeholder
              metadata.fileHash = metadata.merkleRoot || ""
              latestMetadata = metadata
              latestStatus = 'found'
              hasSearched = true
              pushMessage(`Found file: ${metadata.fileName}`, 'success')
              isSearching = false
              return
            } else {
              console.log('⚠️ No metadata found for info_hash:', infoHash)
            }
          } catch (error) {
            console.error('❌ DHT search error:', error)
            console.log('Falling back to magnet download')
          }
        } else {
          console.log('⚠️ Could not extract info_hash from magnet link')
        }

        // If not found in DHT or no info_hash, proceed with magnet download
      } else if (searchMode === 'torrent') {
        if (!torrentFileName) {
          pushMessage('Please select a .torrent file', 'warning')
          isSearching = false
          return
        }
        // Use the file input to get the actual file
        const file = torrentFileInput?.files?.[0]
        if (file) {
          // Try to parse torrent file and search for it first
          try {
            // For now, we'll search using a placeholder - ideally we'd parse the torrent
            // to extract the info hash and search DHT. For simplicity, fall back to placeholder.
          identifier = torrentFileName
          } catch (error) {
            console.log('Failed to parse torrent file:', error)
            identifier = torrentFileName
          }
        } else {
          pushMessage('Please select a .torrent file', 'warning')
          return
        }
      } else if (searchMode === 'ed2k') {
        identifier = searchHash.trim()
        if (!identifier) {
          pushMessage('Please enter an ED2K link', 'warning')
          isSearching = false
          return
        }
        // Basic ED2K link validation
        if (!identifier.startsWith('ed2k://')) {
          pushMessage('Please enter a valid ED2K link starting with ed2k://', 'warning')
          isSearching = false
          return
        }

        // For ED2K links, extract hash and search DHT first
        const parts = identifier.split('|')
        if (parts.length >= 5) {
          const ed2kHash = parts[4]
          try {
            // Search DHT using the ED2K hash as the key (results come via events)
            await dhtService.searchFileMetadata(ed2kHash, SEARCH_TIMEOUT_MS)
            // The found_file event will populate latestMetadata if found
            // If not found, we'll proceed with ED2K download below
            console.log('Triggered DHT search for ED2K hash:', ed2kHash)
          } catch (error) {
            console.log('DHT search failed for ED2K hash:', error)
          }
        }
      } else if (searchMode === 'ftp') {
        identifier = searchHash.trim()
        if (!identifier) {
          pushMessage('Please enter an FTP URL', 'warning')
          isSearching = false
          return
        }
        // Basic FTP URL validation
        if (!identifier.startsWith('ftp://') && !identifier.startsWith('ftps://')) {
          pushMessage('Please enter a valid FTP URL starting with ftp:// or ftps://', 'warning')
          isSearching = false
          return
        }

        // Handle FTP URL - extract hash and search DHT for real metadata
        try {
          const ftpUrl = new URL(identifier)
          const pathSegments = ftpUrl.pathname.split('/').filter(s => s.length > 0)
          let fileName = pathSegments.length > 0 ? decodeURIComponent(pathSegments[pathSegments.length - 1]) : 'unknown_file'

          // Extract hash prefix if present (format: {64-char-hash}_{original_filename})
          let extractedHash = ''
          if (fileName.length > 65 && fileName.charAt(64) === '_') {
            // Check if first 64 chars look like a hex hash
            const potentialHash = fileName.substring(0, 64)
            if (/^[a-f0-9]{64}$/i.test(potentialHash)) {
              extractedHash = potentialHash
              fileName = fileName.substring(65) // Remove hash prefix and underscore
            }
          }

          // If we have a hash, search DHT for real metadata (results come via events)
          if (extractedHash) {
            try {
              await dhtService.searchFileMetadata(extractedHash, SEARCH_TIMEOUT_MS)
              // The found_file event will populate latestMetadata if found
              // For FTP, we'll enhance it with FTP sources in the event handler or use fallback below
              console.log('Triggered DHT search for FTP hash:', extractedHash)
            } catch (error) {
              console.log('DHT search failed for FTP hash, falling back to basic FTP metadata:', error)
            }
          }

          // Fallback: Create basic metadata with FTP source if no hash found or DHT search failed
          latestMetadata = {
            merkleRoot: extractedHash || '',
            fileHash: extractedHash || '',
            fileName: fileName,
            fileSize: 0, // Unknown for FTP URLs without metadata
            seeders: [],
            createdAt: Date.now() / 1000,
            mimeType: undefined,
            isEncrypted: false,
            encryptionMethod: undefined,
            keyFingerprint: undefined,
            cids: undefined,
            isRoot: true,
            downloadPath: undefined,
            price: 0,
            uploaderAddress: undefined,
            httpSources: undefined,
            ftpSources: [{
              url: identifier,
              username: ftpUrl.username || undefined,
              password: ftpUrl.password || undefined,
              supportsResume: true, // Assume true for user-provided FTP URLs
              isAvailable: true
            }]
          }

          latestStatus = 'found'
          hasSearched = true
          isSearching = false
          const fallbackMsg = extractedHash ? `FTP file ready to download: ${fileName} (metadata not found)` : `FTP file ready to download: ${fileName}`
          pushMessage(fallbackMsg, 'success')
        } catch (error) {
          console.error("Failed to parse FTP URL:", error)
          pushMessage(`Invalid FTP URL: ${String(error)}`, 'error')
          isSearching = false
        }
        return
      }

      if (identifier) {
        try {
          
          // Store the pending torrent info for confirmation
          if (searchMode === 'torrent') {
            const file = torrentFileInput?.files?.[0]
            if (file) {
              const arrayBuffer = await file.arrayBuffer()
              const bytes = new Uint8Array(arrayBuffer)
              pendingTorrentBytes = Array.from(bytes)
              pendingTorrentType = 'file'
              pendingTorrentIdentifier = torrentFileName
            }
          } else {
            // For magnet links
            pendingTorrentIdentifier = identifier
            pendingTorrentType = 'magnet'
            pendingTorrentBytes = null
          }
          
          // Show confirmation (metadata display) instead of immediately downloading
          latestMetadata = {
            merkleRoot: '', // No merkle root for torrents
            fileHash: '',
            fileName: pendingTorrentType === 'magnet' ? 'Magnet Link Download' : (torrentFileName || 'Torrent Download'),
            fileSize: 0, // Unknown until torrent metadata is fetched
            seeders: [],
            createdAt: Date.now() / 1000,
            mimeType: undefined,
            isEncrypted: false,
            encryptionMethod: undefined,
            keyFingerprint: undefined,
            cids: undefined,
            isRoot: true,
            downloadPath: undefined,
            price: 0,
            uploaderAddress: undefined,
            httpSources: undefined,
          }
          
          latestStatus = 'found'
          hasSearched = true
          isSearching = false
          pushMessage(`${pendingTorrentType === 'magnet' ? 'Magnet link' : 'Torrent file'} ready to download`, 'success')
        } catch (error) {
          console.error("Failed to prepare torrent:", error)
          pushMessage(`Failed to prepare download: ${String(error)}`, 'error')
          isSearching = false
        }
      }
      return
    }

    // Original DHT search logic for merkle_hash
    const trimmed = searchHash.trim();
    if (!trimmed) {
      const message = searchMode === 'merkle_hash' ? tr('download.notifications.enterHash') :
                     searchMode === 'magnet' ? 'Please enter a magnet link' :
                     searchMode === 'ed2k' ? 'Please enter an ED2K link' :
                     searchMode === 'ftp' ? 'Please enter an FTP URL' :
                     'Please enter a search term';
      pushMessage(message, 'warning');
      isSearching = false; // Reset searching state
      return;
    }

    hasSearched = true;
    latestMetadata = null;
    latestStatus = 'pending';
    searchError = null;

    const startedAt = performance.now();
    searchStartedAtMs = startedAt;

    try {
      // Setup progressive event listeners
      await setupProgressiveEventListeners();

      // Reset progressive search state
      progressiveSearchState = {
        status: 'searching',
        fileHash: trimmed,
        basicMetadata: null,
        providers: [],
        seeders: []
      };

      // Create history entry
      const entry = dhtSearchHistory.addPending(trimmed);
      activeHistoryId = entry.id;

      // Initiate progressive search (non-blocking)
      void dhtService.searchFileMetadata(trimmed, SEARCH_TIMEOUT_MS).catch((error) => {
        // Ignore late failures from a canceled/stale search
        if (searchId !== currentSearchId) return;

        const message = error instanceof Error ? error.message : tr('download.search.status.unknownError');
        const elapsed = Math.round(performance.now() - startedAt);
        lastSearchDuration = elapsed;
        latestStatus = 'error';
        searchError = message;

        if (searchMode === 'merkle_hash' && activeHistoryId) {
          dhtSearchHistory.updateEntry(activeHistoryId, {
            status: 'error',
            errorMessage: message,
            elapsedMs: elapsed,
          });
        }

        console.error('Search failed:', error);
        pushMessage(`${tr('download.search.status.errorNotification')}: ${message}`, 'error', 6000);

        isSearching = false;
        void cleanupProgressiveEventListeners();
      });

      // Note: The search will now progress via events
      // The final metadata will be built in buildFinalMetadata() when search_complete or search_timeout fires

      // Fallback timeout: reset isSearching after 15 seconds if no completion event received
      if (searchCancelTimeoutId) {
        clearTimeout(searchCancelTimeoutId);
      }
      searchCancelTimeoutId = setTimeout(() => {
        if (searchId !== currentSearchId) return;
        if (isSearching && progressiveSearchState.status === 'searching') {
          console.warn('⚠️ Frontend timeout - no completion event received from backend');
          isSearching = false;
          cleanupProgressiveEventListeners();

          if (progressiveSearchState.basicMetadata) {
            // Build metadata with whatever we have
            buildFinalMetadata();
            pushMessage('Search completed with partial results', 'warning');
          } else {
            latestStatus = 'error';
            searchError = 'Search timeout - no response from network';
            pushMessage('Search timeout - no response from network', 'error');
          }
        }
      }, 15000);
    } catch (error) {
      const message = error instanceof Error ? error.message : tr('download.search.status.unknownError');
      const elapsed = Math.round(performance.now() - startedAt);
      lastSearchDuration = elapsed;
      latestStatus = 'error';
      searchError = message;

      if (searchMode === 'merkle_hash' && activeHistoryId) {
        dhtSearchHistory.updateEntry(activeHistoryId, {
          status: 'error',
          errorMessage: message,
          elapsedMs: elapsed,
        });
      }

      console.error('Search failed:', error);
      pushMessage(`${tr('download.search.status.errorNotification')}: ${message}`, 'error', 6000);

      isSearching = false;
      await cleanupProgressiveEventListeners();
    }
  }

  function clearHistory() {
    dhtSearchHistory.clear();
    historyEntries = [];
    activeHistoryId = null;
    latestMetadata = null;
    latestStatus = 'pending';
    searchError = null;
    hasSearched = false;
  }

  function handleCopy(_event: CustomEvent<string>) {
    // Silently copy without toast notification
  }


  function statusIcon(status: string) {
    switch (status) {
      case 'found':
        return CheckCircle2;
      case 'error':
        return AlertCircle;
      default:
        return Search;
    }
  }

  function statusClass(status: string) {
    switch (status) {
      case 'found':
        return 'text-emerald-600';
      case 'error':
        return 'text-red-600';
      case 'not_found':
        return 'text-amber-600';
      default:
        return 'text-muted-foreground';
    }
  }

  function toggleHistoryDropdown() {
    showHistoryDropdown = !showHistoryDropdown;
  }

  function selectHistoryEntry(entry: SearchHistoryEntry) {
    searchHash = entry.hash;
    activeHistoryId = entry.id;
    hydrateFromEntry(entry);
    showHistoryDropdown = false;
  }

  function handleClickOutside(event: MouseEvent) {
    const target = event.target as HTMLElement;
    if (!target.closest('.search-input-container')) {
      showHistoryDropdown = false;
    }
  }

  // Helper function to determine available protocols for a file
  // Uses seeder-reported protocols from progressive search state
  function getAvailableProtocols(metadata: FileMetadata): Array<{id: string, name: string, description: string, available: boolean}> {
    console.log('🔍 getAvailableProtocols called with metadata:', metadata);
    console.log('🔍 progressiveSearchState.seeders:', progressiveSearchState.seeders);

    // Collect all protocols reported by seeders
    const reportedProtocols = new Set<string>();
    for (const seeder of progressiveSearchState.seeders) {
      if (seeder.protocols && seeder.protocols.length > 0) {
        for (const protocol of seeder.protocols) {
          reportedProtocols.add(protocol.toLowerCase());
        }
      }
    }

    console.log('🔍 Seeder-reported protocols:', Array.from(reportedProtocols));

    // Also check metadata fields as fallback for backwards compatibility
    const hasInfoHash = !!metadata.infoHash;
    const hasHttpSources = !!(metadata.httpSources && metadata.httpSources.length > 0);
    const hasFtpSources = !!(metadata.ftpSources && metadata.ftpSources.length > 0);
    const hasEd2kSources = !!(metadata.ed2kSources && metadata.ed2kSources.length > 0);
    const hasCids = !!(metadata.cids && metadata.cids.length > 0);

    console.log('🔍 Metadata fields:', { hasInfoHash, hasHttpSources, hasFtpSources, hasEd2kSources, hasCids });

    // Determine availability: use seeder-reported protocols if available, otherwise fall back to metadata detection
    const isBitSwapAvailable = reportedProtocols.has('bitswap') || (reportedProtocols.size === 0 && hasCids);
    const isWebRTCAvailable = reportedProtocols.has('webrtc') || (reportedProtocols.size === 0 && metadata.seeders && metadata.seeders.length > 0 && !hasInfoHash && !hasHttpSources && !hasFtpSources && !hasEd2kSources);
    const isBitTorrentAvailable = reportedProtocols.has('bittorrent') || (reportedProtocols.size === 0 && hasInfoHash);
    const isHttpAvailable = reportedProtocols.has('http') || (reportedProtocols.size === 0 && hasHttpSources);
    const isEd2kAvailable = reportedProtocols.has('ed2k') || (reportedProtocols.size === 0 && hasEd2kSources);
    const isFtpAvailable = reportedProtocols.has('ftp') || (reportedProtocols.size === 0 && hasFtpSources);

    console.log('🔍 Protocol availability:', {
      bitswap: isBitSwapAvailable,
      webrtc: isWebRTCAvailable,
      bittorrent: isBitTorrentAvailable,
      http: isHttpAvailable,
      ed2k: isEd2kAvailable,
      ftp: isFtpAvailable
    });

    return [
      {
        id: 'bitswap',
        name: 'BitSwap',
        description: 'Content-addressed P2P (IPFS)',
        available: isBitSwapAvailable
      },
      {
        id: 'webrtc',
        name: 'WebRTC',
        description: 'Peer-to-peer via WebRTC',
        available: isWebRTCAvailable
      },
      {
        id: 'http',
        name: 'HTTP',
        description: 'Direct HTTP download',
        available: isHttpAvailable
      },
      {
        id: 'bittorrent',
        name: 'BitTorrent',
        description: 'BitTorrent protocol',
        available: isBitTorrentAvailable
      },
      {
        id: 'ed2k',
        name: 'ED2K',
        description: 'ED2K protocol',
        available: isEd2kAvailable
      },
      {
        id: 'ftp',
        name: 'FTP',
        description: 'FTP protocol',
        available: isFtpAvailable
      }
    ];
  }

  // Check if current user is seeding this file
  function checkIfSeeding(metadata: FileMetadata): boolean {
    try {
      const currentPeerId = dhtService.getPeerId();
      return currentPeerId ? metadata.seeders?.includes(currentPeerId) || false : false;
    } catch (error) {
      console.warn('Failed to check seeding status:', error);
      return false;
    }
  }

  // Handle file download - show protocol selection modal first if multiple protocols available
  async function handleFileDownload(metadata: FileMetadata) {
    console.log('🔍 DEBUG: handleFileDownload called in DownloadSearchSection');
    console.log('🔍 DEBUG: metadata received:', metadata);
    console.log('🔍 DEBUG: metadata.seeders:', metadata.seeders);
    console.log('🔍 DEBUG: pendingTorrentType:', pendingTorrentType);
    console.log('🔍 DEBUG: pendingTorrentIdentifier:', pendingTorrentIdentifier);

    // Check if user is seeding this file
    selectedFileIsSeeding = checkIfSeeding(metadata);
    console.log('🔍 DEBUG: selectedFileIsSeeding:', selectedFileIsSeeding);

    // Handle BitTorrent downloads (magnet/torrent) - skip protocol selection, go directly to peer selection
    if (pendingTorrentType && pendingTorrentIdentifier) {
      console.log('🔍 DEBUG: Handling BitTorrent download - showing peer selection modal');
      selectedFile = metadata;
      selectedProtocol = 'bittorrent';
      showPeerSelectionModal = true;
      return;
    }

    // Get available protocols for this file
    availableProtocols = getAvailableProtocols(metadata);
    const availableProtocolList = availableProtocols.filter(p => p.available);
    console.log('🔍 DEBUG: availableProtocols:', availableProtocols);
    console.log('🔍 DEBUG: availableProtocolList:', availableProtocolList);

    // If no protocols available
    if (availableProtocolList.length === 0) {
      pushMessage('No download protocols available for this file', 'warning');
      return;
    }

    // Select the first available protocol as default (user can change in peer selection modal)
    selectedProtocol = availableProtocolList[0].id as any;
    
    // Go directly to peer selection modal (protocol can be changed there)
    selectedFile = metadata;
    await proceedWithProtocolSelection(metadata, selectedProtocol);
  }

  // Proceed with download using selected protocol
  async function proceedWithProtocolSelection(metadata: FileMetadata, protocolId: string) {
    console.log('🔍 DEBUG: proceedWithProtocolSelection called');
    console.log('🔍 DEBUG: metadata:', metadata);
    console.log('🔍 DEBUG: protocolId:', protocolId);
    console.log('🔍 DEBUG: metadata.seeders:', metadata.seeders);
    
    // Handle HTTP and ED2K direct downloads (no peer selection)
    if (protocolId === 'http' || protocolId === 'ed2k') {
      console.log('🔍 DEBUG: Using direct download for protocol:', protocolId);
      await startDirectDownload(metadata, protocolId);
      return;
    }

    // Handle FTP - show source selection modal
    if (protocolId === 'ftp') {
      if (!metadata.ftpSources || metadata.ftpSources.length === 0) {
        pushMessage('No FTP sources available for this file', 'warning');
        return;
      }

      selectedFile = metadata;
      selectedProtocol = 'ftp';
      
      // Create "peers" from FTP sources
      availablePeers = metadata.ftpSources.map((source, index) => {
        // Extract host from FTP URL
        let host = 'FTP Server';
        try {
          const url = new URL(source.url);
          host = url.hostname;
        } catch {}
        
        return {
          peerId: source.url, // Use URL as the ID
          location: host,
          latency_ms: undefined,
          bandwidth_kbps: undefined,
          reliability_score: source.isAvailable ? 1.0 : 0.0,
          price_per_mb: 0, // FTP is free
          selected: index === 0, // Select first by default
          percentage: index === 0 ? 100 : 0
        };
      });

      showPeerSelectionModal = true;
      return;
    }

    // For P2P protocols (WebRTC, BitSwap, BitTorrent) - need peer selection
    if (protocolId === 'webrtc' || protocolId === 'bitswap' || protocolId === 'bittorrent') {
      // Check if there are any seeders
      if (!metadata.seeders || metadata.seeders.length === 0) {
        pushMessage('No seeders available for this file', 'warning');
        return;
      }

      // Proceed with peer selection for P2P protocols
      await proceedWithPeerSelection(metadata);
    }
  }

  // Start direct download for HTTP/FTP/ED2K protocols
  async function startDirectDownload(metadata: FileMetadata, protocolId: string) {
    try {
      const { invoke } = await import("@tauri-apps/api/core");

      if (protocolId === 'http' && metadata.httpSources && metadata.httpSources.length > 0) {
        await invoke('download_file_http', {
          seeder_url: metadata.httpSources[0],
          merkle_root: metadata.merkleRoot || metadata.fileHash,
          output_path: `./downloads/${metadata.fileName}`,
          peer_id: null
        });
        pushMessage('HTTP download started', 'success');
      } else if (protocolId === 'ftp' && metadata.ftpSources && metadata.ftpSources.length > 0) {
        await invoke('download_ftp', { url: metadata.ftpSources[0].url });
        pushMessage('FTP download started', 'success');
      } else if (protocolId === 'ed2k' && metadata.ed2kSources && metadata.ed2kSources.length > 0) {
        // Construct ED2K file link from source info: ed2k://|file|name|size|hash|/
        const ed2kSource = metadata.ed2kSources[0];
        const ed2kLink = `ed2k://|file|${metadata.fileName}|${metadata.fileSize}|${ed2kSource.file_hash}|/`;
        await invoke('download_ed2k', { link: ed2kLink });
        pushMessage('ED2K download started', 'success');
      } else {
        pushMessage(`No ${protocolId.toUpperCase()} sources available`, 'warning');
      }
    } catch (error) {
      console.error(`Failed to start ${protocolId} download:`, error);
      pushMessage(`Failed to start ${protocolId.toUpperCase()} download: ${String(error)}`, 'error');
    }
  }

  // Proceed with peer selection for P2P protocols
  async function proceedWithPeerSelection(metadata: FileMetadata) {
    if (DEV) {
      console.log('[DownloadSearchSection] proceedWithPeerSelection', {
        fileHash: metadata.fileHash,
        seeders: metadata.seeders?.length ?? 0,
      });
    }

    selectedFile = metadata;
    autoSelectionInfo = null;  // Clear previous auto-selection info

    // Fetch peer metrics for each seeder
    try {
      const allMetrics = await PeerSelectionService.getPeerMetrics();

      // Fallback price for peers without offer info yet.
      let fallbackPerMbPrice = 0.001;
      try {
        fallbackPerMbPrice = await paymentService.getDynamicPricePerMB(1.2);
      } catch (pricingError) {
        console.warn('Failed to get dynamic per MB price, using fallback:', pricingError);
      }

      availablePeers = metadata.seeders.map(seederId => {
        const metrics = allMetrics.find(m => m.peer_id === seederId);
        const seederOffer = progressiveSearchState.seeders.find((s) => s.peerId === seederId);
        const offerPrice = seederOffer?.pricePerMb;
        const pricePerMb = typeof offerPrice === 'number' && Number.isFinite(offerPrice)
          ? offerPrice
          : fallbackPerMbPrice;

        return {
          peerId: seederId,
          latency_ms: metrics?.latency_ms,
          bandwidth_kbps: metrics?.bandwidth_kbps,
          reliability_score: metrics?.reliability_score ?? 0.5,
          price_per_mb: pricePerMb,
          walletAddress: seederOffer?.walletAddress,
          offerSource: typeof offerPrice === 'number' ? 'seeder' : 'fallback',
          selected: true,  // All selected by default
          percentage: Math.round(100 / metadata.seeders.length)  // Equal split
        };
      });

      if (DEV) {
        console.log('[DownloadSearchSection] availablePeers created', availablePeers);
      }

      // If in auto mode, pre-calculate the selection for transparency
      if (peerSelectionMode === 'auto') {
        await calculateAutoSelection(metadata, allMetrics);
      }

      showPeerSelectionModal = true;
    } catch (error) {
      console.error('Failed to fetch peer metrics:', error);
      // Fall back to direct download without peer selection
      pushMessage('Failed to load peer selection, proceeding with default download', 'warning');
      dispatch('download', metadata);
    }
  }

  // Calculate auto-selection for transparency display
  async function calculateAutoSelection(metadata: FileMetadata, allMetrics: any[]) {
    try {
      // Auto-select best peers using backend algorithm
      const autoPeers = await PeerSelectionService.getPeersForParallelDownload(
        metadata.seeders,
        metadata.fileSize,
        3,  // Max 3 peers
        metadata.isEncrypted
      );

      // Get metrics for selected peers
      const selectedMetrics = autoPeers.map(peerId =>
        allMetrics.find(m => m.peer_id === peerId)
      ).filter(m => m !== undefined);

      if (selectedMetrics.length > 0) {
        // Calculate composite scores for each peer
        const peerScores = selectedMetrics.map(m => ({
          peerId: m!.peer_id,
          score: PeerSelectionService.compositeScoreFromMetrics(m!)
        }));

        // Calculate total score
        const totalScore = peerScores.reduce((sum, p) => sum + p.score, 0);

        // Store selection info for transparency display
        autoSelectionInfo = peerScores.map((p, index) => ({
          peerId: p.peerId,
          score: p.score,
          metrics: selectedMetrics[index]!
        }));

        // Update availablePeers with score-weighted percentages
        availablePeers = availablePeers.map(peer => {
          const peerScore = peerScores.find(ps => ps.peerId === peer.peerId);
          if (peerScore) {
            const percentage = Math.round((peerScore.score / totalScore) * 100);
            return {
              ...peer,
              selected: true,
              percentage
            };
          }
          return {
            ...peer,
            selected: false,
            percentage: 0
          };
        });

        // Adjust for rounding to ensure selected peers total 100%
        const selectedPeers = availablePeers.filter(p => p.selected);
        const totalPercentage = selectedPeers.reduce((sum, p) => sum + p.percentage, 0);
        if (totalPercentage !== 100 && selectedPeers.length > 0) {
          selectedPeers[0].percentage += (100 - totalPercentage);
        }
      }
    } catch (error) {
      console.error('Failed to calculate auto-selection:', error);
    }
  }

  // Confirm peer selection and start download
  async function confirmPeerSelection() {
    if (!selectedFile) return;

    // Handle FTP downloads from peer selection modal
    if (selectedProtocol === 'ftp') {
      const selectedSource = availablePeers.find(p => p.selected);
      if (!selectedSource) {
        pushMessage('Please select an FTP source', 'warning');
        return;
      }

      try {
        const { invoke } = await import("@tauri-apps/api/core");
        await invoke('download_ftp', { url: selectedSource.peerId }); // peerId is the FTP URL
        
        showPeerSelectionModal = false;
        selectedFile = null;
        pushMessage('FTP download started', 'success');
      } catch (error) {
        console.error('Failed to start FTP download:', error);
        pushMessage(`Failed to start FTP download: ${String(error)}`, 'error');
      }
      return;
    }

    // Handle direct downloads (HTTP, ED2K) that skip peer selection
    if (selectedProtocol === 'http' || selectedProtocol === 'ed2k') {
      // This shouldn't happen since direct downloads bypass peer selection
      return;
    }

    // Handle BitTorrent downloads from search
    if ((pendingTorrentType && pendingTorrentIdentifier) || selectedProtocol === 'bittorrent') {
      try {
        const { invoke } = await import("@tauri-apps/api/core")

        if (pendingTorrentType === 'file' && pendingTorrentBytes) {
          // For torrent files, pass the file bytes
          await invoke('download_torrent_from_bytes', { bytes: pendingTorrentBytes })
          // We can't easily get the infohash on the frontend from a torrent file
          // The download is already started in the backend, and will be tracked via torrent_event listener
          // So we don't need to dispatch the download event here
        } else if (pendingTorrentType === 'magnet' && pendingTorrentIdentifier) {
          // For magnet links
          await invoke('download', { identifier: pendingTorrentIdentifier })
          // The download is already started in the backend, and will be tracked via torrent_event listener
          // So we don't need to dispatch the download event here
        } else {
          // For BitTorrent from metadata (already on the network)
          await invoke('download', { identifier: selectedFile?.infoHash })
          // The download is already started in the backend, and will be tracked via torrent_event listener
          // So we don't need to dispatch the download event here
        }

        // Note: We don't dispatch the download event for BitTorrent downloads
        // The torrent_event listener in Download.svelte will handle showing the download progress

        // Clear state
        searchHash = ''
        torrentFileName = null
        if (torrentFileInput) torrentFileInput.value = ''
        pendingTorrentIdentifier = null
        pendingTorrentBytes = null
        pendingTorrentType = null

        showPeerSelectionModal = false
        selectedFile = null

        pushMessage('BitTorrent download started', 'success')
      } catch (error) {
        console.error("Failed to start torrent download:", error)
        pushMessage(`Failed to start download: ${String(error)}`, 'error')
      }
      return
    }

    // Get selected peers and their allocations from availablePeers
    const selectedPeers = availablePeers
      .filter(p => p.selected)
      .map(p => p.peerId);

    const peerAllocation = availablePeers
      .filter(p => p.selected)
      .map(p => ({
        peerId: p.peerId,
        percentage: p.percentage
      }));

    // Single payee model: choose lowest offer among selected peers.
    const paymentPeer = pickLowestPricePeer(availablePeers);
    if (!paymentPeer || !paymentPeer.walletAddress) {
      pushMessage('Seeder offer info still loading. Please wait a moment and try again.', 'warning');
      return;
    }

    const estimatedPaymentTotal = costFromPricePerMb({
      bytes: selectedFile.fileSize,
      pricePerMb: paymentPeer.price_per_mb,
    });

    // Log transparency info for auto-selection
    if (peerSelectionMode === 'auto' && autoSelectionInfo) {
      autoSelectionInfo.forEach((info, index) => {
        console.log(`📊 Auto-selected peer ${index + 1}:`, {
          peerId: info.peerId.slice(0, 12),
          score: info.score.toFixed(3),
          allocation: `${availablePeers.find(p => p.peerId === info.peerId)?.percentage}%`,
          metrics: info.metrics
        });
      });

      pushMessage(
        `Auto-selected ${selectedPeers.length} peers with score-weighted distribution`,
        'success',
        3000
      );
    }

    // Route download based on selected protocol
    // Note: 'bittorrent' is handled above and returns early, so it cannot reach here.
    if (selectedProtocol === 'webrtc' || selectedProtocol === 'bitswap') {
      // P2P download flow (WebRTC, Bitswap)
      

      const fileWithSelectedPeers: FileMetadata & {
        peerAllocation?: any[];
        selectedProtocol?: string;
        paymentPeerId?: string;
        paymentPeerWalletAddress?: string;
        paymentPricePerMb?: number;
        estimatedPaymentTotal?: number;
      } = {
        ...selectedFile,
        seeders: selectedPeers,  // Override with selected peers
        peerAllocation,
        selectedProtocol: selectedProtocol,  // Pass the user's protocol selection
        // Keep uploaderAddress aligned with the chosen payment payee for legacy code paths.
        uploaderAddress: paymentPeer.walletAddress,
        paymentPeerId: paymentPeer.peerId,
        paymentPeerWalletAddress: paymentPeer.walletAddress,
        paymentPricePerMb: paymentPeer.price_per_mb,
        estimatedPaymentTotal,
      };

      // Dispatch to parent (Download.svelte)
      dispatch('download', fileWithSelectedPeers);
    } else {
      // This shouldn't happen - direct downloads bypass peer selection
      console.error(`Unexpected protocol in peer selection: ${selectedProtocol}`);
      pushMessage(`Protocol ${selectedProtocol} should not require peer selection`, 'error');
      return;
    }

    // Close modal and reset state
    showPeerSelectionModal = false;
    selectedFile = null;
    pushMessage(`Starting ${selectedProtocol.toUpperCase()} download with ${selectedPeers.length} selected peer${selectedPeers.length === 1 ? '' : 's'}`, 'success', 3000);
  }


  // Cancel peer selection
  function cancelPeerSelection() {
    showPeerSelectionModal = false;
    selectedFile = null;
    // Clear torrent state if canceling a torrent download
    if (pendingTorrentType) {
      pendingTorrentIdentifier = null;
      pendingTorrentBytes = null;
      pendingTorrentType = null;
      latestMetadata = null;
      latestStatus = 'pending';
    }
  }
</script>

<Card class="p-6">
  <div class="space-y-4">
    <div>
      <Label for="hash-input" class="text-xl font-semibold">{tr('download.addNew')}</Label>

      <!-- Search Mode Switcher -->
      <div class="flex gap-2 mb-3 mt-3">
        <select bind:value={searchMode} class="px-3 py-1 text-sm rounded-md border transition-colors bg-muted/50 hover:bg-muted border-border">
            <option value="merkle_hash">Search by File Hash</option>
            <option value="magnet">Search by Magnet Link</option>
            <option value="torrent">Search by .torrent File</option>
            <option value="ed2k">Search by ED2K Link</option>
            <option value="ftp">Search by FTP URL</option>
        </select>
      </div>

      <div class="flex flex-col sm:flex-row gap-3">
        {#if searchMode === 'torrent'}
          <!-- File input for .torrent files -->
          <div class="flex-1">
            <input
              type="file"
              bind:this={torrentFileInput}
              accept=".torrent"
              class="hidden"
              onchange={handleTorrentFileSelect}
            />
            <Button
              variant="default"
              class="w-full h-10 justify-center font-medium cursor-pointer hover:opacity-90"
              on:click={() => torrentFileInput?.click()}
            >
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="mr-2">
                <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
                <polyline points="17 8 12 3 7 8"></polyline>
                <line x1="12" y1="3" x2="12" y2="15"></line>
              </svg>
              {torrentFileName || 'Select .torrent File'}
            </Button>
          </div>
        {:else}
          <div class="relative flex-1 search-input-container">
            <Input
              id="hash-input"
              bind:value={searchHash}
              placeholder={
                searchMode === 'merkle_hash' ? 'Enter file hash (SHA-256)...' :
                searchMode === 'magnet' ? 'magnet:?xt=urn:btih:...' :
                searchMode === 'ed2k' ? 'ed2k://|file|filename|size|hash|/' :
                searchMode === 'ftp' ? 'ftp://user:pass@server.com/path/file' :
                ''
              }
              class="pr-20 h-10"
              on:focus={toggleHistoryDropdown}
              on:keydown={(e: CustomEvent<KeyboardEvent>) => {
                const event = e.detail;
                if (event.key === 'Enter' && searchHash.trim() && !isSearching) {
                  event.preventDefault();
                  searchForFile();
                }
              }}
            />
            {#if searchHash}
              <button
                onclick={clearSearch}
                class="absolute right-10 top-1/2 transform -translate-y-1/2 p-1 hover:bg-muted rounded-full transition-colors"
                type="button"
                aria-label={tr('download.clearInput')}
              >
                <X class="h-4 w-4 text-muted-foreground hover:text-foreground" />
              </button>
            {/if}
            <button
              onclick={toggleHistoryDropdown}
              class="absolute right-2 top-1/2 transform -translate-y-1/2 p-1 hover:bg-muted rounded-full transition-colors"
              type="button"
              aria-label="Toggle search history"
            >
              <History class="h-4 w-4 text-muted-foreground hover:text-foreground" />
            </button>

            {#if showHistoryDropdown}
              <div class="absolute top-full left-0 right-0 mt-1 bg-background border border-border rounded-md shadow-lg z-50 max-h-80 overflow-auto">
              {#if historyEntries.length > 0}
                <div class="p-2 border-b border-border">
                  <div class="flex items-center justify-between">
                    <span class="text-sm font-medium text-muted-foreground">Search History</span>
                    <Button
                      variant="ghost"
                      size="sm"
                      class="h-6 px-2 text-xs"
                      on:click={clearHistory}
                    >
                      <RotateCcw class="h-3 w-3 mr-1" />
                      Clear
                    </Button>
                  </div>
                </div>
                <div class="py-1">
                  {#each historyEntries as entry}
                    {@const StatusIcon = statusIcon(entry.status)}
                    <button
                      type="button"
                      class="w-full px-3 py-2 text-left hover:bg-muted/60 transition-colors flex items-center justify-between"
                      onclick={() => selectHistoryEntry(entry)}
                    >
                      <div class="flex items-center gap-2 flex-1 min-w-0">
                        <span class="text-sm font-medium truncate">{entry.hash}</span>
                      </div>
                      <div class="flex items-center gap-2 text-xs text-muted-foreground">
                        <StatusIcon class={`h-3 w-3 ${statusClass(entry.status)}`} />
                        {#if entry.elapsedMs}
                          <span>{(entry.elapsedMs / 1000).toFixed(1)}s</span>
                        {/if}
                      </div>
                    </button>
                    {#if entry.metadata?.fileName}
                      <div class="px-3 pb-2 text-xs text-muted-foreground truncate">
                        {entry.metadata.fileName}
                      </div>
                    {/if}
                  {/each}
                </div>
              {:else}
                <div class="p-4 text-center">
                  <p class="text-sm text-muted-foreground">No search history yet</p>
                </div>
              {/if}
            </div>
          {/if}
          </div>
        {/if}
        <Button
          on:click={isSearching ? cancelSearch : searchForFile}
          disabled={!isSearching && ((searchMode !== 'torrent' && !searchHash.trim()) || (searchMode === 'torrent' && !torrentFileName))}
          class="h-10 px-6"
          title={isSearching ? 'Cancel search' : (searchMode !== 'torrent' && !searchHash.trim()) ? 'Enter a search hash' : (searchMode === 'torrent' && !torrentFileName) ? 'Select a torrent file' : 'Search'}
        >
          {#if isSearching}
            <X class="h-4 w-4 mr-2" />
            {tr('actions.cancel')}
          {:else}
            <Search class="h-4 w-4 mr-2" />
            {tr('download.search.button')}
          {/if}
        </Button>
      </div>
    </div>

    {#if hasSearched}
      <div class="pt-6 border-t">
        <div class="space-y-4">
            {#if latestStatus === 'found' && latestMetadata}
              <SearchResultCard
                metadata={latestMetadata}
                isLoading={progressiveSearchState.status === 'searching'}
                loadingSeederCount={progressiveSearchState.seeders.filter(s => !s.hasGeneralInfo || !s.hasFileInfo).length}
                seederDetails={progressiveSearchState.seeders}
                on:copy={handleCopy}
                on:download={(event: any) => handleFileDownload(event.detail)}
              />
              {#if progressiveSearchState.status === 'searching'}
                <p class="text-xs text-muted-foreground">Searching for more peers...</p>
              {:else if (progressiveSearchState.status === 'complete' || progressiveSearchState.status === 'timeout') && lastSearchDuration > 0}
                <p class="text-xs text-muted-foreground">
                  {tr('download.search.status.completedIn', { values: { seconds: (lastSearchDuration / 1000).toFixed(1) } })}
                </p>
              {/if}
            {:else if isSearching}
              <SearchResultCardSkeleton />
            {:else if latestStatus === 'not_found'}
              <div class="text-center py-8">
                {#if searchError}
                   <p class="text-sm text-red-500">{searchError}</p>
                {:else}
                   <p class="text-sm text-muted-foreground">{tr('download.search.status.notFoundDetail')}</p>
                {/if}
              </div>
            {:else if latestStatus === 'error'}
              <div class="text-center py-8">
                <p class="text-sm font-medium text-muted-foreground mb-1">{tr('download.search.status.errorHeadline')}</p>
                <p class="text-sm text-muted-foreground">{searchError}</p>
              </div>
            {:else}
              <div class="rounded-md border border-dashed border-muted p-5 text-sm text-muted-foreground text-center">
                {tr('download.search.status.placeholder')}
              </div>
            {/if}
        </div>
      </div>
    {/if}
  </div>
</Card>

<!-- Peer Selection Modal -->
<PeerSelectionModal
  show={showPeerSelectionModal}
  fileName={selectedFile?.fileName || ''}
  fileSize={selectedFile?.fileSize || 0}
  bind:peers={availablePeers}
  bind:mode={peerSelectionMode}
  bind:protocol={selectedProtocol}
  isTorrent={pendingTorrentType !== null}
  {availableProtocols}
  isSeeding={selectedFileIsSeeding}
  on:confirm={confirmPeerSelection}
  on:cancel={cancelPeerSelection}
/>

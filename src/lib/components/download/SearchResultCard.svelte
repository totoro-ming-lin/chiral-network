<script lang="ts">
  import Card from '$lib/components/ui/card.svelte';
  import Badge from '$lib/components/ui/badge.svelte';
  import Button from '$lib/components/ui/button.svelte';
  import { FileIcon, Copy, Download, Server, Globe, Star } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';
  import type { FileMetadata } from '$lib/dht';
  import { formatRelativeTime, toHumanReadableSize } from '$lib/utils';
  import { files, wallet } from '$lib/stores';
  import { favorites } from '$lib/stores/favorites';
  import { t } from 'svelte-i18n';
  import { showToast } from '$lib/toast';
  import { costFromPricePerMb, minPricePerMb } from '$lib/utils/pricing';

  type TranslateParams = { values?: Record<string, unknown>; default?: string };
  const tr = (key: string, params?: TranslateParams): string =>
  $t(key, params);

  interface Props {
    metadata: FileMetadata;
    isBusy?: boolean;
    isLoading?: boolean; // Progressive search loading state
    loadingSeederCount?: number; // Number of seeders still loading
    seederDetails?: Array<{
      index: number;
      peerId: string;
      walletAddress?: string;
      pricePerMb?: number;
      protocols?: string[];
      protocolDetails?: any;
      hasGeneralInfo: boolean;
      hasFileInfo: boolean;
    }>; // Detailed seeder information from progressive search
  }

  let {
    metadata,
    isBusy = false,
    isLoading = false,
    loadingSeederCount = 0,
    seederDetails = []
  }: Props = $props();

  const dispatch = createEventDispatcher<{ download: FileMetadata; copy: string }>();

  const DEV = import.meta.env.DEV;

  let showSeederDetailsModal = $state(false);
  let selectedSeederDetails = $state(null as typeof seederDetails[0] | null);

  const minOfferPricePerMb = $derived.by(() =>
    minPricePerMb((seederDetails || []).map((s) => s.pricePerMb))
  );

  const minOfferTotal = $derived.by(() => {
    if (!metadata?.fileSize || metadata.fileSize <= 0) return null;
    if (minOfferPricePerMb === null) return null;
    return costFromPricePerMb({ bytes: metadata.fileSize, pricePerMb: minOfferPricePerMb });
  });

  const canAfford = $derived.by(() => {
    if (isSeeding) return true;
    if (minOfferTotal === null) return true;
    return $wallet.balance >= minOfferTotal;
  });

  const isPriceLoading = $derived.by(() => !isSeeding && metadata?.fileSize > 0 && minOfferTotal === null);

  function formatFileSize(bytes: number): string {
    return toHumanReadableSize(bytes);
  }

  let seederCount = $derived(metadata?.seeders?.length ?? 0);
  let createdLabel = $derived(metadata?.createdAt
    ? formatRelativeTime(new Date(metadata.createdAt * 1000))
    : null);

  $effect(() => {
    if (!DEV) return;
    if (metadata) {
      console.log('[SearchResultCard] metadata updated', {
        fileHash: metadata.fileHash,
        fileName: metadata.fileName,
        fileSize: metadata.fileSize,
        seeders: metadata.seeders?.length ?? 0,
      });
    }
  });


  // Helper function to determine available protocols for a file
  // Based on progressively updated metadata and seeder details
  let availableProtocols = $derived.by(() => {
    const protocols = [];
    const protocolSet = new Set<string>();

    // First, collect protocols from seeder details if available
    if (seederDetails && seederDetails.length > 0) {
      for (const seeder of seederDetails) {
        if (seeder.protocols && seeder.protocols.length > 0) {
          seeder.protocols.forEach(protocol => protocolSet.add(protocol.toLowerCase()));
        }
      }
    }

    // If no protocols from seeder details, fall back to metadata-based detection
    if (protocolSet.size === 0) {
      const hasInfoHash = !!metadata.infoHash;
      const hasHttpSources = !!(metadata.httpSources && metadata.httpSources.length > 0);
      const hasFtpSources = !!(metadata.ftpSources && metadata.ftpSources.length > 0);
      const hasEd2kSources = !!(metadata.ed2kSources && metadata.ed2kSources.length > 0);
      const hasSeeders = !!(metadata.seeders && metadata.seeders.length > 0);

      // Determine protocols from metadata
      if (hasInfoHash) protocolSet.add('bittorrent');
      if (hasHttpSources) protocolSet.add('http');
      if (hasFtpSources) protocolSet.add('ftp');
      if (hasEd2kSources) protocolSet.add('ed2k');
      if (hasSeeders && !hasInfoHash && !hasHttpSources && !hasFtpSources && !hasEd2kSources) {
        protocolSet.add('webrtc');
      }
    }

    // Convert protocol set to badge objects
    for (const protocol of protocolSet) {
      switch (protocol) {
        case 'webrtc':
          protocols.push({
            id: 'webrtc',
            name: 'WebRTC',
            icon: Globe,
            colorClass: 'bg-blue-100 text-blue-800'
          });
          break;
        case 'bittorrent':
          protocols.push({
            id: 'bittorrent',
            name: 'BitTorrent',
            icon: Server,
            colorClass: 'bg-green-100 text-green-800'
          });
          break;
        case 'http':
          protocols.push({
            id: 'http',
            name: 'HTTP',
            icon: Globe,
            colorClass: 'bg-gray-100 text-gray-800'
          });
          break;
        case 'ftp':
          protocols.push({
            id: 'ftp',
            name: 'FTP',
            icon: Server,
            colorClass: 'bg-gray-100 text-gray-800'
          });
          break;
        case 'ed2k':
          protocols.push({
            id: 'ed2k',
            name: 'ED2K',
            icon: Server,
            colorClass: 'bg-orange-100 text-orange-800'
          });
          break;
        default:
          // Unknown protocol - add a generic badge
          protocols.push({
            id: protocol,
            name: protocol.toUpperCase(),
            icon: Globe,
            colorClass: 'bg-purple-100 text-purple-800'
          });
          break;
      }
    }

    return protocols;
  });

  // Check if user is already seeding this file
  let isSeeding = $derived(!!$files.find(f => f.hash === metadata.fileHash && f.status === 'seeding'));

  function copyHash() {
    navigator.clipboard.writeText(metadata.fileHash).then(() => {
      dispatch('copy', metadata.fileHash);
    });
  }

  function copySeeder(address: string, _index: number) {
    navigator.clipboard.writeText(address).then(() => {
      dispatch('copy', address);
    });
  }

  function copyMagnetLink(link: string) {
    navigator.clipboard.writeText(link).then(() => {
      dispatch('copy', link);
    });
  }

  function copyEd2kLink(link: string) {
    navigator.clipboard.writeText(link).then(() => {
      dispatch('copy', link);
    });
  }

  function copyFtpLink(link: string) {
    navigator.clipboard.writeText(link).then(() => {
      dispatch('copy', link);
    });
  }

  function copyHttpLink(link: string) {
    navigator.clipboard.writeText(link).then(() => {
      dispatch('copy', link);
    });
  }

  function handleDownload() {
    // Check if download should proceed
    if (isBusy) {
      return;
    }

    if (!canAfford && minOfferTotal !== null && minOfferTotal > 0 && !isSeeding) {
      return;
    }

    // Dispatch download event - parent component handles protocol/peer selection
    dispatch('download', metadata);
  }


  function showSeederInfo(peerId: string) {
    // Find the seeder details by peer ID
    const details = seederDetails.find(s => s.peerId === peerId);
    if (details) {
      selectedSeederDetails = details;
      showSeederDetailsModal = true;
    }
  }

  function closeSeederDetailsModal() {
    showSeederDetailsModal = false;
    selectedSeederDetails = null;
  }

  // Favorites functionality
  let isFavorite = $derived(favorites.isFavorite(metadata.fileHash, $favorites));

  function toggleFavorite() {
    if (isFavorite) {
      favorites.remove(metadata.fileHash);
      showToast(tr('toasts.favorites.removed'), 'info');
    } else {
      favorites.add({
        hash: metadata.fileHash,
        name: metadata.fileName,
        size: metadata.fileSize,
        protocol: availableProtocols[0]?.id,
        seeders: metadata.seeders?.length,
        leechers: metadata.leechers?.length
      });
      showToast(tr('toasts.favorites.added'), 'success');
    }
  }

  let seederIds = $derived(metadata.seeders?.map((address, index) => ({
    id: `${metadata.fileHash}-${index}`,
    address,
  })) ?? []);

</script>

<Card class="p-5 space-y-5">
  <div class="flex flex-col gap-4 sm:flex-row sm:items-start sm:justify-between">
    <div class="flex items-start gap-3">
      <div class="w-12 h-12 rounded-md bg-muted flex items-center justify-center">
        <FileIcon class="h-6 w-6 text-muted-foreground" />
      </div>
      <div class="flex-1">
        <h3 class="text-lg font-semibold break-all">{metadata.fileName}</h3>
        <div class="flex flex-wrap items-center gap-2 text-sm text-muted-foreground mt-1">
          {#if createdLabel}
            <span>Published {createdLabel}</span>
          {/if}
          {#if metadata.mimeType}
            {#if createdLabel}
              <span>•</span>
            {/if}
            <span>{metadata.mimeType}</span>
          {/if}
        </div>
      </div>
    </div>

    <div class="flex items-center gap-2 flex-wrap">
      {#each availableProtocols as protocol}
        <Badge class={protocol.colorClass}>
          {@const IconComponent = protocol.icon}
          <IconComponent class="h-3.5 w-3.5 mr-1" />
          {protocol.name}
        </Badge>
      {/each}
    </div>
  </div>

  <div class="grid gap-4 md:grid-cols-2">
    <!-- Left Column: All technical identifiers and details -->
    <div class="space-y-3">
      <div>
        <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">File hash</p>
        <div class="flex items-center gap-2 rounded-md border border-border/50 bg-muted/40 py-1 px-1.5 overflow-hidden">
          <code class="flex-1 text-xs font-mono break-all text-muted-foreground overflow-hidden" style="word-break: break-all;">{metadata.fileHash}</code>
          <Button
            variant="ghost"
            size="icon"
            class="h-7 w-7"
            on:click={copyHash}
          >
            <Copy class="h-3.5 w-3.5" />
            <span class="sr-only">Copy hash</span>
          </Button>
        </div>
      </div>

      {#if metadata.infoHash}
        {@const magnetLink = `magnet:?xt=urn:btih:${metadata.infoHash}${metadata.trackers && metadata.trackers.length > 0 ? '&tr=' + metadata.trackers.join('&tr=') : ''}`}
        <div>
          <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">Magnet Link</p>
          <div class="flex items-center gap-2 rounded-md border border-border/50 bg-muted/40 p-1.5 overflow-hidden">
            <code class="flex-1 text-xs font-mono break-all text-muted-foreground overflow-hidden" style="word-break: break-all;">{magnetLink}</code>
            <Button
              variant="ghost"
              size="icon"
              class="h-7 w-7"
              on:click={() => copyMagnetLink(magnetLink)}
            >
              <Copy class="h-3.5 w-3.5" />
              <span class="sr-only">Copy magnet link</span>
            </Button>
          </div>
        </div>
      {/if}

      {#if metadata.ed2kSources && metadata.ed2kSources.length > 0}
        {@const ed2kSource = metadata.ed2kSources[0]}
        {@const ed2kLink = `ed2k://|file|${metadata.fileName}|${metadata.fileSize}|${ed2kSource.file_hash}|/`}
        <div>
          <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">ED2K Link</p>
          <div class="flex items-center gap-2 rounded-md border border-border/50 bg-muted/40 p-1.5 overflow-hidden">
            <code class="flex-1 text-xs font-mono break-all text-muted-foreground overflow-hidden" style="word-break: break-all;">{ed2kLink}</code>
            <Button
              variant="ghost"
              size="icon"
              class="h-7 w-7"
              on:click={() => copyEd2kLink(ed2kLink)}
            >
              <Copy class="h-3.5 w-3.5" />
              <span class="sr-only">Copy ED2K link</span>
            </Button>
          </div>
        </div>
      {/if}

      {#if metadata.ftpSources && metadata.ftpSources.length > 0}
        {@const ftpSource = metadata.ftpSources[0]}
        <div>
          <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">FTP Link</p>
          <div class="flex items-center gap-2 rounded-md border border-border/50 bg-muted/40 p-1.5 overflow-hidden">
            <code class="flex-1 text-xs font-mono break-all text-muted-foreground overflow-hidden" style="word-break: break-all;">{ftpSource.url}</code>
            <Button
              variant="ghost"
              size="icon"
              class="h-7 w-7"
              on:click={() => copyFtpLink(ftpSource.url)}
            >
              <Copy class="h-3.5 w-3.5" />
              <span class="sr-only">Copy FTP link</span>
            </Button>
          </div>
        </div>
      {/if}

      {#if metadata.httpSources && metadata.httpSources.length > 0}
        {@const httpSource = metadata.httpSources[0]}
        <div>
          <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">HTTP Link</p>
          <div class="flex items-center gap-2 rounded-md border border-border/50 bg-muted/40 p-1.5 overflow-hidden">
            <code class="flex-1 text-xs font-mono break-all text-muted-foreground overflow-hidden" style="word-break: break-all;">{httpSource.url}</code>
            <Button
              variant="ghost"
              size="icon"
              class="h-7 w-7"
              on:click={() => copyHttpLink(httpSource.url)}
            >
              <Copy class="h-3.5 w-3.5" />
              <span class="sr-only">Copy HTTP link</span>
            </Button>
          </div>
        </div>
      {/if}

      <div class="space-y-3">
        <p class="text-xs uppercase tracking-wide text-muted-foreground">Details</p>
        <ul class="space-y-2 text-sm text-foreground">
          <li class="flex items-center justify-between">
            <span class="text-muted-foreground">Seeder count</span>
            <span>{seederCount}</span>
          </li>
          <li class="flex items-center justify-between">
            <span class="text-muted-foreground">Size</span>
            <span>{formatFileSize(metadata.fileSize)}</span>
          </li>
          <li class="flex items-center justify-between">
            <span class="text-muted-foreground">Min file price</span>
            <span class="font-semibold text-emerald-600">
              {#if isSeeding}
                Free
              {:else if isPriceLoading}
                Loading...
              {:else if minOfferTotal !== null}
                {minOfferTotal.toFixed(4)} Chiral
              {:else}
                0.0001 Chiral
              {/if}
            </span>
          </li>
          <li class="text-xs text-muted-foreground text-center col-span-2">
            Min price calculated from currently known seeder offers
          </li>
        </ul>
      </div>
    </div>

    <!-- Right Column: Available peers -->
    <div class="space-y-3">
      {#if isLoading && loadingSeederCount > 0}
        <div class="space-y-2">
          <p class="text-xs uppercase tracking-wide text-muted-foreground">Available peers</p>
          <div class="space-y-2 max-h-40 overflow-auto pr-1">
            {#each Array(loadingSeederCount) as _}
              <div class="flex items-start gap-2 rounded-md border border-border/50 bg-muted/40 p-2 overflow-hidden animate-pulse">
                <div class="mt-0.5 h-2 w-2 rounded-full bg-gray-300 flex-shrink-0"></div>
                <div class="space-y-1 flex-1">
                  <div class="h-4 bg-gray-300 rounded w-3/4"></div>
                  <div class="h-3 bg-gray-200 rounded w-1/4"></div>
                </div>
              </div>
            {/each}
          </div>
          <p class="text-xs text-muted-foreground text-center">Loading seeder information...</p>
        </div>
      {:else if metadata.seeders?.length}
        <div class="space-y-2">
          <p class="text-xs uppercase tracking-wide text-muted-foreground">Available peers</p>
          <div class="space-y-2 max-h-40 overflow-auto pr-1">
            {#each seederIds as seeder, index}
              {@const details = seederDetails.find(s => s.peerId === seeder.address)}
              <!-- Debug: Peer {seeder.address}, Details: {JSON.stringify(details)} -->
              <button
                type="button"
                class="w-full flex items-start gap-2 rounded-md border border-border/50 bg-muted/40 p-2 overflow-hidden hover:bg-muted/60 transition-colors cursor-pointer text-left"
                onclick={() => showSeederInfo(seeder.address)}
                title={details?.hasGeneralInfo ? 'Click to view seeder details' : 'Seeder info loading...'}
              >
                <div class="mt-0.5 h-2 w-2 rounded-full bg-emerald-500 flex-shrink-0"></div>
                <div class="space-y-1 flex-1 min-w-0">
                  <code class="text-xs font-mono break-words block">{seeder.address}</code>
                  <div class="flex items-center gap-2 text-xs text-muted-foreground">
                    <span>Seed #{index + 1}</span>
                    {#if details?.hasGeneralInfo}
                      <span class="text-emerald-600">• Info available</span>
                    {:else}
                      <span class="text-amber-600">• Loading...</span>
                    {/if}
                  </div>
                  {#if details?.walletAddress}
                    <div class="text-xs text-muted-foreground truncate">
                      {details.walletAddress.slice(0, 10)}...
                    </div>
                  {/if}
                </div>
                <Button
                  variant="ghost"
                  size="icon"
                  class="h-7 w-7"
                  on:click={(e) => {
                    e.stopPropagation();
                    copySeeder(seeder.address, index);
                  }}
                >
                  <Copy class="h-3.5 w-3.5" />
                  <span class="sr-only">Copy seeder address</span>
                </Button>
              </button>
            {/each}
          </div>
        </div>
      {:else}
        <div class="space-y-2">
          <p class="text-xs uppercase tracking-wide text-muted-foreground">Available peers</p>
          <p class="text-xs text-muted-foreground italic">No seeders reported yet for this file.</p>
        </div>
      {/if}
    </div>
  </div>

  <div class="flex flex-col sm:flex-row gap-3 sm:items-center sm:justify-between">
    <div class="text-xs text-muted-foreground">
      {#if isSeeding}
        <span class="text-emerald-600 font-semibold">You are seeding this file</span>
        {#if metadata.isEncrypted}
          <span class="ml-2 text-xs text-amber-600">(encrypted)</span>
        {/if}
      {:else if !canAfford && minOfferTotal !== null && minOfferTotal > 0}
        <span class="text-red-600 font-semibold">Insufficient balance to download this file</span>
      {:else if metadata.seeders?.length}
        {metadata.seeders.length > 1 ? '' : 'Single seeder available.'}
      {:else}
        Waiting for peers to announce this file.
      {/if}
    </div>
    <div class="flex items-center gap-2">
      <Button
        variant="ghost"
        size="icon"
        on:click={toggleFavorite}
        class="h-9 w-9 {isFavorite ? 'text-yellow-500 hover:text-yellow-600' : 'text-gray-400 hover:text-gray-600'}"
        title={isFavorite ? tr('favorites.remove') : tr('favorites.add')}
      >
        <Star class="h-4 w-4 {isFavorite ? 'fill-current' : ''}" />
      </Button>
      <Button
        on:click={handleDownload}
        disabled={isBusy || (!canAfford && minOfferTotal !== null && minOfferTotal > 0 && !isSeeding)}
        class={!canAfford && minOfferTotal !== null && minOfferTotal > 0 && !isSeeding ? 'opacity-50 cursor-not-allowed' : ''}
      >
        <Download class="h-4 w-4 mr-2" />
        {#if !canAfford && minOfferTotal !== null && minOfferTotal > 0 && !isSeeding}
          Insufficient funds
        {:else}
          Download
        {/if}
      </Button>
    </div>
  </div>

{#if showSeederDetailsModal && selectedSeederDetails}
  <div class="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <div class="bg-background rounded-lg shadow-lg p-6 w-full max-w-md border border-border">
      <h2 class="text-xl font-bold mb-4">Seeder Details</h2>

      <div class="space-y-4">
        <!-- Peer ID -->
        <div>
          <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">Peer ID</p>
          <div class="flex items-center gap-2 rounded-md border border-border/50 bg-muted/40 p-2">
            <code class="flex-1 text-xs font-mono break-all">{selectedSeederDetails?.peerId}</code>
            <Button
              variant="ghost"
              size="icon"
              class="h-7 w-7"
              on:click={() => {
                navigator.clipboard.writeText(selectedSeederDetails?.peerId || '');
                showToast('Peer ID copied', 'success');
              }}
            >
              <Copy class="h-3.5 w-3.5" />
            </Button>
          </div>
        </div>

        {#if selectedSeederDetails?.hasGeneralInfo}
          <!-- Wallet Address -->
          {#if selectedSeederDetails?.walletAddress}
            <div>
              <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">Wallet Address</p>
              <div class="flex items-center gap-2 rounded-md border border-border/50 bg-muted/40 p-2">
                <code class="flex-1 text-xs font-mono break-all">{selectedSeederDetails?.walletAddress}</code>
                <Button
                  variant="ghost"
                  size="icon"
                  class="h-7 w-7"
                  on:click={() => {
                    navigator.clipboard.writeText(selectedSeederDetails?.walletAddress || '');
                    showToast('Wallet address copied', 'success');
                  }}
                >
                  <Copy class="h-3.5 w-3.5" />
                </Button>
              </div>
            </div>
          {/if}

          <!-- Price -->
          {#if selectedSeederDetails?.pricePerMb !== undefined}
            <div>
              <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">Price per MB</p>
              <div class="rounded-md border border-border/50 bg-muted/40 p-3">
                <p class="text-lg font-bold text-emerald-600">
                  {selectedSeederDetails?.pricePerMb?.toFixed(6)} Chiral
                </p>
                <p class="text-xs text-muted-foreground mt-1">
                  Total: {((selectedSeederDetails?.pricePerMb || 0) * (metadata.fileSize / (1024 * 1024))).toFixed(4)} Chiral
                </p>
              </div>
            </div>
          {/if}

          <!-- Supported Protocols -->
          {#if selectedSeederDetails?.protocols && selectedSeederDetails.protocols.length > 0}
            <div>
              <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">Supported Protocols</p>
              <div class="flex flex-wrap gap-2">
                {#each selectedSeederDetails?.protocols || [] as protocol}
                  <Badge class="bg-blue-100 text-blue-800">
                    {protocol}
                  </Badge>
                {/each}
              </div>
            </div>
          {/if}

          <!-- Protocol Details -->
          {#if selectedSeederDetails?.hasFileInfo && selectedSeederDetails?.protocolDetails}
            <div>
              <p class="text-xs uppercase tracking-wide text-muted-foreground mb-1">Protocol Details</p>
              <div class="rounded-md border border-border/50 bg-muted/40 p-3 max-h-40 overflow-auto">
                <pre class="text-xs font-mono whitespace-pre-wrap break-all">{JSON.stringify(selectedSeederDetails?.protocolDetails, null, 2)}</pre>
              </div>
            </div>
          {/if}
        {:else}
          <div class="p-4 bg-amber-500/10 rounded-lg border border-amber-500/30">
            <p class="text-sm text-amber-600 text-center">
              Seeder information is still loading...
            </p>
          </div>
        {/if}
      </div>

      <div class="flex justify-end gap-2 mt-6">
        <Button variant="outline" on:click={closeSeederDetailsModal}>
          Close
        </Button>
      </div>
    </div>
  </div>
{/if}
</Card>

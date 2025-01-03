<script>
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import { slide } from 'svelte/transition';

  let vaults = [];
  let prices = {};
  let loading = true;
  let error = null;
  let toastMessage = '';
  let showToast = false;
  let nodesData = [];
  let pools = [];
  let networkData = null;
  let showAssetBalances = false;

  const chainIcons = {
    BTC: '/assets/coins/bitcoin-btc-logo.svg',
    ETH: '/assets/coins/ethereum-eth-logo.svg',
    BCH: '/assets/coins/bitcoin-cash-bch-logo.svg',
    LTC: '/assets/coins/litecoin-ltc-logo.svg',
    DOGE: '/assets/coins/dogecoin-doge-logo.svg',
    AVAX: '/assets/coins/avalanche-avax-logo.svg',
    BSC: '/assets/coins/binance-coin-bnb-logo.svg',
    GAIA: '/assets/coins/cosmos-atom-logo.svg',
    THOR: '/assets/coins/RUNE-ICON.svg'
  };

  async function fetchPrices() {
    try {
      const response = await fetch('https://thornode.ninerealms.com/thorchain/pools');
      const pools = await response.json();
      
      const priceMap = pools.reduce((acc, pool) => {
        const assetExistsInVaults = vaults.some(vault => 
          vault.coins.some(coin => coin.asset === pool.asset)
        );
        
        if (assetExistsInVaults) {
          acc[pool.asset] = Number(pool.asset_tor_price) / 1e8;
        }
        return acc;
      }, {});

      return { prices: priceMap, pools };
    } catch (e) {
      console.error('Error fetching prices:', e);
      return { prices: {}, pools: [] };
    }
  }

  function formatAmount(amount) {
    return new Intl.NumberFormat('en-US', {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2
    }).format(Number(amount) / 1e8);
  }

  function formatPrice(amount) {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD'
    }).format(amount);
  }

  function formatAssetName(asset) {
    const [chain, token] = asset.split('.');
    
    // Handle native assets (e.g., BTC.BTC, ETH.ETH)
    if (token === chain) {
      return token;
    }
    
    // Handle synths and tokens (e.g., AVAX.USDC-0XB97EF9EF8734C71904D8002F)
    const baseToken = token.split('-')[0];
    return `${baseToken} (${chain})`;
  }

  async function fetchNodes() {
    try {
      const response = await fetch('https://thornode.ninerealms.com/thorchain/nodes');
      nodesData = await response.json();
    } catch (e) {
      console.error('Error fetching nodes:', e);
    }
  }

  function calculateVaultBond(vault) {
    if (!vault || !vault.membership || !nodesData.length) return 0;
    
    const totalBond = vault.membership.reduce((sum, pubkey) => {
      const node = nodesData.find(n => n.pub_key_set?.secp256k1 === pubkey);
      if (node) {
        return sum + Number(node.total_bond);
      }
      return sum;
    }, 0);
    
    return totalBond / 1e8;
  }

  function calculateVaultBondUSD(bondInRune) {
    if (!networkData) return 0;
    const runePrice = Number(networkData.rune_price_in_tor) / 1e8;
    return bondInRune * runePrice;
  }

  function calculateTotalAssetValue(coins) {
    return coins.reduce((sum, coin) => {
      if (prices[coin.asset]) {
        const amount = Number(coin.amount) / 1e8;
        return sum + (amount * prices[coin.asset]);
      }
      return sum;
    }, 0);
  }

  async function fetchNetwork() {
    try {
      const response = await fetch('https://thornode.ninerealms.com/thorchain/network');
      networkData = await response.json();
    } catch (e) {
      console.error('Error fetching network data:', e);
    }
  }

  async function fetchVaults() {
    try {
      const [vaultsResponse] = await Promise.all([
        fetch('https://thornode.ninerealms.com/thorchain/vaults/asgard'),
        fetchNodes(),
        fetchNetwork()
      ]);
      
      const data = await vaultsResponse.json();
      vaults = data;
      const priceData = await fetchPrices();
      prices = priceData.prices;
      pools = priceData.pools;
      loading = false;
    } catch (e) {
      error = e.message;
      loading = false;
    }
  }

  onMount(() => {
    fetchVaults();
  });

  function formatVaultName(pubKey) {
    return pubKey.slice(-4).toUpperCase();
  }

  function formatUSDValue(amount, price = 1) {
    const value = price === 1 ? amount : (Number(amount) / 1e8) * price;
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: 0,
      maximumFractionDigits: 0
    }).format(value);
  }

  function copyToClipboard(text, description) {
    navigator.clipboard.writeText(text).then(() => {
      toastMessage = `Copied ${description}!`;
      showToast = true;
      setTimeout(() => {
        showToast = false;
      }, 3000);
    });
  }

  function toggleAllVaults() {
    showAssetBalances = !showAssetBalances;
  }

  function shortenAddress(address, maxLength = 24) {
    if (!address) return '';
    if (address.length <= maxLength) return address;
    
    const start = Math.ceil(maxLength / 2);
    const end = Math.floor(maxLength / 2);
    
    return address.slice(0, start) + '...' + address.slice(-end);
  }
</script>

<main>
  <div class="container">
    <h1>THORChain Vaults</h1>

    {#if loading}
      <div class="loading">Loading vaults...</div>
    {:else if error}
      <div class="error">Error: {error}</div>
    {:else}
      <div class="vaults-container">
        {#each vaults as vault}
          <div class="vault-card" transition:fade>
            <div class="card-header">
              <h2>Vault {formatVaultName(vault.pub_key)}</h2>
              <div class="pubkey clickable" on:click={() => copyToClipboard(vault.pub_key, 'vault pubkey')}>
                {shortenAddress(vault.pub_key)}
              </div>
            </div>

            <div class="card-content">
              <h4>Chain Addresses</h4>
              {#each vault.addresses as address}
                <div class="address-row">
                  <img 
                    src={chainIcons[address.chain.split('.')[0]]} 
                    alt={address.chain}
                    class="chain-icon"
                    on:error={(e) => {
                      e.target.onerror = null;
                      e.target.src = '/assets/coins/fallback-logo.svg';
                    }}
                  />
                  <span class="chain-name">{address.chain}</span>
                  <span 
                    class="address clickable" 
                    on:click={() => copyToClipboard(address.address, `${address.chain} address`)}
                  >{shortenAddress(address.address)}</span>
                </div>
              {/each}

              <hr />

              <div class="total-bond">
                <div class="bond-row">
                  <span>Total Bond:</span>
                  <span class="clickable amount-with-icon" on:click={() => copyToClipboard(Math.floor(calculateVaultBond(vault)).toLocaleString(), 'bond amount')}>
                    {Math.floor(calculateVaultBond(vault)).toLocaleString()}
                    <img 
                      src="/assets/coins/RUNE-ICON.svg" 
                      alt="RUNE"
                      class="rune-icon"
                      on:error={(e) => {
                        e.target.onerror = null;
                        e.target.src = '/assets/coins/fallback-logo.svg';
                      }}
                    />
                  </span>
                </div>
                <div class="bond-row">
                  <span> Total Bond Value:</span>
                  <span class="clickable" on:click={() => copyToClipboard(formatUSDValue(calculateVaultBondUSD(calculateVaultBond(vault))), 'bond value')}>
                    {formatUSDValue(calculateVaultBondUSD(calculateVaultBond(vault)))}
                  </span>
                </div>
                <div class="bond-row">
                  <span>Total Asset Value:</span>
                  <span class="clickable" on:click={() => copyToClipboard(formatUSDValue(calculateTotalAssetValue(vault.coins)), 'total asset value')}>
                    {formatUSDValue(calculateTotalAssetValue(vault.coins))}
                  </span>
                </div>
              </div>

              <div class="expand-section">
                <button 
                  class="expand-button" 
                  class:expanded={showAssetBalances}
                  on:click={toggleAllVaults}
                >
                  <svg 
                    width="24" 
                    height="24" 
                    viewBox="0 0 24 24" 
                    fill="none" 
                    stroke="currentColor" 
                    stroke-width="2" 
                    stroke-linecap="round" 
                    stroke-linejoin="round"
                  >
                    <polyline points="6 9 12 15 18 9"></polyline>
                  </svg>
                </button>
              </div>

              {#if showAssetBalances}
                <div transition:slide={{ duration: 300 }}>
                  <h4>Asset Balances</h4>
                  {#each vault.coins.filter(coin => prices[coin.asset]) as coin}
                    <div class="balance-row">
                      <span class="asset-name">{formatAssetName(coin.asset)}</span>
                      <div class="amount-group">
                        <span 
                          class="amount clickable" 
                          on:click={() => copyToClipboard(formatAmount(coin.amount), `${formatAssetName(coin.asset)} amount`)}
                        >{formatAmount(coin.amount)}</span>
                        <span 
                          class="usd-value clickable" 
                          on:click={() => copyToClipboard(formatUSDValue(coin.amount, prices[coin.asset]), `${formatAssetName(coin.asset)} USD value`)}
                        >{formatUSDValue(coin.amount, prices[coin.asset])}</span>
                      </div>
                    </div>
                  {/each}
                </div>
              {/if}
            </div>
          </div>
        {/each}
      </div>
    {/if}

    {#if showToast}
      <div class="toast" transition:fade>
        {toastMessage}
      </div>
    {/if}
  </div>
</main>

<style>
  main {
    padding: 1rem;
    color: var(--text-color);
  }

  .container {
    max-width: 1400px;
    margin: 0 auto;
    padding: 0 2rem;
  }

  h1 {
    text-align: center;
    margin-bottom: 2rem;
    font-size: 1.5rem;
  }

  .vaults-container {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 1rem;
    max-width: 100%;
  }

  @media (min-width: 1400px) {
    .vaults-container {
      grid-template-columns: repeat(3, 1fr);
    }
  }

  .vault-card {
    background-color: var(--surface-color);
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    transition: transform 0.3s ease;
  }

  .vault-card:hover {
    transform: translateY(-5px);
  }

  .card-header {
    background-color: #4A90E2;
    padding: 1rem;
  }

  .card-header h2 {
    margin: 0;
    color: #ecf0f1;
    font-size: 1.2rem;
  }

  .pubkey {
    font-family: monospace;
    font-size: 0.8rem;
    color: rgba(236, 240, 241, 0.8);
    margin-top: 0.5rem;
    white-space: nowrap;
  }

  .card-content {
    padding: 1rem;
  }

  .address-row {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 0.5rem;
    font-size: 0.9rem;
    width: 100%;
  }

  .chain-icon {
    width: 24px;
    height: 24px;
  }

  .chain-name {
    min-width: 60px;
    flex-shrink: 0;
    color: var(--text-color);
  }

  .address {
    font-family: monospace;
    font-size: 0.8rem;
    color: var(--secondary-text-color);
    white-space: nowrap;
    flex: 1;
  }

  .balance-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.25rem;
    font-size: 0.9rem;
  }

  .amount-group {
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .asset-name {
    color: var(--text-color);
    font-weight: 500;
  }

  .amount {
    font-family: monospace;
    color: var(--text-color);
  }

  .usd-value {
    font-size: 0.65rem;
    color: var(--secondary-text-color);
    opacity: 0.8;
  }

  hr {
    border: none;
    border-top: 1px solid var(--border-color);
    margin: 1rem 0;
  }

  h4 {
    margin: 0.5rem 0;
    color: var(--text-color);
    font-size: 1rem;
    font-weight: 500;
  }

  .loading {
    text-align: center;
    padding: 2rem;
    color: var(--text-color);
  }

  .error {
    color: var(--error-color);
    text-align: center;
    padding: 2rem;
  }

  @media (max-width: 600px) {
    .container {
      padding: 0 1rem;
    }

    h1 {
      font-size: 1.2rem;
    }

    .vaults-container {
      grid-template-columns: 1fr;
    }

    .card-header h2 {
      font-size: 1rem;
    }

    .card-content {
      padding: 0.75rem;
    }
  }

  .clickable {
    cursor: pointer;
    user-select: none;
    transition: opacity 0.2s ease;
  }

  .clickable:hover {
    opacity: 0.8;
  }

  .clickable:active {
    opacity: 0.6;
  }

  .toast {
    position: fixed;
    bottom: 60px;
    left: 50%;
    transform: translateX(-50%);
    background-color: var(--surface-color);
    color: var(--text-color);
    padding: 0.5rem 1rem;
    border-radius: 4px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
    z-index: 1000;
    font-size: 0.9rem;
    max-width: 80%;
    text-align: center;
  }

  /* Make text selectable only when copying */
  .address, .amount, .usd-value {
    user-select: none;
  }

  .total-bond {
    font-size: 0.9rem;
    color: var(--text-color);
    margin-bottom: 1rem;
    padding: 0.75rem;
    background: var(--surface-color-secondary);
    border-radius: 4px;
  }

  .bond-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.25rem;
  }

  .bond-row:last-child {
    margin-bottom: 0;
  }

  .amount-with-icon {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .rune-icon {
    width: 16px;
    height: 16px;
  }

  .expand-section {
    display: flex;
    justify-content: center;
    margin-top: 0.5rem;
  }

  .expand-button {
    background: none;
    border: none;
    cursor: pointer;
    padding: 0.5rem;
    color: var(--text-color);
    opacity: 0.7;
    transition: all 0.3s ease;
  }

  .expand-button:hover {
    opacity: 1;
  }

  .expand-button.expanded svg {
    transform: rotate(180deg);
  }

  .expand-button svg {
    transition: transform 0.3s ease;
  }
</style>

<template>
  <v-card class="pa-4 mt-4" elevation="8">
    <v-card-title class="text-h6">DApp Directory</v-card-title>

    <v-row class="mb-4">
      <v-col cols="12" md="6">
        <v-text-field
          v-model="search"
          label="Search dApp"
          prepend-icon="mdi-magnify"
          density="compact"
        />
      </v-col>

      <v-col cols="12" md="6">
        <v-select
          v-model="selectedCategory"
          :items="categories"
          label="Category"
          density="compact"
        />
      </v-col>
    </v-row>

    <v-row>
      <v-col
        v-for="dapp in filteredDapps"
        :key="dapp.id"
        cols="12"
        md="4"
      >
        <v-card class="pa-3 hover-card" @click="selectDapp(dapp)">
          <v-card-title class="d-flex justify-space-between align-items-center">
           <div> {{ dapp.name }}</div>
            <v-img max-width="48" :src="dapp.logo"></v-img>
          </v-card-title>
          <v-card-text>
            <div class="text-caption">{{ dapp.category }}</div>
            <div class="text-body-2 mt-2">{{ dapp.description }}</div>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-card>
</template>

<script setup>
import { ref, computed } from 'vue'

const search = ref('')
const selectedCategory = ref('All')

const dapps = ref([
  {
    id: 1,
    name: 'Catton AI',
    category: 'Gaming',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67eff2c59b8145c6ecb86f1b_CattonAiTokenLogo_400.webp',
    description: 'Catton AI, backed by Forj & Ape Accelerator, leads AI NPC gaming on Telegram with 900K users and 300k holders.'
  },
  {
    id: 2,
    name: 'Cult Markets',
    category: 'NFT',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/6847444702a242ed322515e4_cultL.webp',
    description: 'Gamified omnichain NFT marketplace powering dynamic drops, collectibles, and interactive shard-based campaigns.'
  },
  {
    id: 3,
    name: 'DRKVRS',
    category: 'Game',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67d3a1aa464cdce0ecbee499_drkvrs%20jpg.webp',
    description: 'DRKVRS is a Web3 Multiplayer Action RPG game with innovative mechanics, set in a dystopian and brutalist world.'
  },
  {
    id: 4,
    name: 'Aarna',
    category: 'AI',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/6834a164fae7947828580000_Twitter-Logo.webp',
    description: 'Next-generation DeFi asset management platform via crypto structured products, merging AI and tokenization.'
  },
  {
    id: 5,
    name: 'Accountable',
    category: 'RWA',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/68a7302c461b97699ee0f421_Profile%20Pic%20(7)%20(1).webp',
    description: 'YieldApp is the first yield marketplace backed by live, cryptographically verifiable data users can trust.'
  },
  {
    id: 6,
    name: 'Blocklive',
    category: 'RWA',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67b911fd7f3041834a53238c_Blocklive_logo.webp',
    description: 'Blocklive is a platform for end-to-end onchain event management and ticketing, using proof of history to target and reward fans.'
  },
  {
    id: 7,
    name: 'Farcaster',
    category: 'Social',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/6883e31c7002016d7f174e41_fc.webp',
    description: 'Farcaster is a decentralized social app that has an embedded wallet and mini apps that support Monad'
  },
  {
    id: 7,
    name: 'Flap',
    category: 'Social',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67b914c8f0b471b3c1bcda4d_Flap_logo-p-2000.webp',
    description: 'Launch your coin with just one click on Monad with Flap, the premier memecoin launchpad.'
  },
  {
    id: 8,
    name: 'CoNFT',
    category: 'NFT',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67c6400453e750640690be80_conft.webp',
    description: 'coNFT.app-NFT aggregator where users can create/trade NFT and register web3 domains.120k MAU. 1M+ mints. 70k+ web3 registrations.'
  },
  {
    id: 9,
    name: 'Kingdomly',
    category: 'NFT',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/6883ea2d90ec8f542c785833_kingdom%20logo.webp',
    description: 'An all in one NFT Dapp, where users can launch, mint, trade, bridge, stake on Kingdomly 🏰'
  },
  {
    id: 10,
    name: 'AUSD',
    category: 'DeFi',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67c620f4a0bab10af98e0508_ausd.webp',
    description: 'Agora is a stablecoin issuer of AUSD, backed 1:1 by cash and cash equivalent reserves managed by VanEck and custodied by State Street.'
  },
  {
    id: 11,
    name: 'AethonSwap',
    category: 'DeFi',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/68e0395d301c8d49539f8fed_AethonSwap-icon-Twitter-1-2-.webp',
    description: 'AethonSwap is a CLAMM V4 DEX combining next-gen tech with intelligent design, for low-slippage trading & efficient liquidity management'
  },
  {
    id: 12,
    name: 'Ambient',
    category: 'DeFi',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67b910fa442716854f1f3390_Ambient_logo.webp',
    description: 'Spot AMM with combining multiple liquidity types with modular hooks, dynamic fees and MEV protection.'
  },
  {
    id: 13,
    name: 'Amertis',
    category: 'DeFi',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67cb75a41f9084980878c240_amertis_logo.webp',
    description: 'Connecting users to deep liquidity across multiple sources, ensuring the best rates, minimal slippage, and an optimised DeFi experience.'
  },
  {
    id: 14,
    name: 'Ammalgam',
    category: 'DeFi',
    logo: 'https://cdn.prod.website-files.com/669ade140a683001b9f7fd78/67b911051d1e917a4b1ab9b8_Ammalgam_logo.webp',
    description: 'Ammalgam is a new primitive that combines lending and trading into one protocol called a Decentralized Lending Exchange.'
  },
])

const categories = ['All', 'DeFi', 'NFT', 'Gaming', 'AI', 'RWA', 'Social']

const filteredDapps = computed(() => {
  return dapps.value.filter(dapp => {
    const matchSearch = dapp.name.toLowerCase().includes(search.value.toLowerCase())
    const matchCategory =
      selectedCategory.value === 'All' || dapp.category === selectedCategory.value

    return matchSearch && matchCategory
  })
})

function selectDapp(dapp) {
  alert(`Opening ${dapp.name}`)
}
</script>

<style scoped>
.hover-card {
  cursor: pointer;
  transition: 0.3s;
}
.hover-card:hover {
  transform: translateY(-5px);
}
</style>

<template>
  <v-container fluid>
    <v-row>
      <v-col cols="12" md="12" class="list-panel">
        <v-text-field
          v-model="filter"
          append-icon="mdi-magnify"
          label="Name"
          dense
          hide-details
          outlined
          light
          background-color="white"
          class="mb-2 filter-input"
        />
        <v-select
          v-model="selectedCategory"
          :items="uniqueCategories"
          label="Category"
          dense
          hide-details
          clearable
          outlined
          light
          background-color="white"
          class="filter-input"
        />
        <v-divider class="my-2" />

        <div v-for="d in filtered" :key="d.id" class="mb-3">
          <v-hover v-slot="{ hover }">
            <v-card
              :elevation="d.id === activeDappId ? 12 : (hover ? 4 : 2)"
              class="pa-3 d-flex align-center"
              :class="{ 'active-card': d.id === activeDappId }"
              @click="selectDapp(d.id)"
            >
              <v-avatar size="64" class="me-3">
                <v-img :src="d.logo" alt="logo" />
              </v-avatar>

              <div class="flex-grow-1">
                <div class="d-flex align-center justify-space-between">
                  <div>
                    <div class="title">{{ d.name }}</div>
                    <div class="subtitle-2 text--secondary">{{ d.category }}</div>
                  </div>
                  <v-chip small outlined>
                    <v-icon left small>{{ getCategoryIcon(d.category) }}</v-icon>
                    {{ d.category }}
                  </v-chip>
                </div>
                <div class="text-body-2 mt-2 description">{{ d.description }}</div>
              </div>
            </v-card>
          </v-hover>
        </div>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { ref, computed, watch } from "vue";

const props = defineProps({
  dapps: { type: Array, default: () => [] },
  activeDappId: { type: [String, Number, null], default: null }
});

const emit = defineEmits(["select", "filter-change"]);

const filter = ref("");
const selectedCategory = ref(null);

function selectDapp(id) {
  emit("select", id);
}

// Tự động emit khi filter thay đổi → 3D sẽ cập nhật theo
watch([filter, selectedCategory], () => {
  emit("filter-change", {
    filter: filter.value,
    category: selectedCategory.value
  });
}, { immediate: true });

const uniqueCategories = computed(() => {
  return [...new Set(props.dapps.map(d => d.category))].sort();
});

const filtered = computed(() => {
  let result = props.dapps;
  if (filter.value) {
    const q = filter.value.toLowerCase();
    result = result.filter(d =>
      (d.name + " " + d.description + " " + d.category).toLowerCase().includes(q)
    );
  }
  if (selectedCategory.value) {
    result = result.filter(d => d.category === selectedCategory.value);
  }
  return result;
});

function getCategoryIcon(category) {
  const map = {
    Gaming: "mdi-gamepad-variant",
    NFT: "mdi-image-frame",
    DeFi: "mdi-bank",
    Social: "mdi-account-group",
    Tools: "mdi-tools",
    Infrastructure: "mdi-server-network"
  };
  return map[category] || "mdi-apps";
}
</script>

<style scoped>
.list-panel {
  max-height: 100vh;
  overflow-y: auto;
  padding-right: 8px;
  background: linear-gradient(180deg, #0b1020 0%, #1a1f3a 100%);
}
.active-card {
  border-left: 6px solid #ffd54f;
  background: linear-gradient(90deg, rgba(255,213,79,0.06), transparent);
  box-shadow: 0 4px 12px rgba(255,213,79,0.3);
}
.description {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  color: #bbccdd;
}
.v-card {
  transition: all 0.3s ease;
}
.v-card:hover {
  transform: scale(1.02);
  box-shadow: 0 2px 8px rgba(255,255,255,0.1);
}
.filter-input .v-input__slot { background: white !important; }
.list-panel::-webkit-scrollbar { width: 6px; background: #1a1f3a; }
.list-panel::-webkit-scrollbar-thumb { background: #ffd54f; border-radius: 3px; }
/* NEW: Fix dark mode inputs visibility */
.v-text-field,
.v-select {
  color: #ffffff !important;
}
.v-text-field input,
.v-select input,
.v-field__input {
  color: #ffffff !important;
  caret-color: #ffd54f !important;
}
.v-text-field label,
.v-select label,
.v-label {
  color: #bbccdd !important;
}
.v-field__outline {
  color: #6677cc !important;
}
.v-field--focused .v-field__outline {
  color: #ffd54f !important;
}
.v-field__clearable .v-field__clear {
  color: #bbccdd !important;
}
</style>

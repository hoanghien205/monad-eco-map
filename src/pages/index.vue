<template>
  <v-app>
    <v-app-bar app color="#0b1020" dark>  <!-- NEW: Add app bar for title -->
      <v-toolbar-title>Dapp Explorer</v-toolbar-title>
      <v-spacer />
      <!-- Optional: Add toggle if needed -->
    </v-app-bar>
    <v-main>
      <v-container fluid class="pa-0">  <!-- UPDATED: No padding for full-width -->
        <v-row>
          <v-col cols="12" md="4" class="list-col">  <!-- NEW: Class for responsive -->
            <DappList
              :dapps="dapps"
              :activeDappId="activeDappId"
              @select="onSelectFromList"
            />
          </v-col>

          <v-col cols="12" md="8" class="pa-0 canvas-col">  <!-- NEW: Class for responsive -->
            <!-- NEW: Optional overlay for loading -->
            <v-overlay :value="loading" absolute>
              <v-progress-circular indeterminate color="#ffd54f" />
            </v-overlay>
            <ThreeScene
              :dapps="dapps"
              :highlightId="activeDappId"
              @select-dapp="onSelectFrom3D"
            />
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<script setup>
import { ref } from "vue";
import ThreeScene from "../components/ThreeScene.vue";
import DappList from "../components/DappList.vue";
import data from "../data.json";

const dapps = data;

const activeDappId = ref(null);
const loading = ref(true);  // NEW: Loading state, set false after mount if needed

function onSelectFrom3D(id) {
  activeDappId.value = id;
}

function onSelectFromList(id) {
  activeDappId.value = id;
}

// NEW: Simulate loading end
setTimeout(() => { loading.value = false; }, 1000);
</script>

<style>
html, body, #app {
  height: 100%;
  overflow: hidden;  /* UPDATED: Prevent overflow */
  background: #0b1020;
  font-family: 'Roboto', sans-serif;
}
/* NEW: Responsive media query */
@media (max-width: 960px) {
  .canvas-col { order: 1; }  /* 3D on top for mobile */
  .list-col { order: 2; }
}
</style>

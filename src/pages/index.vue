<template>
  <v-app>
    <v-app-bar app color="#0b1020" dark elevation="0">
      <v-toolbar-title class="font-weight-bold">Dapp Explorer 3D</v-toolbar-title>
      <v-spacer />
    </v-app-bar>

    <v-main>
      <v-container fluid class="pa-0">
        <v-row>
          <v-col cols="12" md="4" class="list-col">
            <DappList
              :dapps="dapps"
              :activeDappId="activeDappId"
              @select="onSelectFromList"
              @filter-change="onFilterChange"
            />
          </v-col>

          <v-col cols="12" md="8" class="pa-0 canvas-col">
            <ThreeScene
              :dapps="dapps"
              :highlightId="activeDappId"
              :filterState="filterState"
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
import DappList from "../components/DappList.vue";
import ThreeScene from "../components/ThreeScene.vue";
import data from "../data.json";
const dapps = data;

const activeDappId = ref(null);
const filterState = ref({ filter: "", category: null });

function onSelectFromList(id) {
  activeDappId.value = id;
}
function onSelectFrom3D(id) {
  activeDappId.value = id;
}
function onFilterChange(state) {
  filterState.value = state;
}
</script>

<style>
html, body, #app { height: 100%; overflow: hidden; background: #0b1020; }
@media (max-width: 960px) {
  .canvas-col { order: 1; }
  .list-col { order: 2; }
}
</style>

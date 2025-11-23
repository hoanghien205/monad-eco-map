<template>
  <v-app>
    <v-main>
      <v-container fluid class="fill-height dark-bg">
        <v-row>
          <v-col cols="12">
            <v-card elevation="10" class="pa-4">
              <v-card-title class="text-h5">Monad 3D Ecosystem Explorer</v-card-title>
              <ThreeScene />
              <DappList />
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<script setup>
import ThreeScene from '../components/ThreeScene.vue'
import DappList from '../components/DappList.vue'
</script>

<style>
.dark-bg {
  background: linear-gradient(#050505, #0a0a2a);
}
</style>

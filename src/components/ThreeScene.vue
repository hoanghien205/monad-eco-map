<template>
  <div ref="container" class="three-wrapper"></div>
</template>

<script setup>
import * as THREE from 'three'
import { onMounted, ref } from 'vue'

const container = ref(null)

onMounted(() => {
  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0x050510)

  const camera = new THREE.PerspectiveCamera(
    60,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    1000
  )
  camera.position.set(0, 2, 6)

  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  container.value.appendChild(renderer.domElement)

  // Light
  const light = new THREE.PointLight(0xffffff, 1)
  light.position.set(10, 10, 10)
  scene.add(light)

  const ambient = new THREE.AmbientLight(0x404040)
  scene.add(ambient)

  // Core planet (ecosystem center)
  const planetGeo = new THREE.SphereGeometry(1, 32, 32)
  const planetMat = new THREE.MeshStandardMaterial({
    color: 0x6f42c1,
    emissive: 0x220033
  })
  const planet = new THREE.Mesh(planetGeo, planetMat)
  scene.add(planet)

  // Orbiting nodes (dapps)
  const nodes = []
  for (let i = 0; i < 8; i++) {
    const geo = new THREE.SphereGeometry(0.2, 16, 16)
    const mat = new THREE.MeshStandardMaterial({ color: 0x00ffff })
    const node = new THREE.Mesh(geo, mat)

    node.userData.angle = (i / 8) * Math.PI * 2
    scene.add(node)
    nodes.push(node)
  }

  function animate() {
    requestAnimationFrame(animate)

    planet.rotation.y += 0.002

    nodes.forEach((node, index) => {
      node.userData.angle += 0.01
      node.position.x = Math.cos(node.userData.angle) * 3
      node.position.z = Math.sin(node.userData.angle) * 3
      node.position.y = Math.sin(node.userData.angle + index) * 0.5
    })

    renderer.render(scene, camera)
  }

  animate()

  window.addEventListener('resize', () => {
    camera.aspect = container.value.clientWidth / container.value.clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  })
})
</script>

<style scoped>
.three-wrapper {
  width: 100%;
  height: 600px;
  border-radius: 12px;
  overflow: hidden;
}
</style>

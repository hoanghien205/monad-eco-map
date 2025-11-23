<template>
  <div ref="container" class="three-wrapper"></div>
</template>

<script setup>
import * as THREE from "three";
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';  // NEW: Import for controls
import { onMounted, onBeforeUnmount, ref, watch, nextTick } from "vue";

const props = defineProps({
  dapps: { type: Array, default: () => [] },
  highlightId: { type: [String, Number, null], default: null }
});

const emit = defineEmits(["select-dapp"]);
const container = ref(null);

let scene, camera, renderer, raycaster, mouse, controls;  // UPDATED: Add controls
let nodes = [];
let lines = [];  // NEW: For connections
let rafId = null;

function createDappNode(dapp) {
  const geo = new THREE.SphereGeometry(1, 32, 32);
  const mat = new THREE.MeshStandardMaterial({
    color: new THREE.Color(0x6677cc),
    emissive: new THREE.Color(0x000000),
    roughness: 0.6,
    metalness: 0.1
  });

  const mesh = new THREE.Mesh(geo, mat);
  mesh.userData.id = dapp.id;
  mesh.position.set(
    (Math.random() - 0.5) * 12,
    Math.random() * 4 + 1,
    (Math.random() - 0.5) * 12
  );
  mesh.userData.baseY = mesh.position.y;
  scene.add(mesh);

  if (dapp.logo) {
    const spriteMap = new THREE.TextureLoader().load(dapp.logo);
    const spriteMat = new THREE.SpriteMaterial({ map: spriteMap });
    const sprite = new THREE.Sprite(spriteMat);
    sprite.scale.set(2.5, 2.5, 1);
    sprite.position.set(0, 1.8, 0);
    mesh.add(sprite);
  }

  // NEW: Add particles around node
  addParticlesToNode(mesh);

  return mesh;
}

// NEW: Function for node particles
function addParticlesToNode(mesh) {
  const particleGeo = new THREE.BufferGeometry();
  const particleCount = 8;
  const positions = new Float32Array(particleCount * 3);
  for (let i = 0; i < particleCount; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 2;
    positions[i * 3 + 1] = (Math.random() - 0.5) * 2;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 2;
  }
  particleGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  const particleMat = new THREE.PointsMaterial({ color: 0x3399ff, size: 0.1, transparent: true });
  const particles = new THREE.Points(particleGeo, particleMat);
  mesh.add(particles);
  mesh.userData.particles = particles;  // For animation
}

function createHighlightRing(mesh) {
  const ringGeo = new THREE.RingGeometry(1.3, 1.6, 64);
  const ringMat = new THREE.MeshBasicMaterial({
    color: 0xffff66,
    side: THREE.DoubleSide,
    transparent: true,
    opacity: 0.35
  });
  const ring = new THREE.Mesh(ringGeo, ringMat);
  ring.rotation.x = -Math.PI / 2;
  ring.position.y = -0.8;
  ring.visible = false;
  mesh.add(ring);
  return ring;
}

function buildScene() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x0b1020);

  camera = new THREE.PerspectiveCamera(60, container.value.clientWidth / container.value.clientHeight, 0.1, 1000);
  camera.position.set(0, 6, 18);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
  renderer.setPixelRatio(window.devicePixelRatio || 1);
  renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  container.value.appendChild(renderer.domElement);

  const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 0.6);
  scene.add(hemi);

  const dir = new THREE.DirectionalLight(0xffffff, 0.8);
  dir.position.set(5, 10, 7);
  scene.add(dir);

  raycaster = new THREE.Raycaster();
  mouse = new THREE.Vector2();

  nodes = props.dapps.map((d) => {
    const mesh = createDappNode(d);
    const ring = createHighlightRing(mesh);
    return { ...d, mesh, ring };
  });

  // NEW: Add connections between same category nodes
  addNodeConnections();

  addStars();

  // NEW: Add OrbitControls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;

  onResize();
  window.addEventListener("resize", onResize);
  renderer.domElement.addEventListener("pointermove", onPointerMove);
  renderer.domElement.addEventListener("click", onClick);

  animate();
}

// NEW: Function for connections
function addNodeConnections() {
  const categories = {};
  nodes.forEach(n => {
    if (!categories[n.category]) categories[n.category] = [];
    categories[n.category].push(n.mesh);
  });

  Object.values(categories).forEach(group => {
    if (group.length > 1) {
      for (let i = 0; i < group.length; i++) {
        for (let j = i + 1; j < group.length; j++) {
          const points = [group[i].position, group[j].position];
          const lineGeo = new THREE.BufferGeometry().setFromPoints(points);
          const lineMat = new THREE.LineDashedMaterial({ color: 0x556699, dashSize: 0.5, gapSize: 0.3 });
          const line = new THREE.Line(lineGeo, lineMat);
          line.computeLineDistances();
          scene.add(line);
          lines.push(line);
        }
      }
    }
  });
}

function addStars() {
  // ... (giữ nguyên)
}

function onResize() {
  // ... (giữ nguyên)
}

let lastPointer = { x: 0, y: 0 };
function onPointerMove(e) {
  // ... (giữ nguyên)
}

function onClick() {
  // ... (giữ nguyên)
}

function animate() {
  rafId = requestAnimationFrame(animate);
  controls.update();  // NEW: Update controls

  const time = performance.now();

  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(nodes.map(n => n.mesh), true);
  let hoveredId = null;
  if (intersects.length > 0) {
    hoveredId = intersects[0].object.userData.id;
  }

  nodes.forEach(n => {
    n.mesh.position.y = n.mesh.userData.baseY + Math.sin((time * 0.001) + n.mesh.userData.baseY) * 0.15;

    // NEW: Animate particles
    if (n.mesh.userData.particles) {
      n.mesh.userData.particles.rotation.y += 0.005;
    }

    const isActive = String(n.id) === String(props.highlightId);
    const isHovered = hoveredId !== null && String(hoveredId) === String(n.id);

    if (isActive) {
      const pulse = 1 + Math.sin(time * 0.006) * 0.06;
      n.mesh.scale.set(1.6 * pulse, 1.6 * pulse, 1.6 * pulse);
      n.mesh.material.emissive.lerp(new THREE.Color(0xffff66), 0.2);
      n.mesh.material.color.lerp(new THREE.Color(0xffffff), 0.05);
      if (n.ring) {
        n.ring.visible = true;
        n.ring.rotation.z += 0.02;
        n.ring.material.opacity = 0.45 + Math.sin(time * 0.01) * 0.05;
      }
      n.mesh.position.z = THREE.MathUtils.lerp(n.mesh.position.z, 0, 0.02);
    } else {
      n.mesh.material.emissive.lerp(new THREE.Color(0x000000), 0.1);
      n.mesh.material.color.lerp(new THREE.Color(0x556699), 0.03);
      n.mesh.scale.lerp(new THREE.Vector3(1,1,1), 0.08);
      if (n.ring) n.ring.visible = false;
      n.mesh.position.z = THREE.MathUtils.lerp(n.mesh.position.z, n.mesh.position.z, 0.02);
    }

    if (!isActive && isHovered) {
      n.mesh.material.emissive.lerp(new THREE.Color(0x3399ff), 0.15);
      n.mesh.scale.lerp(new THREE.Vector3(1.25,1.25,1.25), 0.1);
    }
  });

  // NEW: Update line positions if nodes move
  let lineIndex = 0;
  Object.values(groupByCategory(nodes)).forEach(group => {
    if (group.length > 1) {
      for (let i = 0; i < group.length; i++) {
        for (let j = i + 1; j < group.length; j++) {
          const points = [group[i].mesh.position, group[j].mesh.position];
          lines[lineIndex].geometry.setFromPoints(points);
          lines[lineIndex].computeLineDistances();
          lineIndex++;
        }
      }
    }
  });

  renderer.render(scene, camera);
}

// NEW: Helper to group by category
function groupByCategory(nodes) {
  const categories = {};
  nodes.forEach(n => {
    if (!categories[n.category]) categories[n.category] = [];
    categories[n.category].push(n);
  });
  return categories;
}

watch(() => props.highlightId, (newId) => {
  nodes.forEach(n => {
    if (String(n.id) === String(newId)) {
      if (n.ring) n.ring.visible = true;
      // NEW: Zoom camera to active node
      const targetPos = n.mesh.position.clone().add(new THREE.Vector3(0, 2, 5));
      camera.position.lerp(targetPos, 0.1);
      controls.target.lerp(n.mesh.position, 0.1);
    } else {
      if (n.ring) n.ring.visible = false;
    }
  });
});

onMounted(async () => {
  await nextTick();
  if (!container.value) return;
  buildScene();
});

onBeforeUnmount(() => {
  // ... (giữ nguyên, thêm dispose lines)
  lines.forEach(l => {
    l.geometry.dispose();
    l.material.dispose();
    scene.remove(l);
  });
  if (controls) controls.dispose();
});
</script>

<style scoped>
.three-wrapper {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  position: relative;
  background: radial-gradient(circle at center, #0b1020 0%, #050a18 100%);  /* UPDATED: Radial gradient */
  border-left: 1px solid #1a1f3a;  /* NEW: Border separator */
}
</style>

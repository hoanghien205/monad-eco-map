<template>
  <div ref="container" class="three-wrapper"></div>
</template>

<script setup>
import * as THREE from "three";
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { onMounted, onBeforeUnmount, ref, watch, nextTick } from "vue";

const props = defineProps({
  dapps: { type: Array, default: () => [] },
  highlightId: { type: [String, Number, null], default: null },
  filterState: { type: Object, default: () => ({ filter: "", category: null }) }
});

const emit = defineEmits(["select-dapp"]);
const container = ref(null);

let scene, camera, renderer, controls, raycaster, mouse;
let nodes = [], lines = [];
let rafId = null;

// === Tạo text label ===
function createTextSprite(text) {
  const canvas = document.createElement('canvas');
  canvas.width = 512; canvas.height = 128;
  const ctx = canvas.getContext('2d');
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.font = 'bold 48px Inter, Arial';
  ctx.fillStyle = '#ffffff';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(text.substring(0, 18), canvas.width / 2, canvas.height / 2);
  const texture = new THREE.CanvasTexture(canvas);
  const mat = new THREE.SpriteMaterial({ map: texture, transparent: true });
  const sprite = new THREE.Sprite(mat);
  sprite.scale.set(6, 1.5, 1);
  return sprite;
}

// === Tạo node ===
function createDappNode(dapp) {
  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(0.8, 32, 32),
    new THREE.MeshStandardMaterial({
      color: 0x6677cc,
      emissive: 0x000000,
      roughness: 0.6,
      metalness: 0.1
    })
  );
  mesh.position.set(
    (Math.random() - 0.5) * 25,
    Math.random() * 10 + 3,
    (Math.random() - 0.5) * 25
  );
  mesh.userData = { id: dapp.id, baseY: mesh.position.y, velocity: new THREE.Vector3() };
  scene.add(mesh);

  if (dapp.logo) {
    const tex = new THREE.TextureLoader().load(dapp.logo);
    const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: tex }));
    sprite.scale.set(2.5, 2.5, 1);
    sprite.position.y = 1.8;
    mesh.add(sprite);
  }

  const label = createTextSprite(dapp.name);
  label.position.y = 2.8;
  mesh.add(label);
  mesh.userData.label = label;

  return mesh;
}

// === Cập nhật nodes theo filter ===
function updateNodes() {
  nodes.forEach(n => scene.remove(n.mesh));
  lines.forEach(l => scene.remove(l));
  nodes = []; lines = [];

  let list = props.dapps;
  if (props.filterState?.filter) {
    const q = props.filterState.filter.toLowerCase();
    list = list.filter(d => (d.name + d.description + d.category).toLowerCase().includes(q));
  }
  if (props.filterState?.category) {
    list = list.filter(d => d.category === props.filterState.category);
  }

  list.slice(0, 60).forEach(d => {
    const mesh = createDappNode(d);
    nodes.push({ mesh, id: d.id });
  });

  // Tạo lines nối cùng category
  const catMap = {};
  nodes.forEach(n => {
    const cat = props.dapps.find(d => d.id === n.id)?.category || "";
    if (!catMap[cat]) catMap[cat] = [];
    catMap[cat].push(n.mesh);
  });
  Object.values(catMap).forEach(group => {
    if (group.length > 1) {
      for (let i = 0; i < group.length; i++) {
        for (let j = i + 1; j < group.length; j++) {
          const geometry = new THREE.BufferGeometry().setFromPoints([group[i].position, group[j].position]);
          const material = new THREE.LineDashedMaterial({ color: 0x556699, dashSize: 0.5, gapSize: 0.3 });
          const line = new THREE.Line(geometry, material);
          line.computeLineDistances();
          scene.add(line);
          lines.push({ line, p1: group[i], p2: group[j] });
        }
      }
    }
  });
}

// === Animation loop ===
function animate() {
  rafId = requestAnimationFrame(animate);
  controls.update();

  const time = performance.now() * 0.001;

  // Repulsion + floating
  nodes.forEach((n1, i) => {
    nodes.forEach((n2, j) => {
      if (i !== j) {
        const diff = n1.mesh.position.clone().sub(n2.mesh.position);
        const d = diff.length();
        if (d < 4 && d > 0) {
          const f = diff.normalize().multiplyScalar(0.08);
          n1.mesh.userData.velocity.add(f);
          n2.mesh.userData.velocity.sub(f);
        }
      }
    });
    n1.mesh.position.add(n1.mesh.userData.velocity.multiplyScalar(0.92));
    n1.mesh.position.y = n1.mesh.userData.baseY + Math.sin(time * 2 + n1.mesh.position.x) * 0.3;
  });

  // Hover detect
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(nodes.map(n => n.mesh), true);
  const hoveredId = hits.length > 0 ? hits[0].object.userData.id : null;

  // Highlight + hover
  nodes.forEach(n => {
    const active = String(n.id) === String(props.highlightId);
    const hovered = String(n.id) === String(hoveredId);

    if (active) {
      n.mesh.scale.setScalar(2 + Math.sin(time * 5) * 0.1);
      n.mesh.material.emissive.set(0xffff66);
      n.mesh.userData.label.scale.set(7.5, 1.9, 1);
    } else {
      n.mesh.scale.lerp(new THREE.Vector3(1,1,1), 0.1);
      n.mesh.material.emissive.lerp(new THREE.Color(0x000000), 0.1);
      n.mesh.userData.label.scale.lerp(new THREE.Vector3(6,1.5,1), 0.1);
      if (hovered) {
        n.mesh.scale.setScalar(1.4);
        n.mesh.material.emissive.set(0x3399ff);
      }
    }
  });

  // Update lines
  lines.forEach(l => {
    l.line.geometry.setFromPoints([l.p1.position, l.p2.position]);
    l.line.computeLineDistances();
  });

  renderer.render(scene, camera);
}

// === Khởi tạo scene (fix lỗi clientWidth) ===
function initScene() {
  if (!container.value) return false;

  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x0b1020);

  camera = new THREE.PerspectiveCamera(60, container.value.clientWidth / container.value.clientHeight, 0.1, 1000);
  camera.position.set(0, 8, 30);

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  container.value.appendChild(renderer.domElement);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 0.6));
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(5, 10, 7);
  scene.add(dirLight);

  raycaster = new THREE.Raycaster();
  mouse = new THREE.Vector2();

  // Events
  renderer.domElement.addEventListener("pointermove", e => {
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  });

  renderer.domElement.addEventListener("click", () => {
    raycaster.setFromCamera(mouse, camera);
    const hit = raycaster.intersectObjects(nodes.map(n => n.mesh), true);
    if (hit.length > 0 && hit[0].object.userData.id) {
      emit("select-dapp", hit[0].object.userData.id);
    }
  });

  window.addEventListener("resize", () => {
    if (!container.value) return;
    camera.aspect = container.value.clientWidth / container.value.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  });

  updateNodes();
  animate();
  return true;
}

// === onMounted – đảm bảo container đã sẵn sàng ===
onMounted(() => {
  const tryInit = () => {
    if (initScene()) return;
    requestAnimationFrame(tryInit);
  };
  nextTick(() => requestAnimationFrame(tryInit));
});

// Watch filter & highlight
watch(() => props.filterState, updateNodes, { deep: true });
watch(() => props.highlightId, () => {
  const node = nodes.find(n => String(n.id) === String(props.highlightId));
  if (node) {
    const target = node.mesh.position.clone().add(new THREE.Vector3(0, 3, 10));
    camera.position.lerp(target, 0.1);
    controls.target.lerp(node.mesh.position, 0.1);
  }
});

onBeforeUnmount(() => {
  if (rafId) cancelAnimationFrame(rafId);
  if (renderer) renderer.dispose();
});
</script>

<style scoped>
.three-wrapper {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background: radial-gradient(circle at center, #0b1020 0%, #050a18 100%);
  border-left: 1px solid #1a1f3a;
}
</style>

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
let nodes = [], lines = [], clusterLabels = [], stars;
let rafId = null;

// === Tạo text label ===
function createTextSprite(text, scale = [6, 1.5, 1], color = "#ffffff") {
  const canvas = document.createElement('canvas');
  canvas.width = 512; canvas.height = 128;
  const ctx = canvas.getContext('2d');
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.font = 'bold 48px Inter, Arial';
  ctx.fillStyle = color;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(text.substring(0, 20), canvas.width / 2, canvas.height / 2);
  const texture = new THREE.CanvasTexture(canvas);
  const mat = new THREE.SpriteMaterial({ map: texture, transparent: true });
  const sprite = new THREE.Sprite(mat);
  sprite.scale.set(...scale);
  return sprite;
}

// === Màu theo category ===
function getCategoryColor(cat) {
  const colors = {
    DeFi: 0x8b5cf6,     // tím
    Gaming: 0x10b981,    // xanh lá
    NFT: 0xff6b6b,       // đỏ cam
    Social: 0x3b82f6,    // xanh dương
    Tools: 0xf59e0b,     // vàng
    Infrastructure: 0x06b6d4 // cyan
  };
  return colors[cat] || 0x6677cc;
}

// === Tạo node ===
function createDappNode(dapp, center, groupSize) {
  const color = getCategoryColor(dapp.category || "Unknown");
  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(0.8, 32, 32),
    new THREE.MeshStandardMaterial({
      color: color,
      emissive: color,
      emissiveIntensity: 0.2,
      roughness: 0.4,
      metalness: 0.3
    })
  );

  const radius = Math.min(6, 3 + Math.sqrt(groupSize) * 0.8);
  const angle = Math.random() * Math.PI * 2;
  const height = (Math.random() - 0.5) * 3;

  mesh.position.set(
    center.x + Math.cos(angle) * radius,
    center.y + height,
    center.z + Math.sin(angle) * radius
  );

  mesh.userData = { id: dapp.id, baseY: mesh.position.y, velocity: new THREE.Vector3() };
  scene.add(mesh);

  // Logo
  if (dapp.logo) {
    const tex = new THREE.TextureLoader().load(dapp.logo);
    const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: tex }));
    sprite.scale.set(2.5, 2.5, 1);
    sprite.position.y = 1.8;
    mesh.add(sprite);
  }

  // Label tên dapp
  const label = createTextSprite(dapp.name, [6, 1.5, 1]);
  label.position.y = 2.8;
  mesh.add(label);
  mesh.userData.label = label;

  return mesh;
}

// === Tạo background stars ===
function createStars() {
  const geometry = new THREE.BufferGeometry();
  const vertices = [];
  for (let i = 0; i < 3000; i++) {
    const x = (Math.random() - 0.5) * 200;
    const y = (Math.random() - 0.5) * 200;
    const z = (Math.random() - 0.5) * 200;
    vertices.push(x, y, z);
  }
  geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
  const material = new THREE.PointsMaterial({ color: 0xffffff, size: 0.7, transparent: true, opacity: 0.8 });
  stars = new THREE.Points(geometry, material);
  scene.add(stars);
}

// === Cập nhật toàn bộ scene ===
function updateNodes() {
  // Xóa cũ
  nodes.forEach(n => scene.remove(n.mesh));
  lines.forEach(l => scene.remove(l.line));
  clusterLabels.forEach(l => scene.remove(l));
  nodes = []; lines = []; clusterLabels = [];

  let list = props.dapps;
  if (props.filterState?.filter) {
    const q = props.filterState.filter.toLowerCase();
    list = list.filter(d => (d.name + " " + d.description + " " + d.category).toLowerCase().includes(q));
  }
  if (props.filterState?.category) {
    list = list.filter(d => d.category === props.filterState.category);
  }

  const uniqueCats = [...new Set(list.map(d => d.category))].sort();
  const catCenters = {};
  const spacing = Math.max(40, uniqueCats.length * 18);

  uniqueCats.forEach((cat, i) => {
    const cols = Math.ceil(Math.sqrt(uniqueCats.length));
    const row = Math.floor(i / cols);
    const col = i % cols;
    catCenters[cat] = new THREE.Vector3(
      (col - (cols - 1) / 2) * spacing,
      5,
      (row - (uniqueCats.length > 6 ? 1 : 0)) * 40
    );
  });

  // Tạo nodes
  list.slice(0, 80).forEach(d => {
    const center = catCenters[d.category || "Unknown"] || new THREE.Vector3(0, 5, 0);
    const groupSize = list.filter(x => x.category === d.category).length;
    const mesh = createDappNode(d, center, groupSize);
    nodes.push({ mesh, id: d.id, category: d.category });
  });

  // Label cho từng cụm
  Object.entries(catCenters).forEach(([cat, pos]) => {
    const label = createTextSprite(cat, [14, 3.5, 1], "#ffd54f");
    label.position.copy(pos).add(new THREE.Vector3(0, 10, 0));
    scene.add(label);
    clusterLabels.push(label);
  });

  // Lines thông minh
  const catMap = {};
  nodes.forEach(n => {
    const c = n.category || "Unknown";
    if (!catMap[c]) catMap[c] = [];
    catMap[c].push(n.mesh);
  });

  Object.values(catMap).forEach(group => {
    if (group.length <= 1) return;
    if (group.length <= 8) {
      // Nhóm nhỏ: nối full
      for (let i = 0; i < group.length; i++) {
        for (let j = i + 1; j < group.length; j++) {
          addLine(group[i], group[j], 0xaaaaaa, 0.3);
        }
      }
    } else {
      // Nhóm lớn: chỉ nối dạng lưới nhẹ (mỗi node nối 3-4 node gần nhất)
      group.forEach((node, idx) => {
        const distances = group.map((n, i) => ({ dist: node.position.distanceTo(n.position), idx: i }))
          .filter(d => d.idx !== idx)
          .sort((a, b) => a.dist - b.dist)
          .slice(0, 4);
        distances.forEach(d => {
          if (d.idx > idx) addLine(node, group[d.idx], 0x556699, 0.2);
        });
      });
    }
  });
}

function addLine(p1, p2, color, opacity) {
  const geometry = new THREE.BufferGeometry().setFromPoints([p1.position, p2.position]);
  const material = new THREE.LineBasicMaterial({
    color,
    transparent: true,
    opacity
  });
  const line = new THREE.Line(geometry, material);
  scene.add(line);
  lines.push({ line, p1, p2 });
}

// === Animation loop ===
function animate() {
  rafId = requestAnimationFrame(animate);
  controls.update();

  const time = performance.now() * 0.001;

  // Force + floating
  nodes.forEach(n1 => {
    nodes.forEach(n2 => {
      if (n1 !== n2) {
        const diff = n1.mesh.position.clone().sub(n2.mesh.position);
        const d = diff.length();
        if (d < 6 && d > 0) {
          const force = diff.normalize().multiplyScalar(0.12);
          n1.mesh.userData.velocity.add(force);
          n2.mesh.userData.velocity.sub(force);
        }
      }
    });
    n1.mesh.userData.velocity.multiplyScalar(0.9);
    n1.mesh.position.add(n1.mesh.userData.velocity);
    n1.mesh.position.y = n1.mesh.userData.baseY + Math.sin(time * 1.5 + n1.mesh.position.x * 0.5) * 0.4;
  });

  // Hover
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(nodes.map(n => n.mesh), true);
  const hoveredId = hits.length > 0 ? hits[0].object.userData.id : null;

  nodes.forEach(n => {
    const active = String(n.id) === String(props.highlightId);
    const hovered = String(n.id) === String(hoveredId);

    if (active) {
      n.mesh.scale.setScalar(2.2 + Math.sin(time * 6) * 0.15);
      n.mesh.material.emissiveIntensity = 0.8;
      n.mesh.userData.label.scale.set(8, 2, 1);
    } else {
      n.mesh.scale.lerp(new THREE.Vector3(1, 1, 1), 0.15);
      n.mesh.material.emissiveIntensity = n.mesh.material.emissiveIntensity * 0.9 + 0.2 * 0.2;
      n.mesh.userData.label.scale.lerp(new THREE.Vector3(6, 1.5, 1), 0.15);
      if (hovered) {
        n.mesh.scale.setScalar(1.5);
        n.mesh.material.emissiveIntensity = 0.6;
      }
    }
  });

  // Update lines
  lines.forEach(l => l.line.geometry.setFromPoints([l.p1.position, l.p2.position]));

  // Stars xoay nhẹ
  if (stars) stars.rotation.y += 0.0002;

  renderer.render(scene, camera);
}

// === Init scene ===
function initScene() {
  if (!container.value) return false;

  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x050818);

  camera = new THREE.PerspectiveCamera(60, container.value.clientWidth / container.value.clientHeight, 0.1, 1000);
  camera.position.set(0, 15, 60);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  renderer.outputEncoding = THREE.sRGBEncoding;
  container.value.appendChild(renderer.domElement);

  // Controls - cho zoom thoải mái
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.08;
  controls.minDistance = 20;
  controls.maxDistance = 200;
  controls.maxPolarAngle = Math.PI / 2.2;

  // Ánh sáng
  scene.add(new THREE.AmbientLight(0x404060, 1));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(10, 20, 10);
  scene.add(dirLight);

  // Stars background
  createStars();

  raycaster = new THREE.Raycaster();
  mouse = new THREE.Vector2();

  renderer.domElement.addEventListener("pointermove", e => {
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  });

  renderer.domElement.addEventListener("click", () => {
    raycaster.setFromCamera(mouse, camera);
    const hit = raycaster.intersectObjects(nodes.map(n => n.mesh), true);
    if (hit.length > 0 && hit[0].object.userData?.id) {
      emit("select-dapp", hit[0].object.userData.id);
    }
  });

  window.addEventListener("resize", () => {
    camera.aspect = container.value.clientWidth / container.value.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  });

  updateNodes();
  animate();
  return true;
}

onMounted(() => {
  nextTick(() => {
    const tryInit = () => initScene() || requestAnimationFrame(tryInit);
    requestAnimationFrame(tryInit);
  });
});

watch(() => props.filterState, updateNodes, { deep: true });

// Khi chọn dapp → camera bay tới nhẹ nhàng, vẫn cho phép zoom/xoay tay
watch(() => props.highlightId, () => {
  const node = nodes.find(n => String(n.id) === String(props.highlightId));
  if (node) {
    const targetPos = node.mesh.position.clone().add(new THREE.Vector3(0, 8, 25));
    controls.target.copy(node.mesh.position);
    camera.position.lerp(targetPos, 0.3);
  }
});

onBeforeUnmount(() => {
  if (rafId) cancelAnimationFrame(rafId);
  renderer?.dispose();
});
</script>

<style scoped>
.three-wrapper {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background: linear-gradient(135deg, #0f0c29, #302b63, #24243e);
  border-left: 1px solid #334466;
  position: relative;
}
.three-wrapper::before {
  content: "";
  position: absolute;
  inset: 0;
  background: radial-gradient(circle at 30% 30%, rgba(100, 150, 255, 0.15), transparent 50%),
  radial-gradient(circle at 70% 80%, rgba(255, 100, 200, 0.1), transparent 50%);
  pointer-events: none;
}
</style>

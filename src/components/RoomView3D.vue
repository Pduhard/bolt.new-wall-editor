<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

const props = defineProps<{
  walls: { x1: number; y1: number; x2: number; y2: number }[];
  roomWidth: number;
  roomHeight: number;
}>();

const canvasRef = ref<HTMLCanvasElement | null>(null);

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;

function init() {
  if (!canvasRef.value) return;

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, canvasRef.value.clientWidth / canvasRef.value.clientHeight, 0.1, 1000);
  renderer = new THREE.WebGLRenderer({ canvas: canvasRef.value, antialias: true });
  renderer.setSize(canvasRef.value.clientWidth, canvasRef.value.clientHeight);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.25;
  controls.enableZoom = true;

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
  directionalLight.position.set(0, 1, 0);
  scene.add(directionalLight);

  camera.position.set(props.roomWidth / 2, props.roomHeight / 2, Math.max(props.roomWidth, props.roomHeight));
  controls.target.set(props.roomWidth / 2, props.roomHeight / 2, 0);

  createRoom();
  animate();
}

function createRoom() {
  // Clear existing walls
  scene.children = scene.children.filter(child => !(child instanceof THREE.Mesh));

  // Add floor
  const floorGeometry = new THREE.PlaneGeometry(props.roomWidth, props.roomHeight);
  const floorMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc });
  const floor = new THREE.Mesh(floorGeometry, floorMaterial);
  floor.rotation.x = -Math.PI / 2;
  floor.position.set(props.roomWidth / 2, 0, props.roomHeight / 2);
  scene.add(floor);

  // Add walls
  const wallMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff });
  props.walls.forEach(wall => {
    const wallLength = Math.sqrt(Math.pow(wall.x2 - wall.x1, 2) + Math.pow(wall.y2 - wall.y1, 2));
    const wallGeometry = new THREE.BoxGeometry(wallLength, 100, 10);
    const wallMesh = new THREE.Mesh(wallGeometry, wallMaterial);

    const angle = Math.atan2(wall.y2 - wall.y1, wall.x2 - wall.x1);
    wallMesh.position.set((wall.x1 + wall.x2) / 2, 50, props.roomHeight - (wall.y1 + wall.y2) / 2);
    wallMesh.rotation.y = -angle;

    scene.add(wallMesh);
  });
}

function animate() {
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
}

watch(() => props.walls, createRoom, { deep: true });

onMounted(() => {
  init();
  window.addEventListener('resize', onWindowResize, false);
});

function onWindowResize() {
  if (!canvasRef.value) return;
  camera.aspect = canvasRef.value.clientWidth / canvasRef.value.clientHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(canvasRef.value.clientWidth, canvasRef.value.clientHeight);
}
</script>

<template>
  <canvas ref="canvasRef" style="width: 100%; height: 300px;"></canvas>
</template>
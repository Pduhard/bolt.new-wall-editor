<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import RoomView3D from './RoomView3D.vue';

const canvasRef = ref<HTMLCanvasElement | null>(null);
const ctx = ref<CanvasRenderingContext2D | null>(null);

const roomWidth = 500;
const roomHeight = 400;
const wallThickness = 10;
const handleRadius = 8;
const snapDistance = 15;

interface Wall {
  id: number;
  x1: number;
  y1: number;
  x2: number;
  y2: number;
  linkedStart: number | null;
  linkedEnd: number | null;
}

const walls = ref<Wall[]>([
  { id: 1, x1: 0, y1: 0, x2: roomWidth, y2: 0, linkedStart: null, linkedEnd: null },
  { id: 2, x1: roomWidth, y1: 0, x2: roomWidth, y2: roomHeight, linkedStart: null, linkedEnd: null },
  { id: 3, x1: roomWidth, y1: roomHeight, x2: 0, y2: roomHeight, linkedStart: null, linkedEnd: null },
  { id: 4, x1: 0, y1: roomHeight, x2: 0, y2: 0, linkedStart: null, linkedEnd: null },
]);

const selectedWall = ref<Wall | null>(null);
const isDragging = ref(false);
const draggedHandle = ref<'start' | 'end' | 'middle' | null>(null);
const nextWallId = ref(5);

const selectedWallDimensions = computed(() => {
  if (!selectedWall.value) return null;
  const wall = selectedWall.value;
  const length = Math.sqrt(
    Math.pow(wall.x2 - wall.x1, 2) + Math.pow(wall.y2 - wall.y1, 2)
  );
  return {
    length: length.toFixed(2),
    angle: (Math.atan2(wall.y2 - wall.y1, wall.x2 - wall.x1) * (180 / Math.PI)).toFixed(2),
  };
});

const linkedWalls = computed(() => {
  if (!selectedWall.value) return [];
  return walls.value.filter(
    wall => wall.id === selectedWall.value?.linkedStart || 
            wall.id === selectedWall.value?.linkedEnd ||
            wall.linkedStart === selectedWall.value?.id ||
            wall.linkedEnd === selectedWall.value?.id
  );
});

function initCanvas() {
  if (!canvasRef.value) return;
  
  ctx.value = canvasRef.value.getContext('2d');
  canvasRef.value.width = roomWidth;
  canvasRef.value.height = roomHeight;
  
  drawRoom();
}

function drawRoom() {
  if (!ctx.value) return;

  ctx.value.clearRect(0, 0, roomWidth, roomHeight);
  
  // Draw floor
  ctx.value.fillStyle = '#f0f0f0';
  ctx.value.fillRect(0, 0, roomWidth, roomHeight);

  // Draw walls
  walls.value.forEach((wall) => {
    const isAffected = isDragging.value && (
      wall === selectedWall.value ||
      wall.linkedStart === selectedWall.value?.id ||
      wall.linkedEnd === selectedWall.value?.id
    );
    ctx.value!.strokeStyle = isAffected ? '#ff0000' : '#333';
    ctx.value!.lineWidth = wallThickness;
    ctx.value!.beginPath();
    ctx.value!.moveTo(wall.x1, wall.y1);
    ctx.value!.lineTo(wall.x2, wall.y2);
    ctx.value!.stroke();
    
    // Draw wall handles
    const startColor = isDragging.value && draggedHandle.value === 'start' && wall === selectedWall.value ? '#00ff00' : '#0000ff';
    const endColor = isDragging.value && draggedHandle.value === 'end' && wall === selectedWall.value ? '#00ff00' : '#0000ff';
    
    ctx.value!.fillStyle = startColor;
    ctx.value!.beginPath();
    ctx.value!.arc(wall.x1, wall.y1, handleRadius, 0, Math.PI * 2);
    ctx.value!.fill();
    
    ctx.value!.fillStyle = endColor;
    ctx.value!.beginPath();
    ctx.value!.arc(wall.x2, wall.y2, handleRadius, 0, Math.PI * 2);
    ctx.value!.fill();
  });
}

function handleMouseDown(event: MouseEvent) {
  const rect = canvasRef.value!.getBoundingClientRect();
  const x = event.clientX - rect.left;
  const y = event.clientY - rect.top;

  selectedWall.value = null;
  draggedHandle.value = null;

  for (const wall of walls.value) {
    if (isPointNearPoint(x, y, wall.x1, wall.y1)) {
      selectedWall.value = wall;
      draggedHandle.value = 'start';
      break;
    } else if (isPointNearPoint(x, y, wall.x2, wall.y2)) {
      selectedWall.value = wall;
      draggedHandle.value = 'end';
      break;
    } else if (isPointNearLine(x, y, wall)) {
      selectedWall.value = wall;
      draggedHandle.value = 'middle';
      break;
    }
  }

  if (!selectedWall.value) {
    addNewWall(x, y);
  }

  isDragging.value = true;
  drawRoom();
}

function handleMouseMove(event: MouseEvent) {
  if (!isDragging.value || !selectedWall.value) return;

  const rect = canvasRef.value!.getBoundingClientRect();
  const x = event.clientX - rect.left;
  const y = event.clientY - rect.top;

  if (draggedHandle.value === 'start') {
    updateWallStart(selectedWall.value, x, y);
  } else if (draggedHandle.value === 'end') {
    updateWallEnd(selectedWall.value, x, y);
  } else if (draggedHandle.value === 'middle') {
    const dx = x - (selectedWall.value.x1 + selectedWall.value.x2) / 2;
    const dy = y - (selectedWall.value.y1 + selectedWall.value.y2) / 2;
    updateWallStart(selectedWall.value, selectedWall.value.x1 + dx, selectedWall.value.y1 + dy);
    updateWallEnd(selectedWall.value, selectedWall.value.x2 + dx, selectedWall.value.y2 + dy);
  }

  drawRoom();
}

function updateWallStart(wall: Wall, x: number, y: number) {
  wall.x1 = x;
  wall.y1 = y;
  if (wall.linkedStart !== null) {
    const linkedWall = walls.value.find(w => w.id === wall.linkedStart);
    if (linkedWall) {
      if (linkedWall.linkedStart === wall.id) {
        linkedWall.x1 = x;
        linkedWall.y1 = y;
      } else if (linkedWall.linkedEnd === wall.id) {
        linkedWall.x2 = x;
        linkedWall.y2 = y;
      }
    }
  }
  checkWallConnections(wall);
}

function updateWallEnd(wall: Wall, x: number, y: number) {
  wall.x2 = x;
  wall.y2 = y;
  if (wall.linkedEnd !== null) {
    const linkedWall = walls.value.find(w => w.id === wall.linkedEnd);
    if (linkedWall) {
      if (linkedWall.linkedStart === wall.id) {
        linkedWall.x1 = x;
        linkedWall.y1 = y;
      } else if (linkedWall.linkedEnd === wall.id) {
        linkedWall.x2 = x;
        linkedWall.y2 = y;
      }
    }
  }
  checkWallConnections(wall);
}

function checkWallConnections(currentWall: Wall) {
  walls.value.forEach(wall => {
    if (wall.id !== currentWall.id) {
      if (isPointNearPoint(currentWall.x1, currentWall.y1, wall.x1, wall.y1, snapDistance)) {
        linkWalls(currentWall, 'start', wall, 'start');
      } else if (isPointNearPoint(currentWall.x1, currentWall.y1, wall.x2, wall.y2, snapDistance)) {
        linkWalls(currentWall, 'start', wall, 'end');
      } else if (isPointNearPoint(currentWall.x2, currentWall.y2, wall.x1, wall.y1, snapDistance)) {
        linkWalls(currentWall, 'end', wall, 'start');
      } else if (isPointNearPoint(currentWall.x2, currentWall.y2, wall.x2, wall.y2, snapDistance)) {
        linkWalls(currentWall, 'end', wall, 'end');
      }
    }
  });
}

function linkWalls(wall1: Wall, end1: 'start' | 'end', wall2: Wall, end2: 'start' | 'end') {
  if (end1 === 'start') {
    wall1.linkedStart = wall2.id;
    wall1.x1 = end2 === 'start' ? wall2.x1 : wall2.x2;
    wall1.y1 = end2 === 'start' ? wall2.y1 : wall2.y2;
  } else {
    wall1.linkedEnd = wall2.id;
    wall1.x2 = end2 === 'start' ? wall2.x1 : wall2.x2;
    wall1.y2 = end2 === 'start' ? wall2.y1 : wall2.y2;
  }

  if (end2 === 'start') {
    wall2.linkedStart = wall1.id;
  } else {
    wall2.linkedEnd = wall1.id;
  }
}

function handleMouseUp() {
  isDragging.value = false;
  draggedHandle.value = null;
  drawRoom();
}

function isPointNearPoint(x1: number, y1: number, x2: number, y2: number, threshold = handleRadius) {
  const dx = x1 - x2;
  const dy = y1 - y2;
  return Math.sqrt(dx * dx + dy * dy) <= threshold;
}

function isPointNearLine(x: number, y: number, wall: Wall, threshold = 10) {
  const A = x - wall.x1;
  const B = y - wall.y1;
  const C = wall.x2 - wall.x1;
  const D = wall.y2 - wall.y1;

  const dot = A * C + B * D;
  const lenSq = C * C + D * D;
  let param = dot / lenSq;

  let xx, yy;

  if (param < 0) {
    xx = wall.x1;
    yy = wall.y1;
  } else if (param > 1) {
    xx = wall.x2;
    yy = wall.y2;
  } else {
    xx = wall.x1 + param * C;
    yy = wall.y1 + param * D;
  }

  const dx = x - xx;
  const dy = y - yy;
  const distance = Math.sqrt(dx * dx + dy * dy);

  return distance < threshold;
}

function addNewWall(x: number, y: number) {
  const newWall: Wall = {
    id: nextWallId.value++,
    x1: x,
    y1: y,
    x2: x + 100,
    y2: y,
    linkedStart: null,
    linkedEnd: null,
  };
  walls.value.push(newWall);
  selectedWall.value = newWall;
}

function removeSelectedWall() {
  if (selectedWall.value) {
    unlinkWall(selectedWall.value);
    walls.value = walls.value.filter(wall => wall !== selectedWall.value);
    selectedWall.value = null;
    drawRoom();
  }
}

function unlinkWall(wall: Wall) {
  if (wall.linkedStart !== null) {
    const linkedWall = walls.value.find(w => w.id === wall.linkedStart);
    if (linkedWall) {
      linkedWall.linkedStart = linkedWall.linkedStart === wall.id ? null : linkedWall.linkedStart;
      linkedWall.linkedEnd = linkedWall.linkedEnd === wall.id ? null : linkedWall.linkedEnd;
    }
    wall.linkedStart = null;
  }
  if (wall.linkedEnd !== null) {
    const linkedWall = walls.value.find(w => w.id === wall.linkedEnd);
    if (linkedWall) {
      linkedWall.linkedStart = linkedWall.linkedStart === wall.id ? null : linkedWall.linkedStart;
      linkedWall.linkedEnd = linkedWall.linkedEnd === wall.id ? null : linkedWall.linkedEnd;
    }
    wall.linkedEnd = null;
  }
}

onMounted(() => {
  initCanvas();
});
</script>

<template>
  <div class="room-configurator">
    <h2>2D Editor</h2>
    <canvas
      ref="canvasRef"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @mouseleave="handleMouseUp"
    ></canvas>
    <div v-if="selectedWall" class="wall-info">
      <h3>Selected Wall</h3>
      <p>Length: {{ selectedWallDimensions.length }} px</p>
      <p>Angle: {{ selectedWallDimensions.angle }}°</p>
      <p>Linked Walls: {{ linkedWalls.length }}</p>
      <button @click="removeSelectedWall">Remove Wall</button>
      <button v-if="linkedWalls.length > 0" @click="unlinkWall(selectedWall)">Unlink Wall</button>
    </div>
    <div class="instructions">
      <p>Click on the canvas to add a new wall.</p>
      <p>Click and drag wall endpoints to stretch and rotate.</p>
      <p>Click and drag the middle of a wall to move it.</p>
      <p>Walls will automatically link when their endpoints are close.</p>
      <p>Select a wall and click "Remove Wall" to delete it.</p>
      <p>Select a linked wall and click "Unlink Wall" to disconnect it.</p>
    </div>
    <h2>3D View</h2>
    <RoomView3D :walls="walls" :roomWidth="roomWidth" :roomHeight="roomHeight" />
  </div>
</template>

<style scoped>
.room-configurator {
  width: 100%;
  max-width: 800px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  align-items: center;
}

canvas {
  border: 1px solid #ccc;
  margin-bottom: 20px;
}

.wall-info {
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  border-radius: 5px;
  padding: 10px;
  margin-bottom: 20px;
}

.instructions {
  text-align: left;
  margin-top: 20px;
  margin-bottom: 20px;
}

button {
  background-color: #ff4444;
  color: white;
  border: none;
  padding: 5px 10px;
  border-radius: 3px;
  cursor: pointer;
  margin-right: 10px;
}

button:hover {
  background-color: #ff6666;
}
</style>
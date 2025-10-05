import * as THREE from "three";
import { OrbitControls } from "jsm/controls/OrbitControls.js";

import getStarfield from "./src/getStarfield.js";
import { getFresnelMat } from "./src/getFresnelMat.js";

// === 📏 Setup ===
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(
  60,                                 // narrower FOV = less distortion
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.set(0, 0, 5);

const renderer = new THREE.WebGLRenderer({
  antialias: true,
  precision: "highp",
  powerPreference: "high-performance", // ⚡ better GPU selection
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // 🧠 prevent over-rendering
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.outputColorSpace = THREE.SRGBColorSpace; // 🧠 use sRGB for color correctness
document.body.appendChild(renderer.domElement);

console.log("Max texture size:", renderer.capabilities.maxTextureSize);

// === 🌍 Earth Group ===
const earthGroup = new THREE.Group();
earthGroup.rotation.z = THREE.MathUtils.degToRad(-23.4); // cleaner
scene.add(earthGroup);

// === 🎮 Controls ===
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// === 🪐 Geometry ===
const geometry = new THREE.IcosahedronGeometry(1, 12);
const loader = new THREE.TextureLoader();

// === 🧠 Texture Loader Helper ===
function loadTexture(path) {
  const tex = loader.load(path, (t) =>
    console.log(`Loaded: ${path} → ${t.image.width}x${t.image.height}`)
  );
  tex.wrapS = tex.wrapT = THREE.ClampToEdgeWrapping;
  tex.minFilter = THREE.LinearFilter;
  tex.colorSpace = THREE.SRGBColorSpace;
  return tex;
}

// === 🌍 Earth Material ===
const earthMat = new THREE.MeshPhongMaterial({
  map: loadTexture("./textures/00_earthmap4k.webp"),
  specularMap: loadTexture("./textures/02_earthspec4k.webp"),
  bumpMap: loadTexture("./textures/01_earthbump4k.webp"),
  bumpScale: 0.04,
});

const earthMesh = new THREE.Mesh(geometry, earthMat);
earthGroup.add(earthMesh);

// === 🌃 City Lights ===
const lightsMesh = new THREE.Mesh(
  geometry,
  new THREE.MeshBasicMaterial({
    map: loadTexture("./textures/03_earthlights4k.webp"),
    blending: THREE.AdditiveBlending,
  })
);
earthGroup.add(lightsMesh);

// === ☁️ Clouds ===
const cloudsMesh = new THREE.Mesh(
  geometry,
  new THREE.MeshStandardMaterial({
    map: loadTexture("./textures/04_earthcloudmap4k.webp"),
    alphaMap: loadTexture("./textures/05_earthcloudmaptrans4k.webp"),
    transparent: true,
    opacity: 0.8,
    blending: THREE.AdditiveBlending,
    depthWrite: false, // 🧠 improves transparency rendering
  })
);
cloudsMesh.scale.setScalar(1.003);
earthGroup.add(cloudsMesh);

// === 🌌 Fresnel Glow ===
const fresnelMat = getFresnelMat();
if (fresnelMat.uniforms?.fresnelColor) {
  fresnelMat.uniforms.fresnelColor.value.set(0x00bfff); // cyan glow
} else if (fresnelMat.color) {
  fresnelMat.color.set(0x00bfff);
}
const glowMesh = new THREE.Mesh(geometry, fresnelMat);
glowMesh.scale.setScalar(1.01);
earthGroup.add(glowMesh);

// === ✨ Stars ===
scene.add(getStarfield({ numStars: 2000 }));

// === ☀️ Lights ===
const sunLight = new THREE.DirectionalLight(0xffffff, 2);
sunLight.position.set(-2, 0.5, 1.5);
scene.add(sunLight);

// Ambient light for subtle fill
scene.add(new THREE.AmbientLight(0x404040, 0.8));

// === 🎞️ Animation ===
function animate() {
  requestAnimationFrame(animate);

  // 🌍 Rotation
  const rotationSpeed = 0.002;
  earthMesh.rotation.y += rotationSpeed;
  lightsMesh.rotation.y += rotationSpeed;
  cloudsMesh.rotation.y += rotationSpeed * 1.15;
  glowMesh.rotation.y += rotationSpeed;

  controls.update();
  renderer.render(scene, camera);
}
animate();

// === 📱 Resize ===
window.addEventListener("resize", () => {
  const w = window.innerWidth;
  const h = window.innerHeight;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
});

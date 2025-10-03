<template>
  <div ref="container" class="scene-container"></div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
//todo
import * as THREE from "three"
//todo
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import GUI from 'lil-gui'

//todo
import Stats from "stats.js"



const container = ref<HTMLDivElement | null>(null)

let renderer: THREE.WebGLRenderer | null = null;
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let controls: OrbitControls
let gui: GUI
let frameId = 0

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xbfd1e5)

  const stats = new Stats()
  stats.showPanel(0)
  document.body.appendChild(stats.dom)

  const renderer = new THREE.WebGLRenderer({ antialias: true }) as any;
  renderer.physicallyCorrectLights = true
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5))
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  container.value!.appendChild(renderer.domElement)

  camera = new THREE.PerspectiveCamera(60, 1, 0.1, 100)
  camera.position.set(3, 2, 3)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true

  const ambient = new THREE.AmbientLight(0xffffff, 0.4)
  scene.add(ambient)

  const dirLight = new THREE.DirectionalLight(0xffffff, 1.0)
  dirLight.position.set(5, 10, 7)
  dirLight.castShadow = true
  dirLight.shadow.mapSize.set(2048, 2048)
  dirLight.shadow.camera.near = 0.5
  dirLight.shadow.camera.far = 50
  scene.add(dirLight)

  const planeGeo = new THREE.PlaneGeometry(20, 20)
  const planeMat = new THREE.MeshStandardMaterial({ color: 0x808080, roughness: 1.0, metalness: 0.0 })
  const ground = new THREE.Mesh(planeGeo, planeMat)
  ground.rotation.x = -Math.PI / 2
  ground.receiveShadow = true
  scene.add(ground)

  const sphereGeo = new THREE.SphereGeometry(0.4, 64, 64)
  const sphereMat = new THREE.MeshStandardMaterial({ metalness: 1.0, roughness: 0.05 })
  const sphere = new THREE.Mesh(sphereGeo, sphereMat)
  sphere.position.set(-1, 0.4, 0)
  sphere.castShadow = true
  scene.add(sphere)

  const torusGeo = new THREE.TorusKnotGeometry(0.3, 0.09, 160, 32)
  const torusMat = new THREE.MeshStandardMaterial({ metalness: 0.4, roughness: 0.35 })
  const torus = new THREE.Mesh(torusGeo, torusMat)
  torus.position.set(1, 0.6, 0)
  torus.castShadow = true
  scene.add(torus)

  const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
    format: THREE.RGBAFormat,
    generateMipmaps: true,
    minFilter: THREE.LinearMipmapLinearFilter
  })
  const cubeCamera = new THREE.CubeCamera(0.1, 100, cubeRenderTarget)
  scene.add(cubeCamera)

  let doorMesh: THREE.Mesh | null = null
  const textureLoader = new THREE.TextureLoader()
  //todo
  const woodPath = 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTAtr2vBZ0TlTZ0q0OBetTdyGeMRW7EoBCjzw&s'

  const disposeMesh = (m: THREE.Mesh) => {
    m.geometry.dispose()
    if (Array.isArray(m.material)) {
      (m.material as THREE.Material[]).forEach((mat) => mat.dispose())
    } else {
      (m.material as THREE.Material).dispose()
    }
    scene.remove(m)
  }

  const createDoor = (w = 0.9, h = 2.0, t = 0.06) => {
    if (doorMesh) {
      disposeMesh(doorMesh)
      doorMesh = null
    }
    const geo = new THREE.BoxGeometry(w, h, t)
    const mat = new THREE.MeshStandardMaterial({
      color: 0x8b5a2b,
      roughness: 0.7,
      metalness: 0.05
    })
    textureLoader.load(
      woodPath,
      (tex: THREE.Texture) => {
        tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
        tex.repeat.set(1, 1);
        mat.map = tex;
        mat.needsUpdate = true;
      },
      undefined,
      () => {}
    );

    const mesh = new THREE.Mesh(geo, mat)
    mesh.castShadow = true
    mesh.position.set(0, h / 2, -1.2)
    scene.add(mesh)
    doorMesh = mesh
  }

  const doorParams: { width: number; height: number; thickness: number } = { width: 0.9, height: 2.0, thickness: 0.06 }
  createDoor(doorParams.width, doorParams.height, doorParams.thickness)

  gui = new GUI()
  const doorFolder = gui.addFolder('Door (geometry)')
  doorFolder.add(doorParams, 'width', 0.3, 1.5, 0.01).onChange((v: number) => createDoor(v, doorParams.height, doorParams.thickness))
  doorFolder.add(doorParams, 'height', 0.8, 3.0, 0.01).onChange((v: number) => createDoor(doorParams.width, v, doorParams.thickness))
  doorFolder.add(doorParams, 'thickness', 0.02, 0.3, 0.01).onChange((v: number) => createDoor(doorParams.width, doorParams.height, v))
  doorFolder.open()

  const resize = () => {
    const width = container.value!.clientWidth
    const height = container.value!.clientHeight || window.innerHeight
    renderer.setSize(width, height)
    camera.aspect = width / height
    camera.updateProjectionMatrix()
  }
  window.addEventListener('resize', resize)
  resize()

  const animate = () => {
    stats.begin()

    controls.update()
    updateCameraByKeys()

    sphere.visible = false
    torus.visible = false
    cubeCamera.position.copy(new THREE.Vector3(-1, 0.4, 0))
    cubeCamera.update(renderer, scene)
    sphere.visible = true
    torus.visible = true

    sphereMat.envMap = cubeRenderTarget.texture
    sphereMat.needsUpdate = true
    torusMat.envMap = cubeRenderTarget.texture
    torusMat.needsUpdate = true

    renderer.render(scene, camera)
    stats.end()

    frameId = requestAnimationFrame(animate)
  }
  animate()
})

onUnmounted(() => {
  cancelAnimationFrame(frameId)
  window.removeEventListener('resize', () => {})
  try { controls.dispose() } catch {}
  try { gui.destroy() } catch {}
  if(renderer){
    try {
      (renderer as any)?.dispose();
    } catch {}
  }

})

const keys: Record<string, boolean> = {}

window.addEventListener("keydown", (e: KeyboardEvent) => (keys[e.code] = true))
window.addEventListener("keyup", (e: KeyboardEvent) => (keys[e.code] = false))

const speed = 0.05

function updateCameraByKeys() {
  const dir = new THREE.Vector3()
  camera.getWorldDirection(dir)

  if (keys["KeyW"]) camera.position.addScaledVector(dir, speed)
  if (keys["KeyS"]) camera.position.addScaledVector(dir, -speed)

  const right = new THREE.Vector3()
  right.crossVectors(camera.up, dir).normalize()
  if (keys["KeyA"]) camera.position.addScaledVector(right, speed)
  if (keys["KeyD"]) camera.position.addScaledVector(right, -speed)

  if (keys["Space"]) camera.position.y += speed
  if (keys["ShiftLeft"]) camera.position.y -= speed
}
</script>

<style scoped>
.scene-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
}
</style>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import Globe, { type GlobeInstance } from 'globe.gl'
import * as THREE from 'three'

export type PlanetKey = 'moon' | 'mars'
export type ViewMode = 'globe' | 'roi'
export type RoiLayer = 'optical' | 'slope' | 'thermal'

const currentPlanet = ref<PlanetKey>('moon')
const fullscreenPane = ref<'none' | 'left' | 'right'>('none')
const leftViewMode = ref<ViewMode>('globe')
const activeRoiLayer = ref<RoiLayer>('optical')

// Target Data
const targetInfo = ref({
  lat: -45.33,
  lng: 0.35,
  name: 'Beethoven Crater',
  meta: 'Impact basin rim • Basalt floor',
  slope: '5.2°',
  roughness: '1.15 m',
  elevationRange: '24.6 m'
})

// Info Popover visibility
const showLeftInfo = ref(true)
const showRightInfo = ref(false)
const earthMatched = ref(false)

// DOM refs
const leftGlobeContainer = ref<HTMLDivElement | null>(null)
const rightGlobeContainer = ref<HTMLDivElement | null>(null)

let leftGlobe: GlobeInstance | null = null
let rightGlobe: GlobeInstance | null = null
let leftResizeObserver: ResizeObserver | null = null
let rightResizeObserver: ResizeObserver | null = null

const PLANET_TEXTURES = {
  moon: {
    name: 'Moon',
    globe: '/textures/moon.jpg',
    bump: '/textures/moon-bump.jpg'
  },
  mars: {
    name: 'Mars',
    globe: '/textures/mars.jpg',
    bump: '/textures/mars-bump.jpg'
  },
  earth: {
    name: 'Earth',
    globe: '/textures/earth.jpg',
    bump: '/textures/earth-topology.png'
  }
}

// Terrestrial analog match result
const matchedAnalog = ref({
  name: 'Cinder Lake Crater Field',
  region: 'Arizona, United States',
  lat: 35.31,
  lng: -111.51,
  matchScore: '89.4%',
  slope: '5.8° (93% match)',
  roughness: '1.28 m (91% match)',
  geology: 'Basaltic Volcanic Cinders (96% match)',
  environment: 'Arid High Altitude (88% match)'
})

/**
 * Creates a vibrant RED 3D Map Pin:
 * - Red tapered needle pointing down to the surface
 * - Glowing Red sphere head on top
 * - Red circular base anchor
 */
function create3DPin(colorHex = 0xef4444) {
  const pinGroup = new THREE.Group()

  // 1. Tapered Needle / Cone (pointing down)
  const coneGeo = new THREE.ConeGeometry(1.2, 5.0, 32)
  coneGeo.rotateX(Math.PI)
  coneGeo.translate(0, 2.5, 0)
  const coneMat = new THREE.MeshStandardMaterial({
    color: colorHex,
    metalness: 0.3,
    roughness: 0.2
  })
  const cone = new THREE.Mesh(coneGeo, coneMat)
  pinGroup.add(cone)

  // 2. Head Sphere (bold vibrant red)
  const sphereGeo = new THREE.SphereGeometry(2.1, 32, 32)
  sphereGeo.translate(0, 5.6, 0)
  const sphereMat = new THREE.MeshStandardMaterial({
    color: colorHex,
    emissive: 0x991b1b,
    emissiveIntensity: 0.4,
    roughness: 0.2,
    metalness: 0.4
  })
  const sphere = new THREE.Mesh(sphereGeo, sphereMat)
  pinGroup.add(sphere)

  // 3. Small base disc
  const baseGeo = new THREE.CylinderGeometry(1.3, 1.3, 0.3, 32)
  baseGeo.translate(0, 0.15, 0)
  const baseMat = new THREE.MeshBasicMaterial({ color: colorHex })
  const base = new THREE.Mesh(baseGeo, baseMat)
  pinGroup.add(base)

  pinGroup.scale.set(0.9, 0.9, 0.9)
  return pinGroup
}

function initLeftGlobe() {
  if (!leftGlobeContainer.value) return

  const width = leftGlobeContainer.value.clientWidth || 600
  const height = leftGlobeContainer.value.clientHeight || 700
  const cfg = PLANET_TEXTURES[currentPlanet.value]

  leftGlobe = new Globe(leftGlobeContainer.value)
    .width(width)
    .height(height)
    .globeImageUrl(cfg.globe)
    .bumpImageUrl(cfg.bump)
    .backgroundColor('#ffffff')
    .showAtmosphere(false)
    .customLayerData([{ lat: targetInfo.value.lat, lng: targetInfo.value.lng }])
    .customThreeObject(() => create3DPin(0xef4444))
    .customThreeObjectUpdate((obj: any, d: any) => {
      if (!leftGlobe) return
      const coords = leftGlobe.getCoords(d.lat, d.lng, 0.005)
      if (coords) {
        Object.assign(obj.position, coords)
        const normal = new THREE.Vector3(coords.x, coords.y, coords.z).normalize()
        obj.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), normal)
      }
    })
    .onGlobeClick(({ lat, lng }: { lat: number; lng: number }) => {
      targetInfo.value.lat = Number(lat.toFixed(2))
      targetInfo.value.lng = Number(lng.toFixed(2))
      targetInfo.value.name = currentPlanet.value === 'moon' ? 'Target Site (Moon)' : 'Target Site (Mars)'
      targetInfo.value.meta = 'Selected target coordinate'

      // Automatically popup the info popover over the 'i' button
      showLeftInfo.value = true

      if (leftGlobe) {
        leftGlobe.customLayerData([{ lat: targetInfo.value.lat, lng: targetInfo.value.lng }])
      }
    })

  const controls = leftGlobe.controls()
  if (controls) {
    controls.autoRotate = false
    controls.enableDamping = true
    controls.dampingFactor = 0.05
  }

  // Set initial camera view
  leftGlobe.pointOfView({ lat: -25, lng: 5, altitude: 2.2 })

  leftResizeObserver = new ResizeObserver((entries) => {
    for (const entry of entries) {
      if (leftGlobe && entry.contentRect.width > 0 && entry.contentRect.height > 0) {
        leftGlobe.width(entry.contentRect.width)
        leftGlobe.height(entry.contentRect.height)
      }
    }
  })
  leftResizeObserver.observe(leftGlobeContainer.value)
}

function initRightGlobe() {
  if (!rightGlobeContainer.value) return

  const width = rightGlobeContainer.value.clientWidth || 600
  const height = rightGlobeContainer.value.clientHeight || 700
  const cfg = PLANET_TEXTURES.earth

  rightGlobe = new Globe(rightGlobeContainer.value)
    .width(width)
    .height(height)
    .globeImageUrl(cfg.globe)
    .bumpImageUrl(cfg.bump)
    .backgroundColor('#ffffff')
    .showAtmosphere(true)
    .atmosphereColor('#8ec5fc')
    .atmosphereAltitude(0.12)
    .pointsData([
      { lat: 35.31, lng: -111.51, name: 'Cinder Lake Crater Field (Analog 1)', color: '#ef4444', size: 0.9 },
      { lat: -24.86, lng: -69.25, name: 'Atacama Desert (Analog 2)', color: '#ef4444', size: 0.8 },
      { lat: 19.42, lng: -155.29, name: 'Kilauea Lava Tubes (Analog 3)', color: '#ef4444', size: 0.8 }
    ])
    .pointLat('lat')
    .pointLng('lng')
    .pointColor('color')
    .pointRadius('size')
    .pointAltitude(0.015)
    .pointLabel((d: any) => `<b>${d.name}</b><br/>Lat: ${d.lat}°, Lng: ${d.lng}°`)
    .onPointClick(() => {
      showRightInfo.value = true
      earthMatched.value = true
    })

  const controls = rightGlobe.controls()
  if (controls) {
    controls.autoRotate = false
    controls.enableDamping = true
    controls.dampingFactor = 0.05
  }

  // Focus Earth initially on Western Hemisphere
  rightGlobe.pointOfView({ lat: 20, lng: -100, altitude: 2.2 })

  rightResizeObserver = new ResizeObserver((entries) => {
    for (const entry of entries) {
      if (rightGlobe && entry.contentRect.width > 0 && entry.contentRect.height > 0) {
        rightGlobe.width(entry.contentRect.width)
        rightGlobe.height(entry.contentRect.height)
      }
    }
  })
  rightResizeObserver.observe(rightGlobeContainer.value)
}

function switchPlanet(planet: PlanetKey) {
  if (currentPlanet.value === planet || !leftGlobe) return
  currentPlanet.value = planet
  const cfg = PLANET_TEXTURES[planet]

  targetInfo.value.name = planet === 'moon' ? 'Beethoven Crater' : 'Jezero Crater'
  targetInfo.value.lat = planet === 'moon' ? -45.33 : 18.38
  targetInfo.value.lng = planet === 'moon' ? 0.35 : 77.58

  showLeftInfo.value = true

  leftGlobe
    .globeImageUrl(cfg.globe)
    .bumpImageUrl(cfg.bump)
    .customLayerData([{ lat: targetInfo.value.lat, lng: targetInfo.value.lng }])

  leftGlobe.pointOfView({ lat: targetInfo.value.lat, lng: targetInfo.value.lng, altitude: 2.2 }, 800)
}

function toggleFullscreen(pane: 'left' | 'right') {
  if (fullscreenPane.value === pane) {
    fullscreenPane.value = 'none'
  } else {
    fullscreenPane.value = pane
  }
}

function toggleLeftInfo() {
  showLeftInfo.value = !showLeftInfo.value
}

function toggleRightInfo() {
  showRightInfo.value = !showRightInfo.value
}

function enterRoiMode() {
  leftViewMode.value = 'roi'
  showLeftInfo.value = false
}

function exitRoiMode() {
  leftViewMode.value = 'globe'
  showLeftInfo.value = true
  if (leftGlobe && leftGlobeContainer.value) {
    leftGlobe.width(leftGlobeContainer.value.clientWidth)
    leftGlobe.height(leftGlobeContainer.value.clientHeight)
    leftGlobe.pointOfView({ lat: targetInfo.value.lat, lng: targetInfo.value.lng, altitude: 2.2 }, 300)
  }
}

function findTerrestrialAnalogs() {
  earthMatched.value = true
  showRightInfo.value = true

  if (rightGlobe) {
    // Fly Earth camera to Cinder Lake Crater Field, Arizona
    rightGlobe.pointOfView({ lat: matchedAnalog.value.lat, lng: matchedAnalog.value.lng, altitude: 1.6 }, 1200)
  }
}

onMounted(() => {
  initLeftGlobe()
  initRightGlobe()
})

onBeforeUnmount(() => {
  if (leftResizeObserver) leftResizeObserver.disconnect()
  if (rightResizeObserver) rightResizeObserver.disconnect()
})
</script>

<template>
  <div class="terratwinn-app">
    <!-- Top Bar: Options, Tools, TerraTwinn, Export -->
    <header class="top-bar">
      <div class="top-left">
        <button class="menu-item-btn">Options</button>
        <button class="menu-item-btn">Tools</button>
      </div>

      <div class="top-center">
        <h1 class="brand-title">TerraTwinn</h1>
      </div>

      <div class="top-right">
        <button class="export-capsule-btn">
          <span>Export</span>
          <svg class="chevron-down-icon" viewBox="0 0 16 16" width="14" height="14" fill="currentColor">
            <path d="M3.2 5.5a.75.75 0 0 1 1.06-.04L8 8.87l3.74-3.41a.75.75 0 1 1 1.02 1.1l-4.25 3.88a.75.75 0 0 1-1.02 0L3.24 6.56a.75.75 0 0 1-.04-1.06z"/>
          </svg>
        </button>
      </div>
    </header>

    <!-- Main Dual Viewport -->
    <main class="split-viewport">
      <!-- Left Viewport: Moon / Mars Target -->
      <section
        v-show="fullscreenPane === 'none' || fullscreenPane === 'left'"
        class="viewport-half left-half"
        :class="{ 'is-fullscreen': fullscreenPane === 'left' }"
      >
        <!-- 3D Globe Canvas Container: Permanently mounted so WebGL canvas is never destroyed -->
        <div ref="leftGlobeContainer" class="globe-canvas"></div>

        <!-- Mode 1: 3D Globe Controls Layer -->
        <div v-show="leftViewMode === 'globe'" class="globe-controls-layer">
          <!-- Left Header Controls -->
          <div class="viewport-header">
            <!-- Moon / Mars Segmented Toggle -->
            <div class="celestial-toggle">
              <button
                class="toggle-btn"
                :class="{ 'is-active': currentPlanet === 'moon' }"
                @click="switchPlanet('moon')"
              >
                Moon
              </button>
              <button
                class="toggle-btn"
                :class="{ 'is-active': currentPlanet === 'mars' }"
                @click="switchPlanet('mars')"
              >
                Mars
              </button>
            </div>

            <!-- Search Input -->
            <div class="search-capsule">
              <svg class="search-icon" viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.5">
                <circle cx="11" cy="11" r="8"></circle>
                <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
              </svg>
              <input type="text" placeholder="Search..." />
            </div>
          </div>

          <!-- Bottom Left: Chat Bubble pointing directly to the 'i' button -->
          <div v-if="showLeftInfo" class="info-callout-popover">
            <div class="popover-card">
              <div class="popover-line">Long <strong>{{ targetInfo.lng }}</strong></div>
              <div class="popover-line">Lat <strong>{{ targetInfo.lat }}</strong></div>
              <div class="popover-title"><strong>{{ targetInfo.name }}</strong></div>
              <div class="popover-meta">({{ targetInfo.meta }})</div>

              <div class="popover-divider"></div>

              <button class="inspect-roi-action-btn" @click="enterRoiMode">
                <span>Inspect 300m ROI</span>
                <span class="arrow-symbol">→</span>
              </button>
            </div>
            <!-- Pointer tail pointing down to the 'i' icon below -->
            <div class="popover-pointer-down"></div>
          </div>

          <!-- Left Bottom Icons: Bold Letter 'i' & Fullscreen -->
          <button
            class="bottom-icon-btn bottom-left-icon"
            :class="{ 'is-active-btn': showLeftInfo }"
            title="Toggle Target Information"
            @click="toggleLeftInfo"
          >
            <div class="info-circle-bold">i</div>
          </button>

          <button
            class="bottom-icon-btn bottom-right-icon"
            :title="fullscreenPane === 'left' ? 'Collapse' : 'Expand'"
            @click="toggleFullscreen('left')"
          >
            <svg viewBox="0 0 24 24" width="26" height="26" fill="none" stroke="#000000" stroke-width="2.5" stroke-linecap="round">
              <polyline points="15 3 21 3 21 9"></polyline>
              <polyline points="9 21 3 21 3 15"></polyline>
              <line x1="21" y1="3" x2="14" y2="10"></line>
              <line x1="3" y1="21" x2="10" y2="14"></line>
            </svg>
          </button>
        </div>

        <!-- Mode 2: High-Res 300m ROI Inspector Overlay -->
        <div v-if="leftViewMode === 'roi'" class="roi-inspector-container">
            <!-- ROI Top Toolbar -->
            <div class="roi-toolbar">
              <button class="back-globe-btn" @click="exitRoiMode">
                <span class="back-arrow">←</span>
                <span>Back to Globe</span>
              </button>

              <div class="roi-target-tag">
                <span class="target-badge">{{ targetInfo.name }}</span>
                <span class="coords-badge">300m × 300m ROI</span>
              </div>

              <!-- Layer Switcher -->
              <div class="roi-layer-pills">
                <button
                  class="layer-pill-btn"
                  :class="{ 'is-active': activeRoiLayer === 'optical' }"
                  @click="activeRoiLayer = 'optical'"
                >
                  Optical
                </button>
                <button
                  class="layer-pill-btn"
                  :class="{ 'is-active': activeRoiLayer === 'slope' }"
                  @click="activeRoiLayer = 'slope'"
                >
                  Slope / DEM
                </button>
                <button
                  class="layer-pill-btn"
                  :class="{ 'is-active': activeRoiLayer === 'thermal' }"
                  @click="activeRoiLayer = 'thermal'"
                >
                  Thermal
                </button>
              </div>
            </div>

            <!-- Surface High-Res Map View -->
            <div class="roi-map-viewport">
              <!-- Optical Surface Texture Layer -->
              <div v-if="activeRoiLayer === 'optical'" class="roi-surface-layer optical-layer">
                <div class="crater-shadow-basin"></div>
                <div class="micro-crater c1"></div>
                <div class="micro-crater c2"></div>
                <div class="micro-crater c3"></div>
                <div class="micro-crater c4"></div>
              </div>

              <!-- Slope / DEM Heatmap Layer -->
              <div v-else-if="activeRoiLayer === 'slope'" class="roi-surface-layer slope-layer">
                <div class="slope-gradient-overlay"></div>
                <div class="slope-legend">
                  <span class="legend-label">Slope Gradient</span>
                  <div class="legend-bar"></div>
                  <div class="legend-ticks">
                    <span>0° (Flat)</span>
                    <span>10°</span>
                    <span>25°+ (Steep)</span>
                  </div>
                </div>
              </div>

              <!-- Thermal Layer -->
              <div v-else class="roi-surface-layer thermal-layer">
                <div class="thermal-gradient-overlay"></div>
                <div class="thermal-legend">
                  <span class="legend-label">Radiometric Temp</span>
                  <div class="thermal-bar"></div>
                  <div class="legend-ticks">
                    <span>110 K</span>
                    <span>220 K</span>
                    <span>360 K</span>
                  </div>
                </div>
              </div>

              <!-- 300m x 300m Bounding Box Reticle -->
              <div class="roi-reticle-box">
                <div class="corner-bracket top-left"></div>
                <div class="corner-bracket top-right"></div>
                <div class="corner-bracket bottom-left"></div>
                <div class="corner-bracket bottom-right"></div>
                <div class="center-crosshair"></div>
                <div class="reticle-label">
                  <span class="reticle-dim">300 m × 300 m</span>
                  <span class="reticle-coords">{{ targetInfo.lat }}°, {{ targetInfo.lng }}°</span>
                </div>
              </div>
            </div>

            <!-- Elevation Transect Profile Section -->
            <div class="roi-transect-section">
              <div class="transect-header">
                <div class="transect-title">
                  <strong>Elevation Profile Transect</strong>
                  <span class="transect-sub">West to East Centerline (300 m)</span>
                </div>
                <div class="transect-metrics">
                  <span class="metric-item">Slope: <strong>{{ targetInfo.slope }}</strong></span>
                  <span class="metric-item">Roughness: <strong>{{ targetInfo.roughness }}</strong></span>
                  <span class="metric-item">Depth: <strong>-18.4 m</strong></span>
                </div>
              </div>

              <!-- SVG 1D Elevation Curve -->
              <div class="transect-chart-wrap">
                <svg viewBox="0 0 500 70" preserveAspectRatio="none" class="transect-svg">
                  <!-- Elevation fill area -->
                  <path
                    d="M 0 70 L 0 35 Q 80 32 140 28 Q 180 58 250 62 Q 320 58 360 30 Q 420 34 500 36 L 500 70 Z"
                    fill="#f1f5f9"
                  />
                  <!-- Center baseline -->
                  <line x1="0" y1="35" x2="500" y2="35" stroke="#cbd5e1" stroke-dasharray="3,3" stroke-width="1" />
                  <!-- Elevation line -->
                  <path
                    d="M 0 35 Q 80 32 140 28 Q 180 58 250 62 Q 320 58 360 30 Q 420 34 500 36"
                    fill="none"
                    stroke="#000000"
                    stroke-width="2.5"
                    stroke-linecap="round"
                  />
                  <!-- Rim markers -->
                  <circle cx="140" cy="28" r="3.5" fill="#ef4444" />
                  <circle cx="250" cy="62" r="3.5" fill="#000000" />
                  <circle cx="360" cy="30" r="3.5" fill="#ef4444" />
                </svg>
                <div class="transect-labels">
                  <span>0 m (West)</span>
                  <span>Center (Crater Floor -18.4m)</span>
                  <span>300 m (East)</span>
                </div>
              </div>

              <!-- Find Analogs CTA -->
              <div class="roi-cta-bar">
                <button class="find-analogs-btn" @click="findTerrestrialAnalogs">
                  <span>Find Terrestrial Analogs</span>
                  <span class="arrow-symbol">→</span>
                </button>
              </div>
            </div>
          </div>
        </section>

      <!-- Right Viewport: Earth Analogs -->
      <section
        v-show="fullscreenPane === 'none' || fullscreenPane === 'right'"
        class="viewport-half right-half"
        :class="{ 'is-fullscreen': fullscreenPane === 'right' }"
      >
        <!-- Right Header Controls -->
        <div class="viewport-header justify-end">
          <div class="search-capsule">
            <svg class="search-icon" viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.5">
              <circle cx="11" cy="11" r="8"></circle>
              <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
            </svg>
            <input type="text" placeholder="Search..." />
          </div>
        </div>

        <!-- 3D Globe Canvas Container -->
        <div ref="rightGlobeContainer" class="globe-canvas"></div>

        <!-- Standby or Matched Status Badge -->
        <div class="earth-status-indicator">
          <span v-if="!earthMatched" class="status-pill-standby">Awaiting Target ROI Confirmation</span>
          <span v-else class="status-pill-matched">Matched: {{ matchedAnalog.name }} ({{ matchedAnalog.matchScore }})</span>
        </div>

        <!-- Bottom Left: Chat Bubble pointing directly to Earth's 'i' button -->
        <div v-if="showRightInfo" class="info-callout-popover">
          <div class="popover-card">
            <div class="popover-line">Long <strong>{{ matchedAnalog.lng }}</strong></div>
            <div class="popover-line">Lat <strong>{{ matchedAnalog.lat }}</strong></div>
            <div class="popover-title"><strong>{{ matchedAnalog.name }}</strong></div>
            <div class="popover-meta">{{ matchedAnalog.region }}</div>

            <div class="popover-divider"></div>

            <div class="analog-match-badge-wrap">
              <span class="analog-score-pill">Match {{ matchedAnalog.matchScore }}</span>
            </div>

            <div class="analog-metrics-list">
              <div class="metric-row">
                <span class="metric-lbl">Slope:</span>
                <span class="metric-val">{{ matchedAnalog.slope }}</span>
              </div>
              <div class="metric-row">
                <span class="metric-lbl">Roughness:</span>
                <span class="metric-val">{{ matchedAnalog.roughness }}</span>
              </div>
              <div class="metric-row">
                <span class="metric-lbl">Geology:</span>
                <span class="metric-val">{{ matchedAnalog.geology }}</span>
              </div>
            </div>
          </div>
          <!-- Pointer tail pointing down to Earth's 'i' icon below -->
          <div class="popover-pointer-down"></div>
        </div>

        <!-- Right Bottom Icons: Bold Letter 'i' & Fullscreen -->
        <button
          class="bottom-icon-btn bottom-left-icon"
          :class="{ 'is-active-btn': showRightInfo }"
          title="Toggle Earth Analog Information"
          @click="toggleRightInfo"
        >
          <div class="info-circle-bold">i</div>
        </button>

        <button
          class="bottom-icon-btn bottom-right-icon"
          :title="fullscreenPane === 'right' ? 'Collapse' : 'Expand'"
          @click="toggleFullscreen('right')"
        >
          <svg viewBox="0 0 24 24" width="26" height="26" fill="none" stroke="#000000" stroke-width="2.5" stroke-linecap="round">
            <polyline points="15 3 21 3 21 9"></polyline>
            <polyline points="9 21 3 21 3 15"></polyline>
            <line x1="21" y1="3" x2="14" y2="10"></line>
            <line x1="3" y1="21" x2="10" y2="14"></line>
          </svg>
        </button>
      </section>
    </main>
  </div>
</template>

<style scoped>
.terratwinn-app {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background-color: #ffffff;
  color: #000000;
  font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", "SF Pro Display", -system-ui, "Segoe UI", Roboto, sans-serif;
  user-select: none;
}

/* ================== TOP BAR ================== */
.top-bar {
  height: 58px;
  min-height: 58px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 24px;
  border-bottom: 2px solid #000000;
  background: #ffffff;
  position: relative;
  z-index: 50;
}

.top-left {
  display: flex;
  align-items: center;
  gap: 32px;
}

.menu-item-btn {
  background: transparent;
  border: none;
  font-size: 19px;
  font-weight: 400;
  color: #000000;
  cursor: pointer;
  padding: 4px 0;
  transition: opacity 0.15s ease;
}

.menu-item-btn:hover {
  opacity: 0.7;
}

.top-center {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
}

.brand-title {
  margin: 0;
  font-size: 23px;
  font-weight: 700;
  letter-spacing: -0.3px;
  color: #000000;
}

.top-right {
  display: flex;
  align-items: center;
}

.export-capsule-btn {
  background: #000000;
  color: #ffffff;
  border: none;
  border-radius: 9999px;
  padding: 8px 18px;
  font-size: 15px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  transition: transform 0.15s ease, opacity 0.15s ease;
}

.export-capsule-btn:hover {
  opacity: 0.88;
  transform: scale(0.98);
}

.chevron-down-icon {
  display: block;
}

/* ================== MAIN SPLIT VIEWPORT ================== */
.split-viewport {
  flex: 1;
  display: flex;
  width: 100%;
  height: calc(100vh - 58px);
  position: relative;
  overflow: hidden;
  background: #ffffff;
}

.viewport-half {
  flex: 1;
  position: relative;
  height: 100%;
  overflow: hidden;
  background: #ffffff;
}

.left-half {
  border-right: 2px solid #000000;
}

.viewport-half.is-fullscreen {
  flex: 1 1 100%;
  width: 100%;
  border-right: none;
}

/* 3D Canvas */
.globe-canvas {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  z-index: 1;
}

/* ================== VIEWPORT CONTROLS ================== */
.viewport-header {
  position: absolute;
  top: 18px;
  left: 20px;
  right: 20px;
  z-index: 30;
  display: flex;
  align-items: center;
  justify-content: space-between;
  pointer-events: none;
}

.viewport-header.justify-end {
  justify-content: flex-end;
}

.viewport-header > * {
  pointer-events: auto;
}

/* Moon / Mars Capsule Segmented Toggle */
.celestial-toggle {
  background: #000000;
  border-radius: 9999px;
  padding: 3px 4px;
  display: inline-flex;
  align-items: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.toggle-btn {
  background: transparent;
  color: #ffffff;
  border: none;
  border-radius: 9999px;
  padding: 6px 16px;
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
}

.toggle-btn.is-active {
  background: #8b9bb4;
  color: #000000;
  font-weight: 600;
}

/* Search Capsule */
.search-capsule {
  background: #000000;
  border-radius: 9999px;
  padding: 8px 18px;
  display: inline-flex;
  align-items: center;
  gap: 10px;
  width: 190px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.search-icon {
  color: #ffffff;
  flex-shrink: 0;
}

.search-capsule input {
  background: transparent;
  border: none;
  outline: none;
  color: #ffffff;
  font-size: 15px;
  font-weight: 400;
  width: 100%;
  font-family: inherit;
}

.search-capsule input::placeholder {
  color: rgba(255, 255, 255, 0.75);
}

/* ================== CHAT BUBBLE POPOVER POINTING TO 'i' ================== */
.info-callout-popover {
  position: absolute;
  left: 20px;
  bottom: 64px;
  z-index: 40;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  animation: popoverFadeIn 0.2s ease-out;
}

@keyframes popoverFadeIn {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.popover-card {
  background: #000000;
  color: #ffffff;
  padding: 14px 18px;
  border-radius: 6px;
  font-size: 13.5px;
  line-height: 1.4;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  min-width: 210px;
  max-width: 260px;
}

.popover-line {
  margin-bottom: 2px;
}

.popover-title {
  margin-top: 5px;
  font-size: 15px;
}

.popover-meta {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.8);
  margin-top: 2px;
}

.popover-divider {
  height: 1px;
  background: rgba(255, 255, 255, 0.2);
  margin: 12px 0 10px 0;
}

.inspect-roi-action-btn {
  width: 100%;
  background: #ffffff;
  color: #000000;
  border: none;
  border-radius: 9999px;
  padding: 8px 14px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  transition: background 0.15s ease, transform 0.15s ease;
}

.inspect-roi-action-btn:hover {
  background: #e2e8f0;
  transform: translateY(-1px);
}

.inspect-roi-action-btn:active {
  transform: translateY(0);
}

.arrow-symbol {
  font-size: 14px;
}

/* Pointer tail pointing straight down to the 'i' button */
.popover-pointer-down {
  width: 0;
  height: 0;
  border-left: 9px solid transparent;
  border-right: 9px solid transparent;
  border-top: 11px solid #000000;
  margin-left: 10px;
}

/* Analog Match details inside Earth popover */
.analog-match-badge-wrap {
  margin-bottom: 8px;
}

.analog-score-pill {
  background: #22c55e;
  color: #000000;
  font-size: 11.5px;
  font-weight: 700;
  padding: 2px 8px;
  border-radius: 9999px;
  letter-spacing: 0.3px;
}

.analog-metrics-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 12px;
}

.metric-row {
  display: flex;
  justify-content: space-between;
  gap: 8px;
}

.metric-lbl {
  color: rgba(255, 255, 255, 0.7);
}

.metric-val {
  color: #ffffff;
  font-weight: 500;
}

/* ================== BOTTOM ICONS ================== */
.bottom-icon-btn {
  position: absolute;
  bottom: 20px;
  z-index: 20;
  background: transparent;
  border: none;
  cursor: pointer;
  padding: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.15s ease, opacity 0.15s ease;
}

.bottom-icon-btn:hover {
  opacity: 0.7;
  transform: scale(1.08);
}

.bottom-icon-btn.is-active-btn {
  transform: scale(1.08);
}

.bottom-icon-btn.is-active-btn .info-circle-bold {
  background: #000000;
  color: #ffffff;
}

.bottom-left-icon {
  left: 20px;
}

.bottom-right-icon {
  right: 20px;
}

/* Bold letter 'i' inside circle */
.info-circle-bold {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: 2.6px solid #000000;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: -apple-system, BlinkMacSystemFont, "Times New Roman", Georgia, serif;
  font-size: 17px;
  font-weight: 900;
  color: #000000;
  line-height: 1;
  user-select: none;
  box-sizing: border-box;
  padding-bottom: 2px;
  transition: all 0.15s ease;
}

/* ================== EARTH STANDBY STATUS PILL ================== */
.earth-status-indicator {
  position: absolute;
  top: 24px;
  left: 24px;
  z-index: 20;
}

.status-pill-standby {
  font-size: 12px;
  font-weight: 500;
  color: #64748b;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  padding: 6px 14px;
  border-radius: 9999px;
  letter-spacing: 0.2px;
}

.status-pill-matched {
  font-size: 12px;
  font-weight: 600;
  color: #000000;
  background: #dcfce7;
  border: 1px solid #86efac;
  padding: 6px 14px;
  border-radius: 9999px;
  letter-spacing: 0.2px;
}

.globe-controls-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 25;
}

.globe-controls-layer > * {
  pointer-events: auto;
}

/* ================== MODE 2: HIGH-RES 300M ROI INSPECTOR ================== */
.roi-inspector-container {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  background: #ffffff;
  z-index: 45;
  overflow: hidden;
  animation: fadeIn 0.25s ease;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.roi-toolbar {
  height: 52px;
  padding: 0 18px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid #e2e8f0;
  background: #ffffff;
  z-index: 10;
}

.back-globe-btn {
  background: transparent;
  border: none;
  font-size: 14px;
  font-weight: 500;
  color: #000000;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 10px;
  border-radius: 6px;
  transition: background 0.15s ease;
}

.back-globe-btn:hover {
  background: #f1f5f9;
}

.back-arrow {
  font-size: 16px;
}

.roi-target-tag {
  display: flex;
  align-items: center;
  gap: 8px;
}

.target-badge {
  font-size: 14px;
  font-weight: 700;
  color: #000000;
}

.coords-badge {
  font-size: 12px;
  color: #64748b;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  padding: 2px 8px;
  border-radius: 4px;
}

.roi-layer-pills {
  display: flex;
  background: #f1f5f9;
  border-radius: 9999px;
  padding: 2px;
  gap: 2px;
}

.layer-pill-btn {
  background: transparent;
  border: none;
  border-radius: 9999px;
  padding: 5px 12px;
  font-size: 12px;
  font-weight: 500;
  color: #64748b;
  cursor: pointer;
  transition: all 0.15s ease;
}

.layer-pill-btn.is-active {
  background: #000000;
  color: #ffffff;
  font-weight: 600;
}

/* Surface Map Viewport */
.roi-map-viewport {
  flex: 1;
  position: relative;
  background: #0d0f12;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.roi-surface-layer {
  position: absolute;
  inset: 0;
  background-size: cover;
  background-position: center;
}

.optical-layer {
  background: radial-gradient(circle at 45% 40%, #525a66 0%, #2a2e36 45%, #15181e 85%);
  box-shadow: inset 0 0 100px rgba(0, 0, 0, 0.8);
}

.crater-shadow-basin {
  position: absolute;
  width: 58%;
  height: 58%;
  top: 21%;
  left: 21%;
  border-radius: 50%;
  background: radial-gradient(circle at 40% 35%, #1a1d24 0%, #0d0f14 70%, #3e444f 100%);
  box-shadow: inset 8px 8px 25px rgba(0, 0, 0, 0.9), 2px 2px 12px rgba(255, 255, 255, 0.1);
}

.micro-crater {
  position: absolute;
  border-radius: 50%;
  background: radial-gradient(circle at 35% 35%, #101217 0%, #2f343e 100%);
  box-shadow: inset 2px 2px 6px rgba(0,0,0,0.8);
}

.micro-crater.c1 { width: 42px; height: 42px; top: 15%; left: 70%; }
.micro-crater.c2 { width: 28px; height: 28px; top: 75%; left: 25%; }
.micro-crater.c3 { width: 55px; height: 55px; top: 68%; left: 72%; }
.micro-crater.c4 { width: 20px; height: 20px; top: 32%; left: 12%; }

/* Slope Layer */
.slope-layer {
  background: #111827;
}

.slope-gradient-overlay {
  width: 100%;
  height: 100%;
  background: radial-gradient(circle at 50% 50%, #065f46 0%, #0d9488 25%, #f59e0b 50%, #dc2626 70%, #0d9488 90%);
  opacity: 0.85;
}

.slope-legend, .thermal-legend {
  position: absolute;
  bottom: 16px;
  right: 16px;
  background: rgba(0, 0, 0, 0.75);
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 6px;
  padding: 8px 12px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.legend-label {
  font-size: 11px;
  color: #e2e8f0;
  font-weight: 500;
}

.legend-bar {
  width: 130px;
  height: 8px;
  border-radius: 4px;
  background: linear-gradient(to right, #065f46, #0d9488, #f59e0b, #dc2626);
}

.legend-ticks {
  display: flex;
  justify-content: space-between;
  font-size: 9.5px;
  color: #94a3b8;
}

/* Thermal Layer */
.thermal-layer {
  background: #020617;
}

.thermal-gradient-overlay {
  width: 100%;
  height: 100%;
  background: radial-gradient(circle at 48% 45%, #312e81 0%, #4338ca 30%, #7c3aed 55%, #db2777 75%, #ea580c 95%);
  opacity: 0.88;
}

.thermal-bar {
  width: 130px;
  height: 8px;
  border-radius: 4px;
  background: linear-gradient(to right, #312e81, #7c3aed, #db2777, #ea580c);
}

/* Bounding Box 300m x 300m Reticle */
.roi-reticle-box {
  position: relative;
  width: 220px;
  height: 220px;
  border: 1.5px dashed rgba(255, 255, 255, 0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;
}

.corner-bracket {
  position: absolute;
  width: 14px;
  height: 14px;
  border-color: #ef4444;
  border-style: solid;
}

.corner-bracket.top-left { top: -2px; left: -2px; border-width: 2.5px 0 0 2.5px; }
.corner-bracket.top-right { top: -2px; right: -2px; border-width: 2.5px 2.5px 0 0; }
.corner-bracket.bottom-left { bottom: -2px; left: -2px; border-width: 0 0 2.5px 2.5px; }
.corner-bracket.bottom-right { bottom: -2px; right: -2px; border-width: 0 2.5px 2.5px 0; }

.center-crosshair {
  width: 12px;
  height: 12px;
  position: relative;
}

.center-crosshair::before, .center-crosshair::after {
  content: '';
  position: absolute;
  background: #ef4444;
}

.center-crosshair::before { width: 100%; height: 1.5px; top: 5px; left: 0; }
.center-crosshair::after { height: 100%; width: 1.5px; left: 5px; top: 0; }

.reticle-label {
  position: absolute;
  bottom: 8px;
  left: 8px;
  background: rgba(0, 0, 0, 0.7);
  padding: 3px 6px;
  border-radius: 4px;
  display: flex;
  flex-direction: column;
  gap: 1px;
}

.reticle-dim {
  font-size: 11px;
  font-weight: 700;
  color: #ffffff;
}

.reticle-coords {
  font-size: 9.5px;
  color: #94a3b8;
  font-family: ui-monospace, monospace;
}

/* Elevation Transect Profile */
.roi-transect-section {
  background: #ffffff;
  border-top: 1px solid #e2e8f0;
  padding: 12px 18px 14px 18px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.transect-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.transect-title strong {
  font-size: 13.5px;
  color: #000000;
  display: block;
}

.transect-sub {
  font-size: 11px;
  color: #64748b;
}

.transect-metrics {
  display: flex;
  gap: 12px;
  font-size: 12px;
}

.metric-item {
  color: #64748b;
}

.metric-item strong {
  color: #000000;
}

.transect-chart-wrap {
  position: relative;
  width: 100%;
}

.transect-svg {
  width: 100%;
  height: 64px;
  display: block;
}

.transect-labels {
  display: flex;
  justify-content: space-between;
  font-size: 10px;
  color: #94a3b8;
  margin-top: 2px;
}

.roi-cta-bar {
  display: flex;
  justify-content: flex-end;
  padding-top: 4px;
}

.find-analogs-btn {
  background: #000000;
  color: #ffffff;
  border: none;
  border-radius: 9999px;
  padding: 9px 20px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: transform 0.15s ease, opacity 0.15s ease;
}

.find-analogs-btn:hover {
  opacity: 0.85;
  transform: translateY(-1px);
}

.find-analogs-btn:active {
  transform: translateY(0);
}
</style>

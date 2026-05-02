# Earth — Lively Wallpaper

A photorealistic, real-time 3-D Earth rendered entirely in the browser using
[Three.js](https://threejs.org/) and custom GLSL shaders. Designed for use as
an animated desktop wallpaper in [Lively Wallpaper](https://rocksdanister.com/lively/).

---

## Table of Contents

1. [Features](#features)
2. [How It Works](#how-it-works)
   - [Scene Architecture](#scene-architecture)
   - [Star Field](#star-field)
   - [Earth Surface Shader](#earth-surface-shader)
   - [Ocean Rendering](#ocean-rendering)
   - [Cloud Modelling](#cloud-modelling)
   - [Atmospheric Limb Halo](#atmospheric-limb-halo)
   - [Tone Mapping & Gamma](#tone-mapping--gamma)
   - [Lively Property API](#lively-property-api)
3. [Setting Up in Lively Wallpaper](#setting-up-in-lively-wallpaper)
4. [Customisation Controls](#customisation-controls)
5. [Image Attribution](#image-attribution)
6. [License](#license)

---

## Features

- Real-time WebGL rendering at 60 fps (or locked to 30 fps to save GPU power)
- Physically-based day/night terminator with smooth twilight transition
- NASA Blue Marble satellite imagery with topography and ocean bathymetry shading
- NASA Black Marble city-lights texture on the dark side
- Procedural cloud system driven by multi-octave domain-warped FBM noise, with
  latitude bands that mirror real atmospheric circulation (ITCZ, Hadley cell
  subsidence, mid-latitude storm tracks)
- Realistic ocean: deep-navy colour, shallow coastal teal, Schlick Fresnel
  reflections, broad shimmer + tight specular sun glint, and optional
  bathymetric depth contrast
- Atmospheric limb halo with day, dusk, and night colours
- 15 000-star stellar background with colour-temperature-accurate stellar
  palette, power-law brightness distribution (Salpeter IMF), atmospheric
  scintillation, slow dome rotation, and per-star drift
- Fully adjustable via Lively Properties (no source editing required)

---

## How It Works

### Scene Architecture

The renderer is a standard **Three.js r128** `WebGLRenderer` with:

| Setting | Value | Purpose |
|---|---|---|
| `antialias` | true | MSAA edge smoothing |
| `outputEncoding` | `sRGBEncoding` | Correct display colour space |
| `toneMapping` | `ACESFilmicToneMapping` | Cinematic HDR compression |
| `toneMappingExposure` | ~1.7 | Overall brightness |

The scene contains four objects stacked in a `THREE.Group` (tilted 23.5° on
the X axis to match Earth's axial tilt):

| Object | Geometry | Purpose |
|---|---|---|
| `earthMesh` | `SphereGeometry(1, 256, 128)` | Surface + ocean + night lights |
| `cloudMesh` | `SphereGeometry(1.008, 128, 64)` | Procedural cloud layer |
| `atmosMesh` | `SphereGeometry(1.018, 128, 64)` | Thin limb halo |
| `stars` | `THREE.Points` (15 000 pts) | Background star field |

---

### Star Field

The star field is a `THREE.Points` object driven by a custom vertex/fragment
shader pair. Every aspect is physically motivated:

**Stellar colours** are sampled from a seven-bin spectral-class palette
(O/B blue-white through M orange-red) with frequency weights that approximate
the observed colour distribution of naked-eye stars.

**Brightness** follows a power-law draw (`Math.pow(Math.random(), 3.8)`),
which mimics the Salpeter Initial Mass Function: the vast majority of stars are
faint, while a small number are much brighter.

**Atmospheric scintillation** (twinkle) is produced by two incommensurate sine
waves per star, each driven at a unique frequency. Because the two frequencies
share no small rational ratio, the combined waveform is effectively
non-periodic over wallpaper-viewing timescales.

**Slow dome rotation** rotates the entire field about the Y axis at one full
revolution per ~2 hours (`0.000873 rad/s`), so the star field shifts
noticeably after ~10 minutes — a subtle detail that prevents the background
from looking frozen.

**Individual star drift** adds a slow oscillatory displacement to each star
using `sin`/`cos` with period 5–15 minutes, so the field composition changes
over extended viewing.

**Rendering** uses an Airy-disk approximation: a tight Gaussian core plus a
wide, dim diffuse halo, giving the characteristic "spiky" appearance of bright
stars when viewed through an atmosphere.

---

### Earth Surface Shader

The surface is rendered by a single GLSL fragment shader sampling four
textures simultaneously:

| Uniform | Texture | Source |
|---|---|---|
| `uDay` | `world.topo.bathy.200401.3x5400x2700.png` | NASA Blue Marble (January) |
| `uNight` | `BlackMarble_2016_3km.jpg` | NASA Black Marble 2016 |
| `uWater` | `earth-water.png` | Ocean mask (three-globe / NASA) |
| `uTopo` | `earth-topology.png` | Elevation map (three-globe / NASA) |

**Lighting** is a simple Lambertian diffuse term (`max(NdotL, 0)`) combined
with a bump-normal derived from screen-space derivatives (`dFdx`/`dFdy`) of the
topology texture. This gives free screen-space normal perturbation — mountains
and valleys catch or lose light without a separate normal-map texture.

**Ambient occlusion** is approximated by `mix(0.72, 1.0, topoValue)`: low
elevations (deep ocean, valley floors) receive a 28% AO darkening, while peaks
receive none.

**Sunny land glow** adds a warm golden tint to sun-facing, non-ocean pixels
whose luminance falls below the cloud detection threshold. This simulates
scattered photons re-emitted from vegetation and soil.

**Day/night blend** uses `smoothstep(-0.03, 0.07, NdotL)`: the transition
zone is 0.1 units wide in dot-product space, equivalent to roughly 6° of arc,
giving a realistic ~650 km-wide twilight band at the terminator.

**Night lights** are the city-light texture squared (`nightTex * nightTex`), which
compresses dim suburban glow and boosts bright urban cores — matching the
contrast profile of actual Black Marble imagery. Intensity is further
controlled by the `uNightIntensity` uniform.

**Saturation boost** (`mix(luma, colour, 1.45)`) lifts colour vibrancy to
compensate for the perceptual desaturation that occurs after tone mapping.

---

### Ocean Rendering

The ocean shader path is triggered wherever `uWater.r > 0` (the water-mask
texture is white over ocean and black over land).

**Deep ocean base colour** blends a rich navy (`vec3(0.005, 0.030, 0.130)`)
with a re-tinted version of the satellite texture
(`day * vec3(0.20, 0.55, 1.80)`) and a shallow coastal teal, mixed by a
`shallowness` factor derived from the luminance of the day texture — coastal
pixels are inherently brighter in the imagery, so this naturally picks out
continental shelves.

**Sky diffuse reflection** adds a dim blue scatter across the lit hemisphere
(`skyBlue * 0.10`), simulating the diffuse reflection of the sky dome off a
calm ocean surface.

**Fresnel reflection** uses the Schlick approximation:

$$F(\theta) = 0.02 + 0.98\,(1 - \cos\theta)^5$$

At grazing angles (ocean limb) the reflection coefficient approaches ~1,
brightening the ocean edges with reflected sky light — the same effect that
makes the ocean look silver near the horizon.

**Ocean blue boost** (`uOceanBlueBoost`) is a linear multiplier on the blue
channel, letting users push the ocean from grey (0) through neutral (50) to
vivid cobalt (100).

**Bathymetric depth** (`uOceanDepth`) mixes in a modulation factor derived from
the source texture luminance:

$$bathyMod = \text{mix}(1,\ \text{clamp}(\text{luma} \times 30 + 0.1,\ 0.08,\ 1.5),\ \text{uOceanDepth})$$

Because the NASA topo-bathy imagery encodes deeper water as darker pixels, this
naturally darkens ocean trenches and leaves continental shelves at full
brightness. The default value of 35 gives a subtle but visible bathymetry hint
without making the ocean look posterised.

**Sun glint** combines a broad shimmer lobe (`NdotH^40 × 0.55`) and a tight
specular point (`NdotH^380 × 5.0`), both masked to ocean pixels and to the
lit hemisphere. The result is a realistic elongated oval glitter track with a
bright central hotspot.

---

### Cloud Modelling

The cloud layer is a fully procedural GLSL shader; no cloud texture is loaded.
This means the clouds are never the same twice and evolve continuously.

#### Noise Foundation

A deterministic **value noise** function (`vnoise`) uses a lattice hash to
interpolate smoothly between random values at integer grid points. The
smoothstep kernel is the quintic `f = f³(f(6f − 15) + 10)`, which eliminates
the C² discontinuities that cause visible grid artefacts in simpler cubic
interpolation.

This noise is composed into **Fractal Brownian Motion (FBM)** at two
resolutions:

- `fbm4`: 4 octaves, each scaled by 2.13× and half the amplitude — used for
  large-scale cloud structures (cumulus fields, frontal bands)
- `fbm6`: 6 octaves — used for the final density evaluation, adding smaller
  cumulus turrets and wisps

#### Domain Warping

The shader uses the classic **Inigo Quilez domain-warp** technique:

```
q = fbm4(p)           // first warp
r = fbm4(p + 1.5·q)  // second warp
density = fbm6(p + r) // final sample at doubly-warped coordinate
```

Each stage offsets the sampling coordinate by a noise-derived vector, causing
the final cloud mass to fold back on itself in the complex, cauliflower-like
way real cloud systems do. Without domain warping, FBM clouds look too uniform;
with it they billow and curl organically.

#### Cloud Drift

The cloud sphere is rotated about the Y axis at `uCloudSpeed × 0.022 rad/s`
independently of Earth's surface rotation. Three additional slow offsets
(`uTime × [0.0040, 0.0025, 0.0018] × cloudSpeed`) translate the entire noise
field over time, so that clouds not only rotate but also evolve — new patterns
form as old ones drift out of frame.

#### Synoptic-Scale Evolution

A slow time-varying perturbation vector `tVar` (period ~2–3 hours) is added to
the domain-warp coordinates:

```glsl
float t1 = uTime * 0.00072, t2 = uTime * 0.00055;
vec3 tVar = vec3(sin(t1)*0.38, cos(t2*0.73)*0.26, sin(t2+1.57)*0.38);
```

This simulates the passage of large-scale pressure systems (highs and lows)
over the globe on a timescale consistent with real synoptic meteorology
(~hours to days when time-lapsed). The three incommensurate frequencies prevent
the evolution from appearing periodic.

#### Atmospheric Circulation Zones

The density field is modulated by latitude-dependent factors that mirror the
real tropospheric circulation:

| Zone | Latitude | Effect | Real-world mechanism |
|---|---|---|---|
| **ITCZ** | ±0–10° | +22% density | Trade winds converge at equator; deep convective towers |
| **Subtropical dry belt** | ~20–35° | −52% density | Hadley cell descending branch; subsiding dry air |
| **Mid-latitude storm tracks** | ~45–65° | +20% density | Ferrel cell cyclones; polar-front jet |

These three corrections are applied by multiplying the raw FBM density by:

```glsl
(1.0 - 0.52 * subtrop) + itcz + midLat
```

The result is a globe where the tropics show heavy convective towers, the
subtropics are nearly clear (reproducing the Sahara, Arabian, and Australian
deserts), and the mid-latitudes have the swirling cyclone bands recognisable
from any weather-satellite image.

#### Coverage and Opacity Controls

After density modulation, a `smoothstep(uCloudCovMin, uCloudCovMax, density)`
cuts the continuous noise into a binary-ish cloud/no-cloud mask. The gap
between the two thresholds (`uCloudCovMax − uCloudCovMin`) controls edge
sharpness: a narrow gap gives hard cumulonimbus edges; a wide gap gives soft
stratus layers. A `pow(alpha, 1.35)` contrast boost further hardens cloud
edges, suppressing the thin misty fringes that make procedural clouds look
"foggy".

#### Cloud Lighting

Each cloud fragment is lit by:

1. **Lambertian diffuse** (`max(NdotL, 0) × 0.88 + 0.12`) — the +0.12 ambient
   term keeps the night side faintly visible as a dark blue-grey
2. **Forward scatter** (`pow(max(−VdotL, 0), 7) × 0.50`) — the silver-lining
   effect when the sun is behind the cloud mass, scattered back toward the
   viewer
3. **Day/night colour mix** — lit clouds are bright off-white; unlit clouds are
   a dark cool grey, blended through `smoothstep(-0.12, 0.12, NdotL)`

---

### Atmospheric Limb Halo

A slightly larger sphere (radius 1.018, ~18 km above surface) renders the
atmospheric glow using additive blending. The opacity follows a tight
Fresnel-like `pow(1 − NdotV, 5.5)` — this falls off very sharply away from
the limb, keeping the atmosphere razor-thin and not affecting the face-on view
of the surface.

The colour transitions through three states:

- **Night limb**: deep indigo (`vec3(0.02, 0.04, 0.12)`)
- **Day limb**: sky blue (`vec3(0.16, 0.46, 1.00)`)
- **Dusk/dawn limb**: warm amber-orange (`vec3(0.70, 0.35, 0.10)`) — blended in
  when `NdotL` is close to zero (the day/night boundary), producing the golden
  arc visible at sunset from orbit

---

### Tone Mapping & Gamma

The pipeline uses a **modified Reinhard operator**:

$$C_{out} = \frac{C_{in} \cdot e}{C_{in} \cdot e + 0.85}$$

The constant `0.85` (rather than the standard `1.0`) allows the shoulder to
compress more gradually, preserving mid-tone contrast. A final `pow(colour,
1/2.2)` gamma encodes for sRGB display.

---

### Lively Property API

Lively Wallpaper communicates user-interface changes to the page via the global
`livelyPropertyListener(name, val)` callback. On page load, Lively calls this
function once for every property in `LivelyProperties.json`, initialising the
scene to the user's saved values. Thereafter it is called on every slider or
checkbox interaction. The wallpaper stores the last-applied values in a `config`
object and calls shader-specific helpers (`applyConfig`,
`updateCloudUniforms`, `updateSunDir`) to propagate changes in a single
`requestAnimationFrame` cycle.

---

## Setting Up in Lively Wallpaper

### Requirements

- [Lively Wallpaper](https://rocksdanister.com/lively/) v2.x (Windows 10/11)
- An internet connection for first load (textures stream from NASA servers;
  optional local texture files override network fetches)

### Installation

1. **Download** this repository as a `.zip` file (GitHub → Code → Download ZIP)
   or clone it:
   ```
   git clone https://github.com/vvanhee/earth-wallpaper.git
   ```

2. **Open Lively Wallpaper**.

3. Click **Add Wallpaper** (the `+` button in the top-right corner).

4. Select **From File** and browse to the folder you just downloaded.
   Select `earth-wallpaper.html`.

5. Lively will import the wallpaper, copy the folder to its library, and
   display the Earth on your desktop.

> **Tip:** If Lively shows a blank white or black screen, ensure your machine
> has WebView2 (Microsoft Edge) installed. It ships with Windows 11 by default.
> For Windows 10, download the WebView2 Runtime from
> [microsoft.com](https://go.microsoft.com/fwlink/p/?LinkId=2124703).

### Optional: Local High-Resolution Textures

The wallpaper tries to load local texture files before falling back to remote
URLs. Place these files in the same folder as `earth-wallpaper.html` to avoid
network requests and gain the highest available resolution:

| File | Resolution | Source |
|---|---|---|
| `world.topo.bathy.200401.3x5400x2700.png` | 5400×2700 | NASA Blue Marble — January (see attribution below) |
| `BlackMarble_2016_3km.jpg` | ~13500×6750 | NASA Black Marble 2016 (see attribution below) |

### Customising the Wallpaper

Right-click the wallpaper thumbnail in the Lively library and select
**Customise**. A panel on the right exposes all adjustable parameters. See
[Customisation Controls](#customisation-controls) for a full description.

---

## Customisation Controls

All controls are saved per-display and restored automatically on next launch.
Use **Restore Defaults** in the Lively customisation panel to reset everything.

### Earth

| Control | Range | Default | Description |
|---|---|---|---|
| Rotation Speed | 0–4 | 1.0 | Earth spin speed multiplier. 0 = paused, 1 = real-time scaled, 2 = double. |

### Sun

| Control | Range | Default | Description |
|---|---|---|---|
| Sun Azimuth | 0–360° | 5° | Horizontal sun angle; moves the day/night terminator around the globe. |
| Sun Elevation | −90–90° | 2° | Vertical sun angle. Negative values place the sun below the horizon (full night). |

### Clouds

| Control | Range | Default | Description |
|---|---|---|---|
| Cloud Drift Speed | 0–4 | 1.0 | Cloud layer rotation speed. 0 = frozen. |
| Cloud Coverage | 0–100 | 50 | How much sky is covered. Maps to the FBM density threshold. |
| Cloud Edge Sharpness | 0–100 | 50 | Narrower smoothstep gap = crisper cumulonimbus edges. |
| Cloud Opacity | 0–100 | 94 | Maximum cloud alpha. Reduce for more transparent, veil-like clouds. |

### Ocean

| Control | Range | Default | Description |
|---|---|---|---|
| Ocean Blue Intensity | 0–100 | 50 | Multiplies the ocean blue channel: 0 = grey, 50 = neutral, 100 = 2× vivid blue. |
| Ocean Depth Visibility | 0–100 | **35** | Bathymetric contrast. At 35, ocean trenches are subtly darker; at 100, full depth contrast. |

### Rendering

| Control | Range | Default | Description |
|---|---|---|---|
| Exposure / Brightness | 0–100 | 48 | Tone mapping exposure (~0.5×–3.0×). 48 → 1.70× default. |
| Night Lights Intensity | 0–100 | 40 | City-light brightness on the dark side. |
| Atmosphere Glow | 0–100 | 14 | Intensity of the blue limb halo. |
| Lock to 30 FPS | on/off | off | Caps animation at 30 fps to reduce GPU load. |

---

## Image Attribution

NASA imagery is in the public domain (produced by US government employees as
part of their official duties). Attribution is not legally required but is
provided here as a courtesy to the scientists who produced the data.

### Primary Day Texture (local optional file)

**Blue Marble: Next Generation — January, with Topography and Bathymetry**
`world.topo.bathy.200401.3x5400x2700.png`

> Reto Stöckli, NASA Earth Observatory (2004).
> *Blue Marble: Next Generation.* NASA Goddard Space Flight Center.
> Instrument: MODIS Terra (surface); SRTM/ETOPO2 (topo/bathy shading).
> https://visibleearth.nasa.gov/images/73909

### Primary Night Texture (local optional file)

**Black Marble 2016 — Global Night-Time Lights**
`BlackMarble_2016_3km.jpg`

> NASA Earth Observatory (2016).
> *Black Marble.* NASA Goddard Space Flight Center.
> Instrument: Suomi NPP / VIIRS Day/Night Band.
> https://visibleearth.nasa.gov/images/144898

### Fallback Day Texture (CDN)

**earth-day.jpg** (via `unpkg.com/three-globe`)

> Reto Stöckli, NASA Earth Observatory (2004).
> *Blue Marble: Next Generation.* Distributed via the
> [three-globe](https://github.com/vasturiano/three-globe) npm package (MIT).

### Fallback Night Texture (CDN / NASA direct)

`earth-night.jpg` (via three-globe) /
`earth_lights_lrg.jpg` (via NASA EO legacy URL)

> NASA Earth Observatory / Robert Simmon (2000).
> *Earth's City Lights.* NASA GSFC.
> https://visibleearth.nasa.gov/images/55167

### Water Mask and Topology Map

**earth-water.png** — Ocean/land mask  
**earth-topology.png** — Elevation/depth greyscale map

Both distributed via the [three-globe](https://github.com/vasturiano/three-globe)
npm package (MIT licence), derived from NASA Natural Earth and SRTM/ETOPO data.

### Rendering Library

**Three.js r128** — https://threejs.org  
MIT licence © 2010–2021 three.js authors.  
CDN: `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`

---

## License

MIT — see [LICENSE](LICENSE).

Image assets are NASA public-domain works and are not covered by the MIT
licence. See [Image Attribution](#image-attribution) above.

---

## Submitting to the Lively Community

There is no official Lively marketplace — the community distributes wallpapers
through GitHub and the Lively Discussions forum.

### Step 1 — Host the code on GitHub

Push this repository to a public GitHub repo.  
Update the `Contact` URL in `LivelyInfo.json` to your actual repository URL.

### Step 2 — Export a shareable `.zip`

In Lively, right-click the wallpaper thumbnail → **Export Wallpaper**. This
creates a self-contained `.zip` archive (Lively's `.lwp` format is just a
renamed zip). Upload it as a GitHub Release asset.

### Step 3 — Post in Lively Discussions

Open a new discussion in the
[Show and tell](https://github.com/rocksdanister/lively/discussions/categories/show-and-tell)
category:

- Include a screenshot or screen-recording GIF as a preview
- Link to your GitHub repo and/or the `.zip` release
- Briefly describe features and customisation options

That is the standard way community wallpapers are discovered and shared.
There is no pull-request or review pipeline for wallpaper content itself —
the Lively repository only accepts contributions to the application code.

---
name: glass-lens-effects
description: >
  Build world-class glass, lens, and light refraction effects for the web. Use this skill whenever
  the user mentions: magnifying glass, loupe, cursor lens, glass material, refraction, chromatic
  aberration, dispersion, iridescent, frosted glass, glassmorphism with depth, lens distortion,
  transmission material, MeshTransmissionMaterial, FBO transparency, glass shader, crystal effect,
  glass orb, jelly material, or anything that involves light bending through a surface.
  Also trigger for "make it look like glass", "add a glass effect", "realistic transparency",
  "cool glass thing I saw on Awwwards", or vague requests like "make it look expensive/premium/liquid".
  This skill covers three tiers: CSS-only, MeshTransmissionMaterial (drop-in), and custom GLSL shaders.
  Always use this skill — don't wing glass effects from memory, the gotchas will destroy you.
---

# Glass & Lens Effects — World-Class Reference

## PICK YOUR TIER FIRST

```
What are you building?
│
├─ Plain HTML/CSS/JS site? ──────────────────────────── TIER 0: CSS Frosted Glass
│                                                        or TIER 1: Canvas Loupe
│
├─ React Three Fiber, need it done in 10 minutes? ───── TIER 2: MeshTransmissionMaterial
│   (most people want this one)
│
└─ Custom shader, full control, the real thing? ──────── TIER 3: Custom GLSL
    (read references/shader-advanced.md)
```

---

## TIER 0 — CSS Frosted Glass (no JS required)

```css
.glass {
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(12px) saturate(180%);
  -webkit-backdrop-filter: blur(12px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.18);
  border-radius: 16px;
  box-shadow:
    0 8px 32px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.3);   /* top specular line */
}
```

**Dark glass variant:**
```css
.glass-dark {
  background: rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(20px) saturate(150%) brightness(0.8);
  border: 1px solid rgba(255, 255, 255, 0.08);
}
```

**Tinted glass (blue/purple):**
```css
.glass-tinted {
  background: rgba(100, 120, 255, 0.1);
  backdrop-filter: blur(16px) hue-rotate(15deg) saturate(200%);
  border: 1px solid rgba(150, 170, 255, 0.25);
}
```

> ⚠️ `backdrop-filter` requires a stacking context. If it's not working: add `isolation: isolate` to the parent. Also won't work if parent has `overflow: hidden` in some browsers.

---

## TIER 1 — Canvas Loupe (Magnifying Glass Cursor)

> As seen in Robert Pavlinić's "html-in-canvas" demo. Full implementation in `references/loupe-canvas.md`.

**30-second version (copy-paste, works):**

```html
<canvas id="loupe" style="position:fixed;top:0;left:0;width:100vw;height:100vh;pointer-events:none;z-index:9999;"></canvas>

<script>
const canvas = document.getElementById('loupe');
const ctx = canvas.getContext('2d');
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

const R = 100;       // lens radius
const ZOOM = 2.5;    // magnification
let snap = null;

// Capture page snapshot via html2canvas
// npm install html2canvas
import('https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.esm.js')
  .then(m => m.default(document.body)).then(c => { snap = c; });

document.addEventListener('mousemove', ({ clientX: x, clientY: y }) => {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  if (!snap) return;

  ctx.save();
  ctx.beginPath();
  ctx.arc(x, y, R, 0, Math.PI * 2);
  ctx.clip();

  ctx.drawImage(
    snap,
    x - R / ZOOM, y - R / ZOOM, (2 * R) / ZOOM, (2 * R) / ZOOM,
    x - R,        y - R,        2 * R,           2 * R
  );
  ctx.restore();

  // Chrome ring
  ctx.beginPath();
  ctx.arc(x, y, R, 0, Math.PI * 2);
  ctx.lineWidth = 8;
  ctx.strokeStyle = 'rgba(255,255,255,0.6)';
  ctx.stroke();
});
</script>
```

For the full version with tick marks, shadow, DPR handling, scroll re-capture, and React wrapper → see `references/loupe-canvas.md`.

---

## TIER 2 — MeshTransmissionMaterial (R3F Drop-In)

This is what 90% of people actually want. Handles refraction, chromatic aberration, roughness blur, backside rendering — all in one component.

**Install:**
```bash
npm install three @react-three/fiber @react-three/drei
```

**The simplest glass mesh that looks incredible:**
```jsx
import { Canvas } from '@react-three/fiber'
import { MeshTransmissionMaterial, Environment } from '@react-three/drei'

export default function Scene() {
  return (
    <Canvas camera={{ position: [0, 0, 5] }} dpr={[1, 2]}>
      <Environment preset="city" />          {/* REQUIRED: gives glass something to reflect */}
      <GlassOrb />
    </Canvas>
  )
}

function GlassOrb() {
  return (
    <mesh>
      <sphereGeometry args={[1.5, 64, 64]} />
      <MeshTransmissionMaterial
        transmission={1}           // 0-1, how see-through (1 = fully transmissive)
        thickness={2}              // depth of refraction — TUNE THIS FIRST
        roughness={0}              // 0 = crystal clear, 1 = frosted
        chromaticAberration={0.05} // rainbow edge fringing
        anisotropicBlur={0.1}      // soft directional blur
        distortion={0.2}           // wavy warping
        distortionScale={0.3}
        temporalDistortion={0.1}   // animation speed of distortion
        ior={1.5}                  // glass=1.5, water=1.33, diamond=2.42
        color="white"              // tints everything behind/inside
        samples={6}               // refraction quality (lower = faster)
      />
    </mesh>
  )
}
```

### The Most Important Props (in order of impact)

| Prop | Default | What it does | Tune it when... |
|------|---------|--------------|-----------------|
| `thickness` | 0 | Depth of refraction bending | You want more/less distortion |
| `roughness` | 0 | Blur amount (frosted effect) | You want frosted glass |
| `chromaticAberration` | 0.03 | Rainbow edge fringing | You want that prismatic look |
| `transmission` | 1 | Opacity of transmission | Partial glass |
| `ior` | 1.5 | Index of refraction | More/less bending |
| `distortion` | 0 | Noise-based surface warping | Wavy/jelly look |
| `samples` | 6 | Refraction sample count | Performance issues |
| `resolution` | fullscreen | FBO texture resolution | Performance issues |

### Glass Presets

```jsx
// CRYSTAL CLEAR
<MeshTransmissionMaterial transmission={1} thickness={3} roughness={0} chromaticAberration={0.03} ior={1.5} />

// FROSTED / MATTE
<MeshTransmissionMaterial transmission={1} thickness={1} roughness={0.8} chromaticAberration={0} resolution={64} />

// JELLY / SOAP BUBBLE
<MeshTransmissionMaterial transmission={1} thickness={0.5} roughness={0} chromaticAberration={0.15} distortion={0.5} distortionScale={0.5} temporalDistortion={0.3} />

// DIAMOND (high dispersion)
<MeshTransmissionMaterial transmission={1} thickness={5} ior={2.4} chromaticAberration={0.2} samples={10} />

// COLORED GLASS (wine glass)
<MeshTransmissionMaterial transmission={1} thickness={2} roughness={0} color="#c41e3a" ior={1.5} />

// BACKSIDE (renders interior faces — more realistic, more expensive)
<MeshTransmissionMaterial backside backsideThickness={1} thickness={2} />
```

### Performance Optimization

```jsx
// FASTEST: shared FBO buffer between multiple glass objects
const buffer = useFBO()
useFrame(({ gl, scene, camera }) => {
  gl.setRenderTarget(buffer)
  gl.render(scene, camera)
  gl.setRenderTarget(null)
})

// All glass objects share the same buffer pass
<mesh><MeshTransmissionMaterial buffer={buffer.texture} samples={4} resolution={512} /></mesh>
<mesh><MeshTransmissionMaterial buffer={buffer.texture} samples={4} resolution={512} /></mesh>

// FASTEST FROSTED: tiny resolution is unnoticeable with roughness
<MeshTransmissionMaterial roughness={0.8} resolution={32} />
```

### Known Bugs & Fixes

| Bug | Cause | Fix |
|-----|-------|-----|
| `WebGL: INVALID_OPERATION: Feedback loop` | R3F v9 + old drei | Upgrade to R3F v9 + React 19, or downgrade drei |
| Glass turns black | Another mesh overlapping | Add `depthWrite={false}` to the glass material |
| Glass invisible on mobile | DPR too high | Add `dpr={[1, 2]}` to Canvas |
| Other glass objects not visible through glass | Default behavior | Pass shared `buffer` texture manually (see above) |
| `roughness={1}` makes it invisible | Known behavior | Use max `roughness={0.95}` |
| Looks flat/boring | No environment | Add `<Environment preset="city" />` — this is mandatory |

---

## TIER 3 — Custom GLSL Shader

For when you need full control: custom dispersion, rygcbv color space, specular/diffuse lighting, the works.

→ **Read `references/shader-advanced.md`** for the complete implementation.

**Quick summary of the pipeline:**
1. Hide mesh → render scene to FBO → show mesh
2. Vertex shader: compute `worldNormal` and `eyeVector`
3. Fragment shader: `refract(eyeVector, worldNormal, 1.0/IOR)` per channel
4. Smooth with sampling loop (accumulate N samples with sliding offsets)
5. Saturate with luminance-based `sat()` function
6. Add Blinn-Phong specular + diffuse

→ See `references/shader-advanced.md` for copy-paste complete shader code.

---

## Vibe Coder Cheat Sheet

> "I want that cool glass ball thing I saw on the internet"
→ Tier 2, `sphereGeometry`, `MeshTransmissionMaterial`, add `<Environment preset="sunset" />`

> "I want a frosted card UI"
→ Tier 0, `backdrop-filter: blur(12px)`, done in 5 lines of CSS

> "I want a magnifying glass that follows my cursor"
→ Tier 1, see `references/loupe-canvas.md`

> "I want rainbow light splitting through glass like a prism"
→ Tier 2 with high `chromaticAberration`, or Tier 3 with rygcbv color space

> "I want glass that warps everything behind it like liquid"
→ Tier 2, set `distortion={0.8}`, `temporalDistortion={0.3}`, `roughness={0.1}`

> "It's not working and I don't know why"
→ Read `references/troubleshooting.md`

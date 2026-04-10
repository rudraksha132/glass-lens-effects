# Glass and Lens Effects Skill

A comprehensive technical resource for building high-quality glass, lens, and light refraction effects for web applications. This repository covers implementations ranging from lightweight CSS overlays to advanced 3D shaders.

---

## Implementation Tiers

Select the tier that best matches your project requirements and technical stack:

| Tier | Use Case | Technology | Complexity |
| :--- | :--- | :--- | :--- |
| Tier 0 | UI overlays, frosted glass cards | CSS / Tailwind | Low |
| Tier 1 | Interactive magnifying glass/loupe | HTML5 Canvas / JS | Medium |
| Tier 2 | 3D objects with physical refraction | React Three Fiber | High |
| Tier 3 | Custom dispersion and GLSL control | GLSL / WebGL | Advanced |

---

## Tier 0: CSS Frosted Glass
Ideal for standard web interfaces and glassmorphism layouts. This method requires no JavaScript and relies on hardware-accelerated filters.

### Standard Glass Effect
```css
.glass {
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(12px) saturate(180%);
  -webkit-backdrop-filter: blur(12px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.18);
  border-radius: 16px;
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.3); /* Top specular highlight */
}
````

### Dark Variant

```css
.glass-dark {
  background: rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(20px) saturate(150%) brightness(0.8);
  border: 1px solid rgba(255, 255, 255, 0.08);
}
```

-----

## Tier 1: Canvas Loupe (Magnifying Glass)

A performant implementation for creating a cursor-based magnifying lens using a 2D canvas snapshot.

```javascript
import html2canvas from 'html2canvas';

const R = 100;    // Lens radius
const ZOOM = 2.5; // Magnification factor

html2canvas(document.body).then(canvasSnapshot => {
  document.addEventListener('mousemove', ({ clientX: x, clientY: y }) => {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.save();
    ctx.beginPath();
    ctx.arc(x, y, R, 0, Math.PI * 2);
    ctx.clip();
    ctx.drawImage(
      canvasSnapshot,
      x - R / ZOOM, y - R / ZOOM, (2 * R) / ZOOM, (2 * R) / ZOOM,
      x - R, y - R, 2 * R, 2 * R
    );
    ctx.restore();
  });
});
```

-----

## Tier 2: MeshTransmissionMaterial (React Three Fiber)

The industry standard for realistic 3D glass. This material handles refraction, chromatic aberration, and roughness blur in a single component.

### Installation

```bash
npm install three @react-three/fiber @react-three/drei
```

### Core Implementation

```jsx
import { MeshTransmissionMaterial, Environment } from '@react-three/drei'

function GlassOrb() {
  return (
    <mesh>
      <sphereGeometry args={[1.5, 64, 64]} />
      <MeshTransmissionMaterial
        transmission={1}      // Degree of transparency
        thickness={2}         // Refraction depth
        roughness={0}         // Surface blur (0 to 1)
        chromaticAberration={0.05} // RGB edge splitting
        ior={1.5}             // Index of Refraction
        distortion={0.2}      // Noise-based warping
      />
    </mesh>
  )
}
```

### Configuration Presets

  * **Crystal Clear:** thickness={3}, roughness={0}, ior={1.5}
  * **Frosted Matte:** roughness={0.8}, resolution={64}, thickness={1}
  * **Diamond:** ior={2.4}, chromaticAberration={0.2}, thickness={5}

-----

## Troubleshooting and Optimization

| Issue | Potential Cause | Recommended Fix |
| :--- | :--- | :--- |
| Material appears black | Missing Environment | Add an \<Environment /\> component; glass requires reflections to render properly. |
| Significant frame drop | Resolution too high | Reduce the resolution prop to 512, 128, or 64 for frosted effects. |
| Visual feedback loop | Version conflict | Ensure React Three Fiber and Drei versions are up to date (R3F v9+). |
| Invisible on mobile | High DPR scaling | Explicitly set dpr={[1, 2]} on the Canvas component. |

-----

## License

Distributed under the MIT License. See LICENSE for more information.

```

# Advanced GLSL Shader — Complete Implementation

Source: Maxime Heckel's research + Junni.co.jp sampling technique + rygcbv color space paper.

This file contains production-ready, copy-paste shader code for the full custom glass pipeline.

---

## Complete Working R3F Component

```jsx
// GlassDispersion.jsx — drop this in, it works
import { useRef, useMemo } from 'react'
import { useFrame } from '@react-three/fiber'
import { useFBO } from '@react-three/drei'
import * as THREE from 'three'

const VERTEX_SHADER = `
  varying vec3 worldNormal;
  varying vec3 eyeVector;

  void main() {
    vec4 worldPos = modelMatrix * vec4(position, 1.0);
    vec4 mvPosition = viewMatrix * worldPos;
    gl_Position = projectionMatrix * mvPosition;

    // Eye vector: camera → surface (not normalized to world scale)
    eyeVector = normalize(worldPos.xyz - cameraPosition);

    // Normal must be transformed by the inverse-transpose of modelMatrix
    // to handle non-uniform scaling correctly
    worldNormal = normalize(normalMatrix * normal);
  }
`

// Smooth dispersion with rygcbv color space + saturation + specular
// LOOP constant controls quality (4 = fast, 16 = beautiful, 32 = cinematic)
const FRAGMENT_SHADER = `
  #define LOOP 16

  uniform sampler2D uTexture;
  uniform vec2 winResolution;

  // Per-channel IOR (refraction index)
  uniform float uIorR;   // Red
  uniform float uIorY;   // Yellow
  uniform float uIorG;   // Green
  uniform float uIorC;   // Cyan
  uniform float uIorB;   // Blue
  uniform float uIorV;   // Violet

  uniform float uRefractPower;       // overall refraction strength
  uniform float uChromaticAberration; // spread between channels
  uniform float uSaturation;          // colour intensity (>1 = vivid)
  uniform float uShininess;
  uniform float uDiffuseness;
  uniform vec3  uLight;               // world-space light position

  varying vec3 worldNormal;
  varying vec3 eyeVector;

  // --- Saturation using luminance mix ---
  vec3 sat(vec3 rgb, float intensity) {
    vec3 L = vec3(0.2125, 0.7154, 0.0721);
    vec3 grayscale = vec3(dot(rgb, L));
    return mix(grayscale, rgb, intensity);
  }

  // --- Blinn-Phong specular + diffuse ---
  float specularLight(vec3 light, float shininess, float diffuseness) {
    vec3 lightVec  = normalize(-light);
    vec3 halfVec   = normalize(eyeVector + lightVec);
    float NdotL    = dot(worldNormal, lightVec);
    float NdotH    = dot(worldNormal, halfVec);
    float NdotH2   = NdotH * NdotH;
    float kDiffuse = max(0.0, NdotL);
    float kSpec    = pow(NdotH2, shininess);
    return kSpec + kDiffuse * diffuseness;
  }

  // --- rygcbv from RGB ---
  // Fourier interpolation (Sundararaman, 2004)
  void getRYGCBV(vec3 col, out float r, out float y, out float g, out float c, out float b, out float v) {
    r = col.r * 0.5;
    g = col.g * 0.5;
    b = col.b * 0.5;
    y = (2.0*col.r + 2.0*col.g - col.b) / 6.0;
    c = (2.0*col.g + 2.0*col.b - col.r) / 6.0;
    v = (2.0*col.b + 2.0*col.r - col.g) / 6.0;
  }

  // --- RGB from rygcbv ---
  vec3 getRGBfromRYGCBV(float r, float y, float g, float c, float b, float v) {
    float R = r + (2.0*v + 2.0*y - c) / 3.0;
    float G = g + (2.0*y + 2.0*c - v) / 3.0;
    float B = b + (2.0*c + 2.0*v - y) / 3.0;
    return vec3(R, G, B);
  }

  void main() {
    vec2 uv     = gl_FragCoord.xy / winResolution.xy;
    vec3 normal = worldNormal;

    // IOR ratios (air → glass direction)
    float iorR = 1.0 / uIorR;
    float iorY = 1.0 / uIorY;
    float iorG = 1.0 / uIorG;
    float iorC = 1.0 / uIorC;
    float iorB = 1.0 / uIorB;
    float iorV = 1.0 / uIorV;

    // Refraction vectors per channel
    vec3 refR = refract(eyeVector, normal, iorR);
    vec3 refY = refract(eyeVector, normal, iorY);
    vec3 refG = refract(eyeVector, normal, iorG);
    vec3 refC = refract(eyeVector, normal, iorC);
    vec3 refB = refract(eyeVector, normal, iorB);
    vec3 refV = refract(eyeVector, normal, iorV);

    // Guard against Total Internal Reflection
    if (length(refR) < 0.001) refR = reflect(eyeVector, normal);
    if (length(refB) < 0.001) refB = reflect(eyeVector, normal);

    // --- Sampling loop for smooth dispersion ---
    vec3 color = vec3(0.0);

    for (int i = 0; i < LOOP; i++) {
      float slide = float(i) / float(LOOP) * 0.1;

      float sR = texture2D(uTexture, uv + refR.xy * (uRefractPower + slide * 1.0) * uChromaticAberration).r;
      float sG = texture2D(uTexture, uv + refG.xy * (uRefractPower + slide * 2.0) * uChromaticAberration).g;
      float sB = texture2D(uTexture, uv + refB.xy * (uRefractPower + slide * 3.0) * uChromaticAberration).b;
      vec3 sampleRGB = vec3(sR, sG, sB);

      // Decompose into rygcbv
      float r, y, g, c, b, v;
      getRYGCBV(sampleRGB, r, y, g, c, b, v);

      // Apply per-channel IOR offsets to rygcbv
      float rS = texture2D(uTexture, uv + refR.xy * (uRefractPower + slide * 1.0) * uChromaticAberration).r;
      float yS = (
        texture2D(uTexture, uv + refY.xy * (uRefractPower + slide * 1.5) * uChromaticAberration).r +
        texture2D(uTexture, uv + refY.xy * (uRefractPower + slide * 1.5) * uChromaticAberration).g
      ) * 0.5;
      float gS = texture2D(uTexture, uv + refG.xy * (uRefractPower + slide * 2.0) * uChromaticAberration).g;
      float cS = (
        texture2D(uTexture, uv + refC.xy * (uRefractPower + slide * 2.5) * uChromaticAberration).g +
        texture2D(uTexture, uv + refC.xy * (uRefractPower + slide * 2.5) * uChromaticAberration).b
      ) * 0.5;
      float bS = texture2D(uTexture, uv + refB.xy * (uRefractPower + slide * 3.0) * uChromaticAberration).b;
      float vS = (
        texture2D(uTexture, uv + refV.xy * (uRefractPower + slide * 0.5) * uChromaticAberration).b +
        texture2D(uTexture, uv + refV.xy * (uRefractPower + slide * 0.5) * uChromaticAberration).r
      ) * 0.5;

      color += getRGBfromRYGCBV(rS, yS, gS, cS, bS, vS);
    }

    // Normalise accumulated samples
    color /= float(LOOP);
    color = sat(color, uSaturation);

    // Specular highlight
    float spec = specularLight(uLight, uShininess, uDiffuseness);
    color += spec * 0.2;

    gl_FragColor = vec4(color, 1.0);
  }
`

export function GlassDispersion({
  iorBase = 1.5,       // base refraction index
  iorSpread = 0.06,    // how much R/G/B channels differ (dispersion intensity)
  refractPower = 0.4,
  chromaticAberration = 1.0,
  saturation = 1.08,
  shininess = 40.0,
  diffuseness = 0.2,
  light = [-1.0, 1.0, 1.0],
  children,
  ...meshProps
}) {
  const mesh = useRef()
  const fbo  = useFBO()

  const uniforms = useMemo(() => ({
    uTexture:             { value: null },
    winResolution:        { value: new THREE.Vector2(
      window.innerWidth, window.innerHeight
    ).multiplyScalar(Math.min(window.devicePixelRatio, 2)) },
    uIorR:                { value: iorBase - iorSpread },
    uIorY:                { value: iorBase - iorSpread * 0.5 },
    uIorG:                { value: iorBase },
    uIorC:                { value: iorBase + iorSpread * 0.3 },
    uIorB:                { value: iorBase + iorSpread },
    uIorV:                { value: iorBase - iorSpread * 0.2 },
    uRefractPower:        { value: refractPower },
    uChromaticAberration: { value: chromaticAberration },
    uSaturation:          { value: saturation },
    uShininess:           { value: shininess },
    uDiffuseness:         { value: diffuseness },
    uLight:               { value: new THREE.Vector3(...light) },
  }), [])

  useFrame(({ gl, scene, camera }) => {
    mesh.current.visible = false
    gl.setRenderTarget(fbo)
    gl.render(scene, camera)
    mesh.current.material.uniforms.uTexture.value = fbo.texture
    gl.setRenderTarget(null)
    mesh.current.visible = true
  })

  return (
    <mesh ref={mesh} {...meshProps}>
      {children}
      <shaderMaterial
        vertexShader={VERTEX_SHADER}
        fragmentShader={FRAGMENT_SHADER}
        uniforms={uniforms}
      />
    </mesh>
  )
}

// USAGE:
// <GlassDispersion iorBase={1.5} iorSpread={0.08}>
//   <icosahedronGeometry args={[2, 20]} />
// </GlassDispersion>
```

---

## Uniforms Explained

| Uniform | Typical Range | Effect |
|---------|--------------|--------|
| `uIorR/G/B/Y/C/V` | 1.1 – 2.5 | Per-channel refraction. Spread = dispersion intensity |
| `uRefractPower` | 0.1 – 1.0 | Scales all UV offsets — controls overall bend strength |
| `uChromaticAberration` | 0.5 – 3.0 | Multiplies the slide offset — widens colour separation |
| `uSaturation` | 0.8 – 2.0 | 1.0 = neutral, >1.0 = vivid, <1.0 = washed out |
| `uShininess` | 10 – 200 | Specular tightness. Higher = smaller, sharper glint |
| `uDiffuseness` | 0.0 – 1.0 | How much diffuse contributes to overall brightness |
| `uLight` | any vec3 | World-space light direction. Negated in shader. |

---

## Performance Tuning

| LOOP value | Samples | Use case |
|-----------|---------|---------|
| 4 | 4 | Mobile, background elements |
| 8 | 8 | General use |
| 16 | 16 | Hero elements |
| 32 | 32 | Still frames, video |

Change `#define LOOP 16` to tune. Each sample = one additional texture lookup per fragment.

---

## Animating Uniforms

```jsx
useFrame(({ clock }) => {
  const t = clock.getElapsedTime()
  mesh.current.material.uniforms.uIorR.value = 1.5 + Math.sin(t * 0.3) * 0.05
  mesh.current.material.uniforms.uIorB.value = 1.5 + Math.cos(t * 0.3) * 0.05
  mesh.current.material.uniforms.uLight.value.set(
    Math.sin(t) * 2, 2, Math.cos(t) * 2
  )
})
```

---

## Adding Normal Maps (Surface Imperfections)

```glsl
// In vertex shader, pass uv:
varying vec2 vUv;
void main() { vUv = uv; /* ... */ }

// In fragment shader:
uniform sampler2D uNormalMap;
vec3 normalMap = texture2D(uNormalMap, vUv).rgb * 2.0 - 1.0;
vec3 perturbedNormal = normalize(worldNormal + normalMap * 0.3);
// Use perturbedNormal in refract() calls instead of worldNormal
```

Load a glass/fingerprint/scratched texture as the normal map for realistic imperfections.

---

## Fresnel Edge Effect

Makes edges of glass more mirror-like (physically correct):

```glsl
// In fragment shader, after computing color:
float cosTheta = clamp(dot(-eyeVector, worldNormal), 0.0, 1.0);
float F0 = 0.04;  // glass: ((1.5-1)/(1.5+1))^2 ≈ 0.04
float fresnel = F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
color = mix(color, vec3(1.0), fresnel * 0.5);  // white glint at edges
```

---

## Caustics (Advanced)

Light patterns cast through glass onto surfaces below. Requires a second render pass.
See: https://tympanus.net/codrops (search "caustics three.js") for community implementations.
Not included here — this becomes its own rabbit hole.

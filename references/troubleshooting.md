# Troubleshooting Glass Effects — Every Bug, Every Fix

## MeshTransmissionMaterial

### "It's just black / invisible"
1. Did you add `<Environment>`? Glass with nothing to reflect = black. Add `<Environment preset="city" />`.
2. Is `transmission` set? Default is 1 but check yours explicitly.
3. Another mesh overlapping? Add `depthWrite={false}` to the glass material.
4. `roughness={1}`? Use max `roughness={0.95}` — full roughness makes it invisible.

### "WebGL: INVALID_OPERATION: Feedback loop"
- R3F v9 + React 18 bug. Fix: upgrade to React 19, or pass explicit `buffer` prop via useFBO.
- Or pin `@react-three/fiber` to `^8.x`.

### "Looks flat, nothing interesting"
- Missing Environment (see above).
- `thickness={0}` — this disables refraction. Set `thickness={1.5}` minimum.
- Add `backside` prop for interior face rendering.

### "Too expensive, GPU melting"
```jsx
// Fix 1: reduce resolution
<MeshTransmissionMaterial resolution={256} samples={4} />

// Fix 2: if roughness > 0, tiny resolution is invisible
<MeshTransmissionMaterial roughness={0.7} resolution={32} samples={4} />

// Fix 3: share buffer between multiple glass objects
const buffer = useFBO()
useFrame(({ gl, scene, camera }) => {
  gl.setRenderTarget(buffer)
  gl.render(scene, camera)
  gl.setRenderTarget(null)
})
<MeshTransmissionMaterial buffer={buffer.texture} />
```

### "Other transparent objects not visible through glass"
Pass a shared buffer — the default per-material FBO hides the scene from itself:
```jsx
// Renders everything including other transparent objects
<MeshTransmissionMaterial buffer={mySharedBuffer.texture} />
```

### "Glass looks good on desktop but wrong on mobile"
```jsx
// Always cap DPR
<Canvas dpr={[1, 2]}>
```

---

## Custom GLSL Shader

### "Black screen / nothing rendering"
1. Check browser console for GLSL compile errors (they'll appear as WebGL errors).
2. Common: forgot to multiply `winResolution` by `devicePixelRatio`.
3. FBO texture not assigned: ensure `mesh.current.visible = false` BEFORE `gl.setRenderTarget()`.
4. `gl_FragColor` is `vec4(0)` — verify texture2D is returning values by testing with `gl_FragColor = vec4(uv, 0.0, 1.0)`.

### "Texture offset looks correct but image is flipped"
GLSL `gl_FragCoord` origin is bottom-left, canvas is top-left. Flip Y:
```glsl
vec2 uv = vec2(gl_FragCoord.x, winResolution.y - gl_FragCoord.y) / winResolution.xy;
```

### "Refraction effect is invisible / too subtle"
- Increase `uRefractPower` (try 0.6–1.0).
- Increase IOR spread (e.g., R=1.4, B=1.7).
- Make sure background geometry is BEHIND the glass mesh.

### "Colors look washed out / grey"
- `uSaturation` is too low. Set to 1.2–1.5.
- LOOP is too low — more samples = better colour mixing.

### "Dispersion visible but not smooth (staircase artifacts)"
- Increase LOOP (16 → 32).
- Increase `uChromaticAberration` for wider spread.

### "refract() returns vec3(0) on edges"
Total Internal Reflection — add guard:
```glsl
vec3 refVec = refract(eyeVector, worldNormal, iorRatio);
if (length(refVec) < 0.001) refVec = reflect(eyeVector, worldNormal);
```

### "Shader works but mesh flickers"
Visibility toggle must be inside `useFrame`, not React render:
```jsx
// ✅ correct
useFrame(({ gl, scene, camera }) => {
  mesh.current.visible = false   // ← inside useFrame
  gl.setRenderTarget(fbo)
  gl.render(scene, camera)
  // ...
  mesh.current.visible = true
})

// ❌ wrong
useEffect(() => {
  mesh.current.visible = false   // ← outside — causes flicker
}, [])
```

### "Normal calculation looks wrong on scaled meshes"
The `normalMatrix` in Three.js is already the inverse-transpose of the modelview matrix — this is correct. Don't try to compute it manually. Just use `normalize(normalMatrix * normal)`.

### "Performance is terrible"
1. Reduce LOOP (16 → 8 → 4).
2. Add `#pragma optimize(on)` at top of shader.
3. Use `lowp` or `mediump` precision where full precision isn't needed.
4. Ensure `useMemo` wraps your uniforms object.

---

## CSS / Backdrop Filter

### "backdrop-filter not working"
1. Parent element needs a stacking context: add `isolation: isolate` or `position: relative; z-index: 1`.
2. Safari: add `-webkit-backdrop-filter`.
3. `overflow: hidden` on parent in some browsers kills it.
4. Firefox: partially supported, may need `layout.css.backdrop-filter.enabled` in about:config.

### "backdrop-filter works but looks pixelated on mobile"
```css
.glass {
  -webkit-backdrop-filter: blur(12px) saturate(180%);
  backdrop-filter: blur(12px) saturate(180%);
  /* Smaller blur on mobile: */
  @media (max-width: 768px) {
    backdrop-filter: blur(8px);
  }
}
```

### "Glass card flickers/repaints on scroll"
```css
.glass {
  will-change: transform;  /* promotes to GPU compositing layer */
  transform: translateZ(0);
}
```

---

## General

### "It looks great on my machine but janky on client's laptop"
- Lower LOOP / samples / resolution.
- Add `performanceMode` check: `navigator.deviceMemory < 4` → use lower quality.
- Check GPU: `gl.getParameter(gl.RENDERER)` — integrated GPUs are ~5x slower.

### "I want to animate the glass effect"
```jsx
// Animate uniforms in useFrame (GLSL shader)
useFrame(({ clock }) => {
  const t = clock.getElapsedTime()
  mat.current.uniforms.uIorR.value = 1.45 + Math.sin(t * 0.4) * 0.06
  mat.current.uniforms.uIorB.value = 1.55 + Math.cos(t * 0.4) * 0.06
})

// Animate MTM props via useFrame
useFrame(({ clock }) => {
  const t = clock.getElapsedTime()
  mat.current.chromaticAberration = 0.03 + Math.abs(Math.sin(t)) * 0.1
})
```

### "I want glass to react to mouse position"
```jsx
// Pass mouse position as uniform
const uniforms = useMemo(() => ({
  uMouse: { value: new THREE.Vector2(0, 0) }
}), [])

useFrame(({ mouse }) => {
  uniforms.uMouse.value.set(mouse.x * 0.5 + 0.5, mouse.y * 0.5 + 0.5)
})

// In GLSL, use uMouse to offset refraction:
// vec3 refVec = refract(eyeVector, normal + vec3(uMouse * 0.1, 0.0), iorRatio);
```

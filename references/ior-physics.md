# IOR & Physics Cheat Sheet

## Index of Refraction — Real Materials

| Material | IOR | Visual character |
|----------|-----|-----------------|
| Vacuum / Air | 1.00 | no refraction |
| Ice | 1.31 | subtle, cold |
| Water | 1.33 | gentle ripple |
| Standard glass | 1.52 | classic window glass |
| Crown glass (optical) | 1.52–1.53 | low dispersion, sharp |
| Flint glass | 1.60–1.70 | higher dispersion, colorful |
| Sapphire | 1.77 | strong, gemlike |
| Cubic Zirconia | 2.15 | very strong |
| Diamond | 2.42 | extreme, rainbow-heavy |
| Silicon | 3.96 | effectively opaque in visible light |

## Abbe Number (Dispersion Intensity)

Higher Abbe V = lower dispersion (less rainbow). Lower = more prismatic.

| Glass type | Abbe V | IOR spread for R→B |
|-----------|--------|--------------------|
| Crown glass | ~60 | ±0.008 |
| Dense flint | ~36 | ±0.018 |
| Very dense flint | ~25 | ±0.03 |
| "Movie glass" | ~15 | ±0.06 (exaggerated) |

For web shaders, use exaggerated values (±0.03–0.10) for visible effect.

## Snell's Law

```
n₁ · sin(θ₁) = n₂ · sin(θ₂)
```

In GLSL: `refract(incident, normal, n1/n2)`
- incident: normalized vector from camera to surface
- normal: surface normal (pointing away from surface)
- n1/n2: ratio of entry medium IOR to exit medium IOR

For air→glass: `refract(eyeVec, normal, 1.0/1.52)`

## Fresnel Equations (Schlick Approximation)

```
R(θ) = R₀ + (1 - R₀)(1 - cos θ)⁵
where R₀ = ((n₁ - n₂) / (n₁ + n₂))²
```

For glass (n=1.5): R₀ = ((1-1.5)/(1+1.5))² = 0.04

In GLSL:
```glsl
float fresnelSchlick(float cosTheta, float F0) {
  return F0 + (1.0 - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
}
float cosTheta = dot(-eyeVector, worldNormal);
float fresnel  = fresnelSchlick(cosTheta, 0.04);
```

## Total Internal Reflection

Occurs when light exits a denser medium at too shallow an angle.
Critical angle = arcsin(n₂/n₁)

For glass→air: arcsin(1/1.5) ≈ 41.8°

In practice: `refract()` returns `vec3(0)` when TIR occurs. Guard with:
```glsl
vec3 r = refract(I, N, eta);
if (length(r) < 0.001) r = reflect(I, N);
```

## rygcbv Color Space

Extends RGB to 6 channels for richer dispersion simulation.

**RGB → rygcbv:**
```
r = R/2
g = G/2
b = B/2
y = (2R + 2G - B) / 6
c = (2G + 2B - R) / 6
v = (2B + 2R - G) / 6
```

**rygcbv → RGB (after IOR manipulation):**
```
R = r + (2v + 2y - c) / 3
G = g + (2y + 2c - v) / 3
B = b + (2c + 2v - y) / 3
```

Source: Sundararaman, R. "Color Gamut Expansion Using Fourier Interpolation" (2004).
Taylor Petrick's WebGL lens implementation brought this into shader art.

## Suggested Starting Presets

### Regular glass (like a window):
```
IorR=1.14, IorY=1.16, IorG=1.18, IorC=1.20, IorB=1.22, IorV=1.16
```

### Crystal / premium (visible dispersion):
```
IorR=1.12, IorY=1.15, IorG=1.18, IorC=1.22, IorB=1.28, IorV=1.13
```

### Diamond (wild, exaggerated):
```
IorR=1.05, IorY=1.12, IorG=1.22, IorC=1.34, IorB=1.48, IorV=1.08
```

### Water lens (subtle, organic):
```
IorR=1.28, IorY=1.30, IorG=1.31, IorC=1.33, IorB=1.35, IorV=1.29
```

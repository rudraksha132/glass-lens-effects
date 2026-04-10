# Canvas Loupe — Complete Implementation

## Vanilla JS (Production-Ready)

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { cursor: none; margin: 0; }
    #loupe-canvas {
      position: fixed; top: 0; left: 0;
      width: 100vw; height: 100vh;
      pointer-events: none;
      z-index: 9999;
    }
  </style>
</head>
<body>
  <!-- your page content here -->

  <canvas id="loupe-canvas"></canvas>

  <script type="module">
    import html2canvas from 'https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.esm.js'

    const canvas = document.getElementById('loupe-canvas')
    const ctx    = canvas.getContext('2d')
    const DPR    = Math.min(window.devicePixelRatio, 2)

    const CONFIG = {
      radius:  110,      // px — lens radius
      zoom:    2.8,      // magnification factor
      ringW:   10,       // ring width
      ticks:   72,       // tick marks around ring
      shadow:  true,     // drop shadow under lens
    }

    // DPR-correct canvas sizing
    function resize() {
      canvas.width  = window.innerWidth  * DPR
      canvas.height = window.innerHeight * DPR
      canvas.style.width  = window.innerWidth  + 'px'
      canvas.style.height = window.innerHeight + 'px'
      ctx.scale(DPR, DPR)
    }
    resize()
    window.addEventListener('resize', resize)

    // --- Snapshot management ---
    let snap = null
    let snapTimeout = null

    async function takeSnap() {
      snap = await html2canvas(document.body, {
        scale: 1,
        useCORS: true,
        allowTaint: true,
        logging: false,
        ignoreElements: el => el.id === 'loupe-canvas',
      })
    }

    // Initial capture + re-capture on scroll (throttled)
    takeSnap()
    window.addEventListener('scroll', () => {
      clearTimeout(snapTimeout)
      snapTimeout = setTimeout(takeSnap, 150)
    })
    window.addEventListener('resize', () => {
      clearTimeout(snapTimeout)
      snapTimeout = setTimeout(takeSnap, 300)
    })

    // --- Drawing ---
    let mouseX = -999, mouseY = -999

    function drawLens(x, y) {
      const { radius: R, zoom: Z, ringW, ticks } = CONFIG
      ctx.clearRect(0, 0, canvas.width / DPR, canvas.height / DPR)

      if (!snap) return

      // Drop shadow
      if (CONFIG.shadow) {
        ctx.save()
        ctx.shadowColor = 'rgba(0,0,0,0.35)'
        ctx.shadowBlur  = 24
        ctx.shadowOffsetY = 8
        ctx.beginPath()
        ctx.arc(x, y, R + ringW / 2, 0, Math.PI * 2)
        ctx.fillStyle = 'transparent'
        ctx.fill()
        ctx.restore()
      }

      // Clipped magnified content
      ctx.save()
      ctx.beginPath()
      ctx.arc(x, y, R, 0, Math.PI * 2)
      ctx.clip()

      // Source rect on the snapshot (accounting for scroll)
      const scrollX = window.scrollX
      const scrollY = window.scrollY
      const srcW = (2 * R) / Z
      const srcH = (2 * R) / Z
      const srcX = (x + scrollX) - srcW / 2
      const srcY = (y + scrollY) - srcH / 2

      ctx.drawImage(snap, srcX, srcY, srcW, srcH, x - R, y - R, 2 * R, 2 * R)

      // Inner highlight (specular glint at top-left)
      const gInner = ctx.createRadialGradient(x - R * 0.3, y - R * 0.3, 0, x, y, R)
      gInner.addColorStop(0, 'rgba(255,255,255,0.12)')
      gInner.addColorStop(1, 'rgba(0,0,0,0)')
      ctx.fillStyle = gInner
      ctx.fillRect(x - R, y - R, 2 * R, 2 * R)

      ctx.restore()

      // --- Chrome ring ---
      const gRing = ctx.createRadialGradient(x, y, R - ringW, x, y, R + ringW * 0.5)
      gRing.addColorStop(0,   'rgba(255,255,255,0.95)')
      gRing.addColorStop(0.3, 'rgba(210,210,215,0.85)')
      gRing.addColorStop(0.7, 'rgba(150,150,160,0.7)')
      gRing.addColorStop(1,   'rgba(80,80,90,0.4)')

      ctx.beginPath()
      ctx.arc(x, y, R, 0, Math.PI * 2)
      ctx.lineWidth   = ringW
      ctx.strokeStyle = gRing
      ctx.stroke()

      // --- Tick marks (measuring loupe aesthetic) ---
      for (let i = 0; i < ticks; i++) {
        const angle    = (i / ticks) * Math.PI * 2 - Math.PI / 2
        const isMajor  = i % 9 === 0
        const isMed    = i % 3 === 0
        const inner    = R + ringW * 0.6
        const outer    = R + ringW * (isMajor ? 1.3 : isMed ? 1.1 : 0.85)
        const cos = Math.cos(angle), sin = Math.sin(angle)

        ctx.beginPath()
        ctx.moveTo(x + cos * inner, y + sin * inner)
        ctx.lineTo(x + cos * outer, y + sin * outer)
        ctx.lineWidth   = isMajor ? 1.5 : 0.75
        ctx.strokeStyle = `rgba(40,40,50,${isMajor ? 0.8 : 0.5})`
        ctx.stroke()
      }

      // Outer ring edge
      ctx.beginPath()
      ctx.arc(x, y, R + ringW, 0, Math.PI * 2)
      ctx.lineWidth   = 1
      ctx.strokeStyle = 'rgba(80,80,90,0.3)'
      ctx.stroke()
    }

    // Smooth cursor with lerp
    let lerpX = 0, lerpY = 0
    let rafId = null

    document.addEventListener('mousemove', ({ clientX, clientY }) => {
      mouseX = clientX
      mouseY = clientY
    })

    function animate() {
      lerpX += (mouseX - lerpX) * 0.15   // lower = smoother/laggier
      lerpY += (mouseY - lerpY) * 0.15
      drawLens(lerpX, lerpY)
      rafId = requestAnimationFrame(animate)
    }
    animate()

    // Hide lens when mouse leaves
    document.addEventListener('mouseleave', () => {
      mouseX = -999; mouseY = -999
    })
  </script>
</body>
</html>
```

---

## React Component Version

```jsx
// Loupe.jsx
import { useEffect, useRef, useCallback } from 'react'

const CONFIG = {
  radius: 110,
  zoom:   2.8,
  ringW:  10,
  ticks:  72,
}

export function Loupe() {
  const canvasRef = useRef(null)
  const snapRef   = useRef(null)
  const mouseRef  = useRef({ x: -999, y: -999 })
  const lerpRef   = useRef({ x: 0, y: 0 })
  const rafRef    = useRef(null)

  // Dynamically import html2canvas only on client
  const captureSnap = useCallback(async () => {
    const { default: html2canvas } = await import('html2canvas')
    snapRef.current = await html2canvas(document.body, {
      scale: 1,
      useCORS: true,
      logging: false,
      ignoreElements: el => el === canvasRef.current,
    })
  }, [])

  useEffect(() => {
    const canvas = canvasRef.current
    const ctx    = canvas.getContext('2d')
    const DPR    = Math.min(window.devicePixelRatio, 2)

    const resize = () => {
      canvas.width  = window.innerWidth  * DPR
      canvas.height = window.innerHeight * DPR
      canvas.style.width  = window.innerWidth  + 'px'
      canvas.style.height = window.innerHeight + 'px'
      ctx.scale(DPR, DPR)
    }
    resize()
    window.addEventListener('resize', resize)

    captureSnap()

    let snapTimeout
    const onScroll = () => {
      clearTimeout(snapTimeout)
      snapTimeout = setTimeout(captureSnap, 150)
    }
    window.addEventListener('scroll', onScroll)

    const onMove = ({ clientX, clientY }) => {
      mouseRef.current = { x: clientX, y: clientY }
    }
    document.addEventListener('mousemove', onMove)
    document.addEventListener('mouseleave', () => { mouseRef.current = { x: -999, y: -999 } })

    const { radius: R, zoom: Z, ringW } = CONFIG

    function draw() {
      const { x: mx, y: my } = mouseRef.current
      const lerp = lerpRef.current
      lerp.x += (mx - lerp.x) * 0.15
      lerp.y += (my - lerp.y) * 0.15
      const x = lerp.x, y = lerp.y

      ctx.clearRect(0, 0, canvas.width, canvas.height)
      const snap = snapRef.current
      if (!snap || x < 0) { rafRef.current = requestAnimationFrame(draw); return }

      ctx.save()
      ctx.beginPath()
      ctx.arc(x, y, R, 0, Math.PI * 2)
      ctx.clip()
      const srcW = (2 * R) / Z
      ctx.drawImage(snap, x + window.scrollX - srcW / 2, y + window.scrollY - srcW / 2, srcW, srcW, x - R, y - R, 2 * R, 2 * R)
      ctx.restore()

      // Ring
      ctx.beginPath()
      ctx.arc(x, y, R, 0, Math.PI * 2)
      ctx.lineWidth = ringW
      ctx.strokeStyle = 'rgba(220,220,225,0.85)'
      ctx.stroke()

      rafRef.current = requestAnimationFrame(draw)
    }
    rafRef.current = requestAnimationFrame(draw)

    return () => {
      cancelAnimationFrame(rafRef.current)
      window.removeEventListener('resize', resize)
      window.removeEventListener('scroll', onScroll)
      document.removeEventListener('mousemove', onMove)
    }
  }, [captureSnap])

  return (
    <canvas
      ref={canvasRef}
      style={{ position: 'fixed', top: 0, left: 0, pointerEvents: 'none', zIndex: 9999 }}
    />
  )
}

// Usage: <Loupe /> anywhere in your app, add CSS: body { cursor: none; }
```

---

## Common Loupe Problems

| Problem | Fix |
|---------|-----|
| Snapshot is blank | `useCORS: true` + serve assets with CORS headers |
| Snapshot misaligned after scroll | Subtract `window.scrollX/Y` from source coords |
| Lens blurry on retina | Cap DPR at 2 and scale canvas accordingly |
| html2canvas misses `::before`/`::after` | Known limitation — use screenshot API or puppeteer for complex cases |
| Canvas flickers | Use `requestAnimationFrame` loop, don't redraw on every `mousemove` |
| Snapshot captures the loupe canvas itself | Pass `ignoreElements: el => el === canvasRef.current` |
| Looks laggy | Increase lerp factor (0.15 → 0.3 = snappier) |
| Works on dev, breaks on prod | html2canvas can't cross iframe boundaries — use same-origin content |

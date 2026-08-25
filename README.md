# Wreath Scroll Scene — Setup Guide

Everything in this bundle is ready to drop into a Next.js or React project.

## What's inside

```
bundle/
├── WreathScrollScene.jsx    The React component
├── frames/
│   ├── desktop/             96 WebP frames @ 1920x1080 (~6.5MB total)
│   └── mobile/              96 WebP frames @ 1280x720  (~3.4MB total)
└── README.md                This file
```

## Installation

### 1. Install GSAP

```bash
npm install gsap
```

### 2. Copy the frames into your public directory

Place the `frames/` folder in `public/frames/` so the paths resolve as:
- `public/frames/desktop/frame_0001.webp` ... `frame_0096.webp`
- `public/frames/mobile/frame_0001.webp`  ... `frame_0096.webp`

### 3. Drop the component into your project

Put `WreathScrollScene.jsx` in your components directory, then import it into your landing page:

```jsx
import WreathScrollScene from "@/components/WreathScrollScene";

export default function HomePage() {
  return (
    <>
      <WreathScrollScene />

      {/* Your next section — starts with black background to match
          the final frame seamlessly */}
      <section style={{ background: "#000", color: "#f5f0e6" }}>
        {/* Demo, features, product copy... */}
      </section>
    </>
  );
}
```

## Tuning

Three values in the component control how the scene feels:

**`height: "300vh"`** on the container — how much scrolling completes the animation. 250vh feels snappy, 400vh feels cinematic. Start at 300vh.

**`scrub: 0.5`** on ScrollTrigger — how much the frame update lags behind the scroll. 0.3 feels responsive, 1.0 feels dreamlike. 0.5 is the usual sweet spot.

**`window.innerWidth < 768`** — the mobile breakpoint. Raise to 1024 if you want tablets to use the lighter mobile frames too.

## Performance notes

- First frame renders immediately on load (before the rest finish preloading), so LCP is fast
- Canvas with device-pixel-ratio scaling stays crisp on retina displays
- DPR is capped at 2 to avoid wasteful 3x rendering on ultra-high-DPI phones
- Total load: ~6.5MB desktop, ~3.4MB mobile — comparable to one large hero image

## Seamless transition into the next section

The final frame is pure black, so the section immediately below the scroll scene should also have a black background. The handoff will be invisible. Once you're in that section, you can introduce your cream-toned UI, product demo, feature sections, etc.

## If something feels off

**Animation feels too fast:** increase container height to 400vh.

**Scroll feels laggy:** lower scrub to 0.3 or 0.2.

**Frames look pixelated:** you may be on a >2x DPR display and hitting the cap. Raise the DPR cap to 3, but watch performance.

**Long blank flash on first load:** preloading is async. Consider adding a simple black overlay that fades out once the first frame loads, using the firstLoaded flag already in the component.

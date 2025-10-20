<script lang="ts">
  import '../app.css';
  import favicon from '$lib/assets/favicon.svg';
  import { onMount } from 'svelte';

  let { children } = $props();

  function readZenThemeCSSVariables() {
    const cs = getComputedStyle(document.documentElement);
    const bg = cs.getPropertyValue('--zen-main-browser-background').trim();
    const opacity = parseFloat(cs.getPropertyValue('--zen-background-opacity'));
    const texture = parseFloat(cs.getPropertyValue('--zen-grainy-background-opacity'));
    return { bg, opacity, texture };
  }

  interface Color { r: number; g: number; b: number; a: number; }

  function parseLinearGradientColors(gradientStr: string): Color[] {
    const regex = /rgba?\((\d+),\s*(\d+),\s*(\d+),?\s*([\d\.]+)?\)/g;
    const colors: Color[] = [];
    let m: RegExpExecArray | null;
    while ((m = regex.exec(gradientStr)) !== null) {
      colors.push({
        r: parseInt(m[1], 10),
        g: parseInt(m[2], 10),
        b: parseInt(m[3], 10),
        a: m[4] !== undefined ? parseFloat(m[4]) : 1
      });
    }
    return colors;
  }

  onMount(() => {
    const canvas = document.getElementById('zen-canvas')! as HTMLCanvasElement;
    const ctx = canvas.getContext('2d')!;

    const resize = () => {
      const dpr = window.devicePixelRatio || 1;
      const cssW = window.innerWidth;
      const cssH = window.innerHeight;
      canvas.style.width = cssW + 'px';
      canvas.style.height = cssH + 'px';
      canvas.width = Math.round(cssW * dpr);
      canvas.height = Math.round(cssH * dpr);
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    };

    const hasVars = () => {
      const cs = getComputedStyle(document.documentElement);
      const bg = cs.getPropertyValue('--zen-main-browser-background').trim();
      const op = cs.getPropertyValue('--zen-background-opacity').trim();
      const tx = cs.getPropertyValue('--zen-grainy-background-opacity').trim();
      return Boolean(bg && op && tx);
    };

    const drawIfReady = () => {
      if (!hasVars()) return;
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      const { bg, opacity, texture } = readZenThemeCSSVariables();
      const colors = parseLinearGradientColors(bg);
      if (!colors.length || !Number.isFinite(opacity) || !Number.isFinite(texture)) return;
      drawGradient(colors, opacity, ctx, canvas);
      drawTexture(texture, ctx, canvas);
    };

    const onTheme = () => {
      resize();
      drawIfReady();
    };

    // Redraw when the extension applies/updates the theme
    document.addEventListener('zen-theme-applied', onTheme);

    // Also redraw if the content script changes documentElement.style directly
    const mo = new MutationObserver(onTheme);
    mo.observe(document.documentElement, { attributes: true, attributeFilter: ['style'] });

    // Initial layout and first attempt to draw (will no-op until vars exist)
    resize();
    drawIfReady();

    const onResize = () => { resize(); drawIfReady(); };
    window.addEventListener('resize', onResize);

    return () => {
      document.removeEventListener('zen-theme-applied', onTheme);
      window.removeEventListener('resize', onResize);
      mo.disconnect();
    };
  });

  function drawGradient(
    colors: Color[],
    opacity: number,
    ctx: CanvasRenderingContext2D,
    canvas: HTMLCanvasElement
  ) {
    const dpr = window.devicePixelRatio || 1;
    const vw = canvas.width / dpr;
    const vh = canvas.height / dpr;
    const cx = vw / 2;
    const cy = vh / 2;
    const length = Math.hypot(vw, vh);
    const x1 = cx - length / 2;
    const y1 = cy - length / 2;
    const x2 = cx + length / 2;
    const y2 = cy + length / 2;

    const grad = ctx.createLinearGradient(x1, y1, x2, y2);
    const stops = colors.length;
    colors.forEach((c, i) => {
      const pos = stops === 1 ? 0 : i / (stops - 1);
      grad.addColorStop(pos, `rgba(${c.r},${c.g},${c.b},${Math.max(0, Math.min(1, c.a * opacity))})`);
    });

    ctx.fillStyle = grad;
    ctx.fillRect(0, 0, canvas.width, canvas.height);
  }

  function drawTexture(texture: number, ctx: CanvasRenderingContext2D, canvas: HTMLCanvasElement) {
    if (!(texture > 0)) return;
    const dpr = window.devicePixelRatio || 1;
    const vw = canvas.width / dpr;
    const vh = canvas.height / dpr;
    const centerX = vw / 2;
    const centerY = vh / 2;
    const radius = Math.min(centerX, centerY) * 0.8;
    const dotRadius = radius / 16;

    const activeDots = Math.round(Math.max(0, Math.min(1, texture)) * 16);

    ctx.fillStyle = `rgba(255, 255, 255, 0.05)`;
    for (let i = 0; i < activeDots; i++) {
      const angle = i * ((2 * Math.PI) / 16);
      const dotX = centerX + Math.cos(angle) * radius;
      const dotY = centerY + Math.sin(angle) * radius;
      ctx.beginPath();
      ctx.arc(dotX, dotY, dotRadius, 0, Math.PI * 2);
      ctx.fill();
    }
  }
</script>

<svelte:head>
  <link rel="icon" href={favicon} />
</svelte:head>

<div class="layout-container">
  <canvas id="zen-canvas"></canvas>
  <div class="content">
    {@render children?.()}
  </div>
</div>

<style>
  .layout-container {
    position: relative;
    width: 100vw;
    height: 100vh;
    margin: 0;
    padding: 0;
    overflow: hidden;
  }

  #zen-canvas {
    position: fixed;
    inset: 0;
    display: block;
    width: 100vw;
    height: 100vh;
    z-index: 0;           /* keep canvas visible behind content */
    pointer-events: none;
  }

  .content {
    position: relative;
    z-index: 1;
    width: 100%;
    height: 100%;
  }
</style>
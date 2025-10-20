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
	interface Color {
		r: number;
		g: number;
		b: number;
		a: number;
	}

	function parseLinearGradientColors(gradientStr: string): Color[] {
		const regex = /rgba?\((\d+),\s*(\d+),\s*(\d+),?\s*([\d\.]+)?\)/g;
		let match;
		const colors: Color[] = [];
		while ((match = regex.exec(gradientStr)) !== null) {
			colors.push({
				r: parseInt(match[1], 10),
				g: parseInt(match[2], 10),
				b: parseInt(match[3], 10),
				a: match[4] !== undefined ? parseFloat(match[4]) : 1
			});
		}
		return colors;
	}

	onMount(() => {
		const canvas = document.getElementById('zen-canvas')! as HTMLCanvasElement;
		const ctx = canvas.getContext('2d')!;

		function resize() {
			canvas.width = window.innerWidth;
			canvas.height = window.innerHeight;
		}

		function draw() {
			ctx.clearRect(0, 0, canvas.width, canvas.height);

			const { bg, opacity, texture } = readZenThemeCSSVariables();
			const colors = parseLinearGradientColors(bg);

			drawGradient(colors, opacity, ctx, canvas);
			drawTexture(texture, ctx, canvas);
		}

		const resizeHandler = () => {
			resize();
			draw();
		};

		window.addEventListener('resize', resizeHandler);

		resize();
		draw();

		return () => {
			window.removeEventListener('resize', resizeHandler);
		};
	});

	function drawGradient(
		colors: Color[],
		opacity: number,
		ctx: CanvasRenderingContext2D,
		canvas: HTMLCanvasElement
	) {
		const cx = canvas.width / 2;
		const cy = canvas.height / 2;
		const length = Math.sqrt(canvas.width * canvas.width + canvas.height * canvas.height);
		const x1 = cx - length / 2;
		const y1 = cy - length / 2;
		const x2 = cx + length / 2;
		const y2 = cy + length / 2;

		const grad = ctx.createLinearGradient(x1, y1, x2, y2);

		const stops = colors.length;
		colors.forEach((color: Color, i: number) => {
			const stopPos = i / (stops - 1);
			grad.addColorStop(stopPos, `rgba(${color.r},${color.g},${color.b},${color.a * opacity})`);
		});

		ctx.fillStyle = grad;
		ctx.fillRect(0, 0, canvas.width, canvas.height);
	}

	function drawTexture(texture: number, ctx: CanvasRenderingContext2D, canvas: HTMLCanvasElement) {
		if (texture <= 0) return;
		const centerX = canvas.width / 2;
		const centerY = canvas.height / 2;
		const radius = Math.min(centerX, centerY) * 0.8;
		const dotRadius = radius / 16;

		const activeDots = Math.round(texture * 16);

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
		position: absolute;
		top: 0;
		left: 0;
		display: block;
		width: 100vw;
		height: 100vh;
		z-index: -1;
	}

	.content {
		position: relative;
		z-index: 1;
		width: 100%;
		height: 100%;
	}
</style>

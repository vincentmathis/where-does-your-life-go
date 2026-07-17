# Where Does Your Life Go?

An interactive visualization of how you spend your remaining weeks. Scroll through categories like sleep, work, commute, and free time to see where your life actually goes.

**Live:** [life.vincentmathis.xyz](https://life.vincentmathis.xyz)

## Tech Stack

- [SvelteKit](https://kit.svelte.dev) + [Svelte 5](https://svelte.dev)
- [GSAP](https://gsap.com) for scroll-based animations
- [Tailwind CSS v4](https://tailwindcss.com)
- [Biome](https://biomejs.dev) for linting/formatting

## Development

```bash
bun install
bun run dev
```

## Building

```bash
bun run build
bun run preview
```

## Docker

```bash
docker build -t life .
docker run -p 3000:3000 life
```

## License

GPL-3.0

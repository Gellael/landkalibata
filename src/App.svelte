<script lang="ts">
  import { onMount, tick as svelteTick } from 'svelte';
  import { cubicOut } from 'svelte/easing';

  type Project = {
    id: string;
    name: string;
    kind: string;
    desc: string;
    points: string[];
    cta: string;
    url: string;
    tilt: number;
  };

  // ====== EDIT BAGIAN INI ======
  const instagram = 'https://www.instagram.com/kalibata.project';

  const projects: Project[] = [
    {
      id: 'songin',
      name: 'SongIn',
      kind: 'Game kartu online',
      desc: 'Main kartu Song-In langsung dari browser, gratis. Siapa yang skornya paling kecil, dia yang menang.',
      points: [
        'Tujuan: skor paling kecil di antara semua pemain.',
        'Song (−100 poin): habiskan semua kartu dengan memainkan seri baru.',
        'Leng (−50 poin): habiskan semua kartu dengan cara selain seri baru.'
      ],
      cta: 'Mainkan sekarang',
      url: 'https://songin.my.id',
      tilt: 6
    },
    {
      id: 'petakilat',
      name: 'PetaKilat',
      kind: 'Pembuat peta desa',
      desc: 'Susun peta administrasi desa atau kelurahan yang siap cetak, tanpa mengatur layout peta secara manual.',
      points: [
        'Pilih wilayah desa atau kelurahan, lalu atur tampilan petanya.',
        'Lengkapi judul, arah utara, skala, legenda, dan metadata.',
        'Ekspor hasilnya ke PNG atau PDF ukuran A4.'
      ],
      cta: 'Buat peta',
      url: 'https://petakilat.my.id',
      tilt: 40
    }
    // Tambah proyek baru: salin satu blok di atas, beri id baru, lalu tambahkan ilustrasinya di snippet "art".
  ];
  // =============================

  const bandText = projects.map((p) => p.name).join(' ✦ ') + ' ✦ Where ideas become projects ✦ ';

  let selected: string | null = null;
  $: current = projects.find((p) => p.id === selected) ?? null;

  let heroIn: HTMLDivElement;
  let heroScenes: HTMLDivElement[] = [];
  let bar: HTMLDivElement;
  let tilesEl: HTMLDivElement;
  let detailWrap: HTMLDivElement;
  let tiles: HTMLDivElement[] = [];
  let tileScenes: HTMLDivElement[] = [];
  let detailScene: HTMLDivElement | null = null;

  const reduced = () => matchMedia('(prefers-reduced-motion: reduce)').matches;

  async function choose(id: string) {
    selected = selected === id ? null : id;
    if (selected) {
      await svelteTick();
      detailWrap?.scrollIntoView({ behavior: reduced() ? 'auto' : 'smooth', block: 'start' });
    }
  }

  // Transisi masuk 3D untuk panel detail
  function rise(_node: Element, { duration = 650 } = {}) {
    return {
      duration: reduced() ? 0 : duration,
      easing: cubicOut,
      css: (t: number) =>
        `opacity:${t};transform-origin:50% 0;transform:perspective(1400px) rotateX(${(1 - t) * -30}deg) translateY(${(1 - t) * 70}px)`
    };
  }

  onMount(() => {
    if (reduced()) return () => {};

    const clamp = (v: number, a = 0, b = 1) => Math.min(b, Math.max(a, v));
    const wideQuery = matchMedia('(min-width: 900px) and (min-height: 600px)');
    const hasHover = matchMedia('(hover: hover)').matches;
    const doc = document.documentElement;
    let vh = innerHeight;
    let wide = wideQuery.matches;
    const onResize = () => { vh = innerHeight; wide = wideQuery.matches; };

    const mouse = { cx: -1, cy: -1, nx: 0, ny: 0, sx: 0, sy: 0 };
    const onPointer = (e: PointerEvent) => {
      mouse.cx = e.clientX;
      mouse.cy = e.clientY;
      mouse.nx = (e.clientX / innerWidth) * 2 - 1;
      mouse.ny = (e.clientY / innerHeight) * 2 - 1;
    };
    addEventListener('resize', onResize, { passive: true });
    if (hasHover) addEventListener('pointermove', onPointer, { passive: true });

    let heroS = 0;
    let sway = 0;
    const hov = projects.map(() => ({ x: 0, y: 0 }));
    const enterS = projects.map(() => 1);

    let last = performance.now();
    let raf = 0;
    const frame = (now: number) => {
      const dt = Math.min(now - last, 64);
      last = now;
      const k = 1 - Math.exp(-dt / 110); // makin besar pembagi, makin lembut
      const sec = now / 1000;

      bar.style.transform = `scaleX(${clamp(scrollY / Math.max(1, doc.scrollHeight - vh))})`;

      // Hero miring ke belakang saat bagian proyek naik menutupinya
      const heroT = wide ? clamp(scrollY / vh) : clamp(scrollY / (vh * 1.2)) * 0.6;
      heroS += (heroT - heroS) * k;
      heroIn.style.transform = `translate3d(0, ${-heroS * 5}%, ${-heroS * 300}px) rotateX(${heroS * 14}deg)`;
      heroIn.style.opacity = String(1 - heroS * 0.75);

      mouse.sx += (mouse.nx - mouse.sx) * k;
      mouse.sy += (mouse.ny - mouse.sy) * k;

      // Ilustrasi hero: melayang pelan, mengikuti kursor (atau bergoyang di layar sentuh)
      const hy = hasHover ? mouse.sx * 20 : Math.sin(sec / 1.8) * 10;
      const hx = hasHover ? mouse.sy * 10 : 0;
      const bob = Math.sin(sec * 1.1) * 8;
      heroScenes[0].style.transform = `translate3d(0, ${bob}px, 0) rotateX(${6 - hx}deg) rotateY(${-14 + hy}deg)`;
      heroScenes[1].style.transform = `translate3d(0, ${-bob}px, 0) rotateX(${40 - hx}deg) rotateY(${12 + hy}deg)`;

      // Kartu pilihan: masuk dengan miring 3D, lalu mengikuti kursor
      const wrapTop = tilesEl.getBoundingClientRect().top;
      projects.forEach((p, i) => {
        const r = tiles[i].getBoundingClientRect();
        const inside = hasHover && mouse.cx >= r.left && mouse.cx <= r.right && mouse.cy >= r.top && mouse.cy <= r.bottom;
        const nx = inside ? ((mouse.cx - r.left) / r.width) * 2 - 1 : 0;
        const ny = inside ? ((mouse.cy - r.top) / r.height) * 2 - 1 : 0;
        hov[i].x += (nx - hov[i].x) * k;
        hov[i].y += (ny - hov[i].y) * k;

        const enterT = clamp((wrapTop - vh * (0.6 - i * 0.05)) / (vh * 0.4));
        enterS[i] += (enterT - enterS[i]) * k;
        const e = enterS[i];

        tiles[i].style.transform = `perspective(1200px) translateZ(${-e * 220}px) rotateX(${e * 32 - hov[i].y * 8}deg) rotateY(${hov[i].x * 10}deg)`;
        tileScenes[i].style.transform = `rotateX(${p.tilt - hov[i].y * 14}deg) rotateY(${hov[i].x * 24}deg)`;
      });

      // Ilustrasi di panel detail
      if (detailScene && current) {
        sway += (Math.sin(scrollY / 160) * 10 - sway) * k;
        const ry = hasHover ? mouse.sx * 18 : sway;
        const rx = current.tilt - (hasHover ? mouse.sy * 8 : 0);
        detailScene.style.transform = `rotateX(${rx}deg) rotateY(${ry}deg)`;
      }

      raf = requestAnimationFrame(frame);
    };
    raf = requestAnimationFrame(frame);

    return () => {
      cancelAnimationFrame(raf);
      removeEventListener('resize', onResize);
      removeEventListener('pointermove', onPointer);
    };
  });
</script>

<svelte:head>
  <title>Kalibata Project: where ideas become projects</title>
  <meta name="description" content="Kalibata Project membuat produk digital dari ide, mulai dari game kartu sampai alat pembuat peta desa." />
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link href="https://fonts.googleapis.com/css2?family=Anton&family=Archivo:wght@400;600;700&display=swap" rel="stylesheet" />
</svelte:head>

<!-- Ilustrasi berlapis (tiap lapis punya kedalaman --z) -->
{#snippet art(id: string)}
  {#if id === 'songin'}
    <svg class="layer" style="--z:0px" viewBox="0 0 360 300">
      <g transform="rotate(-12 150 150)">
        <rect x="70" y="30" width="170" height="240" rx="16" fill="#d4181f" stroke="#fbf3ea" stroke-width="5" />
        <circle cx="155" cy="150" r="34" fill="none" stroke="#fbf3ea" stroke-width="5" />
        <path d="M155 128v44M133 150h44M139 134l32 32M171 134l-32 32" stroke="#fbf3ea" stroke-width="5" />
      </g>
    </svg>
    <svg class="layer lift" style="--z:60px" viewBox="0 0 360 300">
      <g transform="rotate(9 220 150)">
        <rect x="140" y="30" width="170" height="240" rx="16" fill="#fbf3ea" />
        <text x="158" y="78" font-family="Anton, sans-serif" font-size="46" fill="#7e0f14">A</text>
        <text x="158" y="116" font-size="30" fill="#7e0f14">♠</text>
      </g>
    </svg>
    <svg class="layer" style="--z:120px" viewBox="0 0 360 300">
      <g transform="rotate(9 220 150)">
        <text x="225" y="190" text-anchor="middle" font-size="84" fill="#d4181f">♥</text>
        <text x="292" y="256" text-anchor="end" font-family="Anton, sans-serif" font-size="32" fill="#7e0f14">−100</text>
      </g>
    </svg>
  {:else}
    <svg class="layer lift" style="--z:0px" viewBox="0 0 360 300">
      <rect x="20" y="20" width="320" height="260" rx="14" fill="#e9fbf0" />
      <g fill="none" stroke="#1fbf6b" stroke-width="3" stroke-linecap="round">
        <path d="M40 90c50-40 110 30 170-8s90-10 110 20" />
        <path d="M40 140c60-30 100 40 170 0s80 0 110 18" />
        <path d="M40 190c50-20 110 40 170 0s90 4 110 20" />
      </g>
    </svg>
    <svg class="layer" style="--z:40px" viewBox="0 0 360 300">
      <path d="M60 255c40-60 100-30 140-80s60-70 100-90" fill="none" stroke="#06361f" stroke-width="4" stroke-dasharray="2 10" stroke-linecap="round" />
      <path d="M44 262h90M44 256v12M89 258v8M134 256v12" stroke="#06361f" stroke-width="3" />
    </svg>
    <svg class="layer" style="--z:70px" viewBox="0 0 360 300">
      <text x="318" y="62" text-anchor="end" font-family="Anton, sans-serif" font-size="30" fill="#06361f">U ↑</text>
    </svg>
    <svg class="layer lift" style="--z:130px" viewBox="0 0 360 300">
      <path d="M205 176c-24-31-31-45-31-60a31 31 0 0162 0c0 15-7 29-31 60z" fill="#06361f" />
      <circle cx="205" cy="116" r="10" fill="#e9fbf0" />
    </svg>
  {/if}
{/snippet}

<div class="progress" bind:this={bar} aria-hidden="true"></div>

<header class="nav">
  <a class="brand" href="#top">Kalibata<br />Project</a>
  <nav aria-label="Navigasi utama">
    <a class="link" href="#proyek">Proyek</a>
    <a class="btn small" href={instagram} target="_blank" rel="noopener noreferrer">Instagram</a>
  </nav>
</header>

<main id="top">
  <section class="hero">
    <div class="hero-in" bind:this={heroIn}>
      <div class="hero-art" aria-hidden="true">
        <div class="stage hs a"><div class="scene" bind:this={heroScenes[0]} style="--base:6deg">{@render art('songin')}</div></div>
        <div class="stage hs b"><div class="scene" bind:this={heroScenes[1]} style="--base:40deg">{@render art('petakilat')}</div></div>
      </div>
      <h1>
        <span style="--d:0">Where ideas</span>
        <span style="--d:1">become</span>
        <span style="--d:2">projects.</span>
      </h1>
      <div class="hero-foot">
        <p>
          Kami mengubah ide menjadi produk digital yang bisa langsung dipakai: dari game kartu sampai alat pembuat
          peta desa.
        </p>
        <a class="btn" href="#proyek">Pilih proyek</a>
      </div>
    </div>
  </section>

  <div class="band" aria-hidden="true">
    <div class="band-track">
      {#each [0, 1] as g}
        <div class="grp">
          {#each [0, 1, 2] as j}<span>{bandText}</span>{/each}
        </div>
      {/each}
    </div>
  </div>

  <section id="proyek" class="pick" aria-label="Proyek">
    <div class="wrap">
      <div class="pick-head">
        <h2>Pilih satu.</h2>
        <p>Klik salah satu proyek untuk melihat isinya.</p>
      </div>

      <div class="tiles" bind:this={tilesEl}>
        {#each projects as p, i (p.id)}
          <div class="tile {p.id}" class:active={selected === p.id} bind:this={tiles[i]}>
            <div class="tile-art" aria-hidden="true">
              <div class="stage">
                <div class="scene" bind:this={tileScenes[i]} style="--base:{p.tilt}deg">
                  {@render art(p.id)}
                </div>
              </div>
            </div>
            <button class="pick-btn" aria-expanded={selected === p.id} aria-controls="detail" on:click={() => choose(p.id)}>
              <span class="tile-kind">{p.kind}</span>
              <span class="tile-name">{p.name}</span>
              <span class="tile-hint">{selected === p.id ? 'Sedang dibuka' : 'Lihat proyek'}<i class="plus"></i></span>
            </button>
          </div>
        {/each}
      </div>

      <div class="detail-wrap" id="detail" bind:this={detailWrap}>
        {#if current}
          {#key current.id}
            <article class="detail {current.id}" in:rise>
              <div class="copy">
                <p class="kind">{current.kind}</p>
                <h3>{current.name}</h3>
                <p class="desc">{current.desc}</p>
                <ul>
                  {#each current.points as pt}<li>{pt}</li>{/each}
                </ul>
                <div class="actions">
                  <a class="btn" href={current.url} target="_blank" rel="noopener noreferrer">
                    {current.cta}
                    <svg width="18" height="18" viewBox="0 0 16 16" fill="none" aria-hidden="true"><path d="M5 11l6-6M6 5h5v5" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" /></svg>
                  </a>
                  <button class="btn ghost" on:click={() => (selected = null)}>Tutup</button>
                </div>
              </div>
              <div class="art" aria-hidden="true">
                <div class="stage">
                  <div class="scene" bind:this={detailScene} style="--base:{current.tilt}deg">
                    {@render art(current.id)}
                  </div>
                </div>
              </div>
            </article>
          {/key}
        {/if}
      </div>
    </div>
  </section>

  <section class="closing">
    <div class="wrap">
      <h2>Punya ide yang ingin dijadikan proyek?</h2>
      <p>Ceritakan lewat DM Instagram kami.</p>
      <a class="btn" href={instagram} target="_blank" rel="noopener noreferrer">Kirim DM di @kalibata.project</a>
    </div>
  </section>
</main>

<footer>
  <span>© {new Date().getFullYear()} Kalibata Project</span>
  <a href={instagram} target="_blank" rel="noopener noreferrer">@kalibata.project</a>
</footer>

<style>
  :global(:root) {
    --red: #d4181f;
    --maroon: #7e0f14;
    --cream: #fbf3ea;
    --green: #1fbf6b;
    --forest: #06361f;
    --pad: clamp(1.25rem, 5vw, 4.5rem);
    --grain: url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='160' height='160'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'/><feColorMatrix values='0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .5 0'/></filter><rect width='100%' height='100%' filter='url(%23n)'/></svg>");
  }
  :global(html) { scroll-behavior: smooth; }
  :global(body) { margin: 0; background: var(--red); color: var(--cream); font-family: 'Archivo', system-ui, sans-serif; line-height: 1.55; overflow-x: hidden; -webkit-text-size-adjust: 100%; }
  h1, h2, h3, .brand, .kind, .tile-name, .band span { font-family: 'Anton', 'Archivo', sans-serif; font-weight: 400; }
  a { color: inherit; text-decoration: none; }
  :global(:focus-visible) { outline: 3px solid var(--cream); outline-offset: 3px; }
  .wrap { width: 100%; max-width: 1440px; margin-inline: auto; }

  .progress { position: fixed; inset: 0 0 auto 0; height: 5px; z-index: 20; background: var(--cream); mix-blend-mode: difference; transform: scaleX(0); transform-origin: 0 50%; pointer-events: none; }

  .btn {
    display: inline-flex; align-items: center; justify-content: center; gap: 0.5rem; align-self: flex-start; min-height: 48px; box-sizing: border-box;
    background: var(--cream); color: var(--maroon); font: inherit; font-weight: 700; border: 0; cursor: pointer;
    padding: 0.8rem 1.5rem; border-radius: 0.6rem; transition: transform 0.15s, box-shadow 0.15s;
  }
  .btn.small { min-height: 44px; padding: 0.5rem 1.1rem; }
  .btn.ghost { background: transparent; color: inherit; box-shadow: inset 0 0 0 2px currentColor; }
  .link { padding: 0.6rem 0.25rem; }
  @media (hover: hover) { .btn:hover { transform: translate(-2px, -2px); box-shadow: 4px 4px 0 rgba(0, 0, 0, 0.35); } }

  /* Navigasi */
  .nav { position: absolute; inset: 0 0 auto 0; z-index: 5; display: flex; justify-content: space-between; align-items: center; padding: 1rem var(--pad); }
  .brand { font-size: 1.35rem; line-height: 1; text-transform: uppercase; letter-spacing: 0.02em; }
  nav { display: flex; align-items: center; gap: 1rem; font-weight: 600; }

  /* Hero */
  .hero { position: relative; min-height: 100svh; box-sizing: border-box; display: flex; flex-direction: column; justify-content: flex-end; padding: 6.5rem var(--pad) 3rem; perspective: 1400px; overflow: hidden; }
  .hero::before, .closing::before { content: ''; position: absolute; inset: 0; background-image: var(--grain); opacity: 0.22; mix-blend-mode: multiply; pointer-events: none; }
  .hero-in { position: relative; display: grid; gap: 1.75rem 3rem; width: 100%; max-width: 1600px; margin-inline: auto; grid-template-areas: 'art' 'title' 'foot'; transform-origin: 50% 0; will-change: transform, opacity; }
  .hero-art { grid-area: art; position: relative; justify-self: center; width: min(100%, 420px, calc(34svh * 1.25)); aspect-ratio: 5 / 4; }
  .stage.hs { position: absolute; width: auto; }
  .stage.hs.a { right: 0; top: 0; width: 78%; }
  .stage.hs.b { left: 0; bottom: 0; width: 56%; }
  h1 { grid-area: title; margin: 0; text-transform: uppercase; font-size: clamp(3.4rem, 17vw, 9.5rem); line-height: 0.9; perspective: 900px; }
  h1 span { display: block; transform-origin: 50% 100%; animation: flip 1s cubic-bezier(0.2, 0.8, 0.2, 1) both; animation-delay: calc(var(--d) * 120ms); }
  @keyframes flip { from { opacity: 0; transform: rotateX(-75deg) translateY(30%); } to { opacity: 1; transform: none; } }
  .hero-foot { grid-area: foot; display: flex; flex-wrap: wrap; gap: 1.25rem 3rem; align-items: center; justify-content: space-between; border-top: 3px solid var(--cream); padding-top: 1.25rem; }
  .hero-foot p { margin: 0; max-width: 44ch; font-size: clamp(1rem, 1.6vw, 1.2rem); }

  /* Pita berjalan */
  .band { position: relative; z-index: 2; margin: -1.1rem -4vw -1.1rem; padding: 0.55rem 0; background: var(--maroon); color: var(--cream); transform: rotate(-2deg); overflow: hidden; white-space: nowrap; }
  .band-track { display: inline-flex; animation: slide 34s linear infinite; will-change: transform; }
  .grp { display: inline-flex; }
  .band span { font-size: clamp(1.3rem, 3.2vw, 2.2rem); text-transform: uppercase; padding-right: 1.5rem; }
  @keyframes slide { to { transform: translateX(-50%); } }

  /* Bagian pilih proyek */
  .pick, .closing, footer { position: relative; z-index: 1; }
  .pick { background: var(--cream); color: var(--maroon); padding: 4.5rem var(--pad) 5rem; }
  .pick-head { margin-bottom: 2rem; }
  .pick-head h2 { margin: 0; font-size: clamp(3.4rem, 12vw, 9.5rem); line-height: 0.9; text-transform: uppercase; }
  .pick-head p { margin: 0.75rem 0 0; font-size: clamp(1.05rem, 1.8vw, 1.35rem); font-weight: 600; }

  .tiles { display: grid; grid-template-columns: repeat(auto-fit, minmax(min(100%, 340px), 1fr)); gap: 1.25rem; perspective: 1400px; }
  .tile { position: relative; container-type: inline-size; min-height: clamp(340px, 56svh, 560px); border-radius: 1.75rem; overflow: hidden; will-change: transform; transition: outline-offset 0.2s; }
  .tile.songin { background: var(--maroon); color: var(--cream); --ring: var(--maroon); }
  .tile.petakilat { background: var(--green); color: var(--forest); --ring: var(--forest); }
  .tile.active { outline: 4px solid var(--ring); outline-offset: 6px; }
  .tile-art { position: absolute; right: -3%; bottom: -3%; width: 68%; pointer-events: none; }
  .pick-btn { all: unset; box-sizing: border-box; display: flex; flex-direction: column; align-items: flex-start; gap: 0.3rem; height: 100%; min-height: inherit; padding: clamp(1.25rem, 6cqw, 2.25rem); cursor: pointer; }
  .pick-btn::after { content: ''; position: absolute; inset: 0; border-radius: inherit; }
  .pick-btn:focus-visible::after { outline: 3px solid var(--cream); outline-offset: -8px; }
  .tile.petakilat .pick-btn:focus-visible::after { outline-color: var(--forest); }
  .tile-kind { font-weight: 700; opacity: 0.85; }
  .tile-name { font-size: clamp(2.8rem, 19cqw, 8rem); line-height: 0.9; text-transform: uppercase; }
  .tile-hint { margin-top: auto; display: inline-flex; align-items: center; background: var(--cream); color: var(--maroon); font-weight: 700; padding: 0.7rem 1.2rem; border-radius: 999px; transition: transform 0.2s; }
  .tile.petakilat .tile-hint { background: var(--forest); color: var(--cream); }
  .plus { position: relative; display: inline-block; width: 0.9em; height: 0.9em; margin-left: 0.65rem; transition: transform 0.3s; }
  .plus::before, .plus::after { content: ''; position: absolute; background: currentColor; border-radius: 2px; }
  .plus::before { left: 0; right: 0; top: calc(50% - 1px); height: 2px; }
  .plus::after { top: 0; bottom: 0; left: calc(50% - 1px); width: 2px; }
  .tile.active .plus { transform: rotate(45deg); }
  @media (hover: hover) { .tile:hover .tile-hint { transform: translateX(4px); } }

  /* Panel detail */
  .detail-wrap { scroll-margin-top: 1rem; }
  .detail { margin-top: 1.25rem; display: grid; grid-template-columns: 1fr; gap: 1.5rem; align-items: center; padding: clamp(1.25rem, 4vw, 3.5rem); border-radius: 1.75rem; }
  .detail .art { order: -1; }
  .detail.songin { background: var(--maroon); color: var(--cream); }
  .detail.petakilat { background: var(--green); color: var(--forest); }
  .detail.petakilat .btn { background: var(--forest); color: var(--cream); }
  .detail.petakilat .btn.ghost { background: transparent; color: var(--forest); }
  .copy { display: flex; flex-direction: column; gap: 1.1rem; }
  .kind { margin: 0; font-size: 1.1rem; letter-spacing: 0.04em; opacity: 0.85; }
  .detail h3 { margin: 0; font-size: clamp(3rem, 9vw, 7rem); line-height: 0.9; text-transform: uppercase; }
  .desc { margin: 0; max-width: 40ch; font-size: clamp(1.05rem, 1.7vw, 1.35rem); font-weight: 600; }
  ul { margin: 0; padding: 0 0 0 1.1rem; max-width: 46ch; display: grid; gap: 0.4rem; }
  .actions { display: flex; flex-wrap: wrap; gap: 0.75rem; }
  .art { display: grid; place-items: center; }

  /* Ilustrasi 3D berlapis */
  .stage { position: relative; width: min(100%, 460px); aspect-ratio: 360 / 300; perspective: 1100px; }
  .scene { position: absolute; inset: 0; transform-style: preserve-3d; transform: rotateX(var(--base, 0deg)); will-change: transform; }
  .layer { position: absolute; inset: 0; width: 100%; height: 100%; overflow: visible; transform: translateZ(var(--z, 0px)); }
  .layer.lift { filter: drop-shadow(0 16px 16px rgba(0, 0, 0, 0.28)); }

  /* Penutup */
  .closing { background: var(--red); padding: 5rem var(--pad); overflow: hidden; }
  .closing .wrap { position: relative; display: flex; flex-direction: column; gap: 1.25rem; }
  .closing h2 { margin: 0; text-transform: uppercase; font-size: clamp(2.6rem, 8vw, 7rem); line-height: 0.95; max-width: 14ch; }
  .closing p { margin: 0; font-size: 1.2rem; }
  footer { background: var(--maroon); display: flex; flex-wrap: wrap; justify-content: space-between; gap: 0.5rem; padding: 1.5rem var(--pad); font-size: 0.9rem; }

  /* Tablet ke atas */
  @media (min-width: 700px) {
    .hero-art { width: min(100%, 520px, calc(40svh * 1.25)); }
    .pick { padding-block: 5.5rem 6rem; }
  }
  @media (min-width: 900px) {
    .detail { grid-template-columns: 1.1fr 1fr; gap: 2.5rem; }
    .detail .art { order: 0; }
  }
  /* Desktop lebar: judul di kiri, ilustrasi 3D di kanan */
  @media (min-width: 1000px) {
    .hero-in { grid-template-columns: 1.15fr 1fr; grid-template-areas: 'title art' 'foot foot'; align-items: end; }
    .hero-art { width: min(100%, 600px, calc(52svh * 1.25)); justify-self: end; }
    h1 { font-size: clamp(3.6rem, 10.5vw, 10rem); }
  }
  /* Hero menempel, bagian proyek naik menutupinya */
  @media (min-width: 900px) and (min-height: 600px) {
    .hero { position: sticky; top: 0; z-index: 0; }
    .pick { border-radius: 2rem 2rem 0 0; }
  }
  /* HP kecil */
  @media (max-width: 599px) {
    .stage.hs.b { display: none; }
    .stage.hs.a { width: 100%; right: 0; }
    .actions .btn { flex: 1 1 100%; }
    .nav { gap: 0.5rem; }
    .brand { font-size: 1.2rem; }
  }
  @media (max-height: 520px) and (orientation: landscape) {
    .hero-art { display: none; }
  }
  @media (prefers-reduced-motion: reduce) {
    :global(html) { scroll-behavior: auto; }
    h1 span { animation: none; }
    .band-track { animation: none; }
    .btn, .tile, .tile-hint, .plus { transition: none; }
  }
</style>
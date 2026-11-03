<script lang="ts">
  import { onMount, tick as svelteTick } from 'svelte';
  import { cubicOut, quintOut } from 'svelte/easing';

  type Project = {
    id: string;
    name: string;
    kind: string;
    desc: string;
    points: string[];
    cta: string;
    url: string;
    tilt: number;
    accent: string;
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
      tilt: 6,
      accent: '#d4181f'
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
      tilt: 40,
      accent: '#1fbf6b'
    }
    // Tambah proyek baru: salin satu blok di atas, beri id baru, lalu tambahkan ilustrasinya di snippet "art".
  ];
  // =============================

  const bandText = projects.map((p) => p.name).join(' ✦ ') + ' ✦ Where ideas become projects ✦ ';

  let selected: string | null = null;
  $: current = projects.find((p) => p.id === selected) ?? null;

  // Hero slideshow state
  let slideIndex = 0;
  let slideProgress = 0;
  const SLIDE_DURATION = 4000; // ms per slide
  let slideInterval: ReturnType<typeof setInterval> | null = null;

  // Scroll-reveal tracking
  let revealEls: HTMLElement[] = [];
  let scrollY_local = 0;

  // Navbar scroll state
  let navScrolled = false;

  let heroIn: HTMLDivElement;
  let heroScenes: HTMLDivElement[] = [];
  let bar: HTMLDivElement;
  let tilesEl: HTMLDivElement;
  let detailWrap: HTMLDivElement;
  let tiles: HTMLDivElement[] = [];
  let tileScenes: HTMLDivElement[] = [];
  let detailScene: HTMLDivElement | null = null;

  // Floating particles
  let particles: { x: number; y: number; size: number; speed: number; opacity: number; delay: number }[] = [];

  const reduced = () => matchMedia('(prefers-reduced-motion: reduce)').matches;

  function initParticles() {
    particles = Array.from({ length: 20 }, () => ({
      x: Math.random() * 100,
      y: Math.random() * 100,
      size: Math.random() * 4 + 2,
      speed: Math.random() * 20 + 15,
      opacity: Math.random() * 0.3 + 0.1,
      delay: Math.random() * -20
    }));
  }

  function startSlideshow() {
    if (reduced()) return;
    slideInterval = setInterval(() => {
      slideIndex = (slideIndex + 1) % projects.length;
      slideProgress = 0;
    }, SLIDE_DURATION);
  }

  function goToSlide(i: number) {
    slideIndex = i;
    slideProgress = 0;
    if (slideInterval) clearInterval(slideInterval);
    startSlideshow();
  }

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

  // Scroll reveal transition
  function reveal(node: Element, { delay = 0, y = 60 } = {}) {
    return {
      duration: reduced() ? 0 : 800,
      delay: reduced() ? 0 : delay,
      easing: quintOut,
      css: (t: number) =>
        `opacity:${t};transform:translateY(${(1 - t) * y}px)`
    };
  }

  // Intersection observer for scroll reveals
  function observeReveal(node: HTMLElement) {
    if (reduced()) {
      node.style.opacity = '1';
      node.style.transform = 'none';
      return { destroy() {} };
    }
    node.style.opacity = '0';
    node.style.transform = 'translateY(50px)';
    node.style.transition = 'opacity 0.8s cubic-bezier(0.16, 1, 0.3, 1), transform 0.8s cubic-bezier(0.16, 1, 0.3, 1)';
    const io = new IntersectionObserver(
      (entries) => {
        entries.forEach((e) => {
          if (e.isIntersecting) {
            const d = parseFloat(node.dataset.delay || '0');
            setTimeout(() => {
              node.style.opacity = '1';
              node.style.transform = 'translateY(0)';
            }, d);
            io.unobserve(node);
          }
        });
      },
      { threshold: 0.15, rootMargin: '0px 0px -60px 0px' }
    );
    io.observe(node);
    return { destroy() { io.disconnect(); } };
  }

  onMount(() => {
    initParticles();
    startSlideshow();

    // Track scroll for navbar
    const onScroll = () => {
      scrollY_local = window.scrollY;
      navScrolled = scrollY_local > 60;
    };
    addEventListener('scroll', onScroll, { passive: true });

    if (reduced()) return () => {
      if (slideInterval) clearInterval(slideInterval);
      removeEventListener('scroll', onScroll);
    };

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

    // Slideshow progress tracker
    let slideStart = performance.now();

    let last = performance.now();
    let raf = 0;
    const frame = (now: number) => {
      const dt = Math.min(now - last, 64);
      last = now;
      const k = 1 - Math.exp(-dt / 110);
      const sec = now / 1000;

      // Slide progress (0–1)
      slideProgress = Math.min((now - slideStart) / SLIDE_DURATION, 1);

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
      if (heroScenes[0]) heroScenes[0].style.transform = `translate3d(0, ${bob}px, 0) rotateX(${6 - hx}deg) rotateY(${-14 + hy}deg)`;
      if (heroScenes[1]) heroScenes[1].style.transform = `translate3d(0, ${-bob}px, 0) rotateX(${40 - hx}deg) rotateY(${12 + hy}deg)`;

      // Kartu pilihan: masuk dengan miring 3D, lalu mengikuti kursor
      if (tilesEl) {
        const wrapTop = tilesEl.getBoundingClientRect().top;
        projects.forEach((p, i) => {
          if (!tiles[i]) return;
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
          if (tileScenes[i]) tileScenes[i].style.transform = `rotateX(${p.tilt - hov[i].y * 14}deg) rotateY(${hov[i].x * 24}deg)`;
        });
      }

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

    // Reset slideStart when slide changes
    const origGoToSlide = goToSlide;
    const slideObserver = setInterval(() => {
      slideStart = performance.now() - (slideProgress * SLIDE_DURATION);
    }, 100);

    return () => {
      cancelAnimationFrame(raf);
      removeEventListener('resize', onResize);
      removeEventListener('pointermove', onPointer);
      removeEventListener('scroll', onScroll);
      if (slideInterval) clearInterval(slideInterval);
      clearInterval(slideObserver);
    };
  });
</script>

<svelte:head>
  <title>Kalibata Project — Where Ideas Become Projects</title>
  <meta name="description" content="Kalibata Project membuat produk digital dari ide, mulai dari game kartu sampai alat pembuat peta desa. Since 2026." />
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link href="https://fonts.googleapis.com/css2?family=Anton&family=Archivo:wght@400;600;700;800&display=swap" rel="stylesheet" />
  {@html `<script type="application/ld+json">${JSON.stringify({
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Kalibata Project",
    "url": "https://kalibataproject.com",
    "logo": "https://kalibataproject.com/logo.png",
    "foundingDate": "2026",
    "description": "Kalibata Project membuat produk digital dari ide, mulai dari game kartu sampai alat pembuat peta desa.",
    "sameAs": ["https://www.instagram.com/kalibata.project"]
  })}</script>`}
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

<!-- Floating particles background -->
<div class="particles" aria-hidden="true">
  {#each particles as p}
    <div
      class="particle"
      style="left:{p.x}%;top:{p.y}%;width:{p.size}px;height:{p.size}px;opacity:{p.opacity};--speed:{p.speed}s;--delay:{p.delay}s"
    ></div>
  {/each}
</div>

<div class="progress" bind:this={bar} aria-hidden="true"></div>

<header class="nav" class:scrolled={navScrolled}>
  <a class="brand" href="#top">
    <img src="/logo.png" alt="Kalibata Project" class="brand-logo" />
    <span class="brand-text">Kalibata<br />Project</span>
  </a>
  <nav aria-label="Navigasi utama">
    <a class="link" href="#proyek">Proyek</a>
    <a class="btn small nav-btn" href={instagram} target="_blank" rel="noopener noreferrer">
      <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true"><path d="M12 2.163c3.204 0 3.584.012 4.85.07 3.252.148 4.771 1.691 4.919 4.919.058 1.265.069 1.645.069 4.849 0 3.205-.012 3.584-.069 4.849-.149 3.225-1.664 4.771-4.919 4.919-1.266.058-1.644.07-4.85.07-3.204 0-3.584-.012-4.849-.07-3.26-.149-4.771-1.699-4.919-4.92-.058-1.265-.07-1.644-.07-4.849 0-3.204.013-3.583.07-4.849.149-3.227 1.664-4.771 4.919-4.919 1.266-.057 1.645-.069 4.849-.069zM12 0C8.741 0 8.333.014 7.053.072 2.695.272.273 2.69.073 7.052.014 8.333 0 8.741 0 12c0 3.259.014 3.668.072 4.948.2 4.358 2.618 6.78 6.98 6.98C8.333 23.986 8.741 24 12 24c3.259 0 3.668-.014 4.948-.072 4.354-.2 6.782-2.618 6.979-6.98.059-1.28.073-1.689.073-4.948 0-3.259-.014-3.667-.072-4.947-.196-4.354-2.617-6.78-6.979-6.98C15.668.014 15.259 0 12 0zm0 5.838a6.162 6.162 0 100 12.324 6.162 6.162 0 000-12.324zM12 16a4 4 0 110-8 4 4 0 010 8zm6.406-11.845a1.44 1.44 0 100 2.881 1.44 1.44 0 000-2.881z"/></svg>
      Instagram
    </a>
  </nav>
</header>

<main id="top">
  <section class="hero">
    <!-- Animated gradient orbs -->
    <div class="hero-orbs" aria-hidden="true">
      <div class="orb orb-1"></div>
      <div class="orb orb-2"></div>
      <div class="orb orb-3"></div>
    </div>

    <div class="hero-in" bind:this={heroIn}>
      <!-- Slideshow art area -->
      <div class="hero-art" aria-hidden="true">
        {#each projects as p, i}
          <div class="slide-card" class:active={slideIndex === i} class:prev={slideIndex !== i}>
            <div class="stage hs single">
              <div class="scene" bind:this={heroScenes[i]} style="--base:{p.tilt}deg">
                {@render art(p.id)}
              </div>
            </div>
          </div>
        {/each}
        <!-- Slide indicators -->
        <div class="slide-indicators">
          {#each projects as p, i}
            <button
              class="indicator"
              class:active={slideIndex === i}
              on:click={() => goToSlide(i)}
              aria-label="Lihat {p.name}"
            >
              <span class="indicator-fill" style={slideIndex === i ? `animation: fillBar ${SLIDE_DURATION}ms linear forwards` : ''}></span>
              <span class="indicator-label">{p.name}</span>
            </button>
          {/each}
        </div>
      </div>

      <div class="hero-text">
        <p class="hero-tag" use:observeReveal data-delay="0">
          <span class="tag-dot"></span>
          Since 2026
        </p>
        <h1>
          <span style="--d:0">Where ideas</span>
          <span style="--d:1">become</span>
          <span style="--d:2">projects.</span>
        </h1>
        <div class="hero-foot">
          <p>
            Kami mengubah ide menjadi produk digital yang bisa langsung dipakai — dari game kartu sampai alat pembuat
            peta desa.
          </p>
          <div class="hero-actions">
            <a class="btn glow" href="#proyek">
              <span>Pilih proyek</span>
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none" aria-hidden="true"><path d="M10 4v12M10 16l5-5M10 16l-5-5" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/></svg>
            </a>
            <a class="btn ghost hero-ghost" href={instagram} target="_blank" rel="noopener noreferrer">Hubungi kami</a>
          </div>
        </div>
      </div>
    </div>

    <!-- Scroll hint -->
    <div class="scroll-hint" aria-hidden="true">
      <div class="scroll-mouse">
        <div class="scroll-wheel"></div>
      </div>
      <span>Scroll</span>
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
      <div class="pick-head" use:observeReveal>
        <p class="section-label">Proyek kami</p>
        <h2>Pilih satu.</h2>
        <p class="pick-sub">Klik salah satu proyek untuk melihat isinya.</p>
      </div>

      <div class="tiles" bind:this={tilesEl}>
        {#each projects as p, i (p.id)}
          <div class="tile {p.id}" class:active={selected === p.id} bind:this={tiles[i]}>
            <div class="tile-shimmer"></div>
            <div class="tile-art" aria-hidden="true">
              <div class="stage">
                <div class="scene" bind:this={tileScenes[i]} style="--base:{p.tilt}deg">
                  {@render art(p.id)}
                </div>
              </div>
            </div>
            <button class="pick-btn" aria-expanded={selected === p.id} aria-controls="detail" on:click={() => choose(p.id)}>
              <span class="tile-number">0{i + 1}</span>
              <span class="tile-kind">{p.kind}</span>
              <span class="tile-name">{p.name}</span>
              <span class="tile-hint">{selected === p.id ? 'Sedang dibuka' : 'Lihat proyek'}<i class="arrow-icon">→</i></span>
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
                  {#each current.points as pt, j}
                    <li style="--li-d:{j}">{pt}</li>
                  {/each}
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
    <div class="wrap" use:observeReveal>
      <p class="section-label light">Kolaborasi</p>
      <h2>Punya ide yang ingin dijadikan proyek?</h2>
      <p>Ceritakan lewat DM Instagram kami.</p>
      <a class="btn glow" href={instagram} target="_blank" rel="noopener noreferrer">
        Kirim DM di @kalibata.project
        <svg width="18" height="18" viewBox="0 0 16 16" fill="none" aria-hidden="true"><path d="M5 11l6-6M6 5h5v5" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" /></svg>
      </a>
    </div>
  </section>
</main>

<footer>
  <span>© {new Date().getFullYear()} Kalibata Project</span>
  <div class="footer-links">
    <a href={instagram} target="_blank" rel="noopener noreferrer">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true"><path d="M12 2.163c3.204 0 3.584.012 4.85.07 3.252.148 4.771 1.691 4.919 4.919.058 1.265.069 1.645.069 4.849 0 3.205-.012 3.584-.069 4.849-.149 3.225-1.664 4.771-4.919 4.919-1.266.058-1.644.07-4.85.07-3.204 0-3.584-.012-4.849-.07-3.26-.149-4.771-1.699-4.919-4.92-.058-1.265-.07-1.644-.07-4.849 0-3.204.013-3.583.07-4.849.149-3.227 1.664-4.771 4.919-4.919 1.266-.057 1.645-.069 4.849-.069zM12 0C8.741 0 8.333.014 7.053.072 2.695.272.273 2.69.073 7.052.014 8.333 0 8.741 0 12c0 3.259.014 3.668.072 4.948.2 4.358 2.618 6.78 6.98 6.98C8.333 23.986 8.741 24 12 24c3.259 0 3.668-.014 4.948-.072 4.354-.2 6.782-2.618 6.979-6.98.059-1.28.073-1.689.073-4.948 0-3.259-.014-3.667-.072-4.947-.196-4.354-2.617-6.78-6.979-6.98C15.668.014 15.259 0 12 0zm0 5.838a6.162 6.162 0 100 12.324 6.162 6.162 0 000-12.324zM12 16a4 4 0 110-8 4 4 0 010 8zm6.406-11.845a1.44 1.44 0 100 2.881 1.44 1.44 0 000-2.881z"/></svg>
      @kalibata.project
    </a>
  </div>
</footer>

<style>
  :global(:root) {
    --red: #d4181f;
    --maroon: #7e0f14;
    --cream: #fbf3ea;
    --green: #1fbf6b;
    --forest: #06361f;
    --pad: clamp(1rem, 5vw, 4.5rem);
    --grain: url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='160' height='160'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'/><feColorMatrix values='0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .5 0'/></filter><rect width='100%' height='100%' filter='url(%23n)'/></svg>");
  }
  :global(html) { scroll-behavior: smooth; }
  :global(*, *::before, *::after) { box-sizing: border-box; }
  :global(body) { margin: 0; background: var(--red); color: var(--cream); font-family: 'Archivo', system-ui, sans-serif; line-height: 1.55; overflow-x: hidden; -webkit-text-size-adjust: 100%; }
  h1, h2, h3, .brand-text, .kind, .tile-name, .band span { font-family: 'Anton', 'Archivo', sans-serif; font-weight: 400; }
  a { color: inherit; text-decoration: none; }
  :global(:focus-visible) { outline: 3px solid var(--cream); outline-offset: 3px; }
  .wrap { width: 100%; max-width: 1440px; margin-inline: auto; }

  /* Floating particles */
  .particles { position: fixed; inset: 0; z-index: 0; pointer-events: none; overflow: hidden; }
  .particle {
    position: absolute; border-radius: 50%; background: var(--cream);
    animation: float var(--speed) var(--delay) linear infinite;
  }
  @keyframes float {
    0% { transform: translateY(100vh) rotate(0deg); }
    100% { transform: translateY(-100px) rotate(720deg); }
  }

  .progress { position: fixed; inset: 0 0 auto 0; height: 4px; z-index: 30; background: linear-gradient(90deg, var(--cream), var(--green), var(--cream)); mix-blend-mode: difference; transform: scaleX(0); transform-origin: 0 50%; pointer-events: none; }

  /* Buttons */
  .btn {
    display: inline-flex; align-items: center; justify-content: center; gap: 0.5rem; align-self: flex-start; min-height: 48px; box-sizing: border-box;
    background: var(--cream); color: var(--maroon); font: inherit; font-weight: 700; border: 0; cursor: pointer;
    padding: 0.85rem 1.6rem; border-radius: 99px; transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
    position: relative; overflow: hidden;
  }
  .btn::before {
    content: ''; position: absolute; inset: 0; background: linear-gradient(135deg, rgba(255,255,255,0.4), transparent);
    opacity: 0; transition: opacity 0.3s;
  }
  .btn.glow {
    box-shadow: 0 0 0 0 rgba(251, 243, 234, 0), 0 4px 16px rgba(0,0,0,0.15);
  }
  .btn.small { min-height: 44px; padding: 0.5rem 1.2rem; font-size: 0.9rem; }
  .btn.ghost { background: transparent; color: inherit; box-shadow: inset 0 0 0 2px currentColor; }
  .btn.hero-ghost { color: var(--cream); }
  .link { padding: 0.6rem 0.25rem; position: relative; }
  .link::after { content: ''; position: absolute; bottom: 0.4rem; left: 0.25rem; right: 0.25rem; height: 2px; background: var(--cream); transform: scaleX(0); transition: transform 0.3s; transform-origin: right; }
  @media (hover: hover) {
    .btn:hover { transform: translateY(-3px); box-shadow: 0 0 30px rgba(251, 243, 234, 0.25), 0 8px 24px rgba(0,0,0,0.2); }
    .btn:hover::before { opacity: 1; }
    .btn.glow:hover { box-shadow: 0 0 40px rgba(251, 243, 234, 0.35), 0 8px 32px rgba(0,0,0,0.25); }
    .link:hover::after { transform: scaleX(1); transform-origin: left; }
  }

  /* Navigation — Glassmorphism */
  .nav {
    position: fixed; inset: 0 0 auto 0; z-index: 15; display: flex; justify-content: space-between; align-items: center;
    padding: 1rem var(--pad); transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
  }
  .nav.scrolled {
    background: rgba(126, 15, 20, 0.85); backdrop-filter: blur(20px) saturate(1.8);
    -webkit-backdrop-filter: blur(20px) saturate(1.8);
    padding-block: 0.7rem; box-shadow: 0 4px 30px rgba(0,0,0,0.15);
  }
  .brand { display: flex; align-items: center; gap: 0.6rem; }
  .brand-logo { width: 42px; height: 42px; border-radius: 10px; object-fit: cover; transition: transform 0.3s; }
  .nav.scrolled .brand-logo { width: 34px; height: 34px; }
  .brand-text { font-size: 1.1rem; line-height: 1; text-transform: uppercase; letter-spacing: 0.02em; }
  nav { display: flex; align-items: center; gap: 1.25rem; font-weight: 600; }
  .nav-btn { background: rgba(251, 243, 234, 0.15); color: var(--cream); border: 1px solid rgba(251, 243, 234, 0.3); backdrop-filter: blur(10px); }
  @media (hover: hover) { .nav-btn:hover { background: var(--cream); color: var(--maroon); } }

  /* Hero */
  .hero { position: relative; min-height: 100svh; box-sizing: border-box; display: flex; flex-direction: column; justify-content: flex-end; padding: 7rem var(--pad) 3rem; perspective: 1400px; overflow: hidden; }
  .hero::before { content: ''; position: absolute; inset: 0; background-image: var(--grain); opacity: 0.18; mix-blend-mode: multiply; pointer-events: none; z-index: 1; }
  .hero::after { content: ''; position: absolute; bottom: 0; left: 0; right: 0; height: 40%; background: linear-gradient(to top, rgba(126, 15, 20, 0.6), transparent); pointer-events: none; z-index: 1; }

  /* Hero gradient orbs */
  .hero-orbs { position: absolute; inset: 0; overflow: hidden; z-index: 0; }
  .orb { position: absolute; border-radius: 50%; filter: blur(80px); opacity: 0.4; animation: orbFloat 20s ease-in-out infinite; }
  .orb-1 { width: 500px; height: 500px; background: #ff6b35; top: -10%; right: -5%; animation-delay: 0s; }
  .orb-2 { width: 400px; height: 400px; background: #7e0f14; bottom: -5%; left: -5%; animation-delay: -7s; }
  .orb-3 { width: 300px; height: 300px; background: #fbf3ea; top: 40%; left: 30%; opacity: 0.15; animation-delay: -13s; }
  @keyframes orbFloat {
    0%, 100% { transform: translate(0, 0) scale(1); }
    33% { transform: translate(30px, -40px) scale(1.05); }
    66% { transform: translate(-20px, 20px) scale(0.95); }
  }

  .hero-in { position: relative; z-index: 2; display: grid; gap: 2rem 4rem; width: 100%; max-width: 1600px; margin-inline: auto; grid-template-areas: 'art' 'text'; transform-origin: 50% 0; will-change: transform, opacity; }
  .hero-art { grid-area: art; position: relative; justify-self: center; width: min(100%, 440px, calc(36svh * 1.25)); aspect-ratio: 5 / 4; }

  /* Slideshow */
  .slide-card { position: absolute; inset: 0; opacity: 0; transform: scale(0.9) rotateY(12deg); transition: all 0.8s cubic-bezier(0.16, 1, 0.3, 1); pointer-events: none; }
  .slide-card.active { opacity: 1; transform: scale(1) rotateY(0deg); pointer-events: auto; }
  .slide-card.prev { opacity: 0; transform: scale(0.85) rotateY(-12deg); }
  .stage.hs.single { position: absolute; inset: 0; width: 100%; }

  .slide-indicators { position: absolute; bottom: -2.5rem; left: 0; right: 0; display: flex; gap: 0.5rem; justify-content: center; }
  .indicator {
    all: unset; cursor: pointer; position: relative; height: 4px; flex: 1; max-width: 80px;
    background: rgba(251, 243, 234, 0.25); border-radius: 4px; overflow: hidden;
    transition: background 0.3s;
  }
  .indicator-fill { position: absolute; inset: 0; background: var(--cream); border-radius: inherit; transform: scaleX(0); transform-origin: left; }
  .indicator.active .indicator-fill { animation: fillBar 4s linear forwards; }
  @keyframes fillBar { to { transform: scaleX(1); } }
  .indicator-label { position: absolute; bottom: 10px; left: 50%; transform: translateX(-50%); font-size: 0.7rem; font-weight: 700; text-transform: uppercase; letter-spacing: 0.05em; white-space: nowrap; opacity: 0; transition: opacity 0.3s; }
  .indicator:hover .indicator-label, .indicator.active .indicator-label { opacity: 1; }

  .hero-text { grid-area: text; }
  .hero-tag { display: inline-flex; align-items: center; gap: 0.5rem; font-weight: 700; font-size: 0.9rem; letter-spacing: 0.08em; text-transform: uppercase; opacity: 0.8; margin: 0 0 0.5rem; }
  .tag-dot { width: 8px; height: 8px; border-radius: 50%; background: var(--cream); animation: pulse 2s ease-in-out infinite; }
  @keyframes pulse { 0%, 100% { opacity: 1; transform: scale(1); } 50% { opacity: 0.5; transform: scale(0.8); } }

  h1 { margin: 0; text-transform: uppercase; font-size: clamp(3.2rem, 16vw, 9rem); line-height: 0.88; perspective: 900px; letter-spacing: -0.02em; }
  h1 span { display: block; transform-origin: 50% 100%; animation: flip 1s cubic-bezier(0.2, 0.8, 0.2, 1) both; animation-delay: calc(var(--d) * 120ms); }
  @keyframes flip { from { opacity: 0; transform: rotateX(-75deg) translateY(30%); } to { opacity: 1; transform: none; } }

  .hero-foot { display: flex; flex-direction: column; gap: 1.5rem; margin-top: 1.75rem; padding-top: 1.5rem; border-top: 2px solid rgba(251, 243, 234, 0.3); }
  .hero-foot p { margin: 0; max-width: 48ch; font-size: clamp(1rem, 1.5vw, 1.15rem); line-height: 1.6; opacity: 0.9; }
  .hero-actions { display: flex; flex-wrap: wrap; gap: 0.75rem; }

  /* Scroll hint */
  .scroll-hint { position: absolute; bottom: 2rem; left: 50%; transform: translateX(-50%); z-index: 3; display: flex; flex-direction: column; align-items: center; gap: 0.5rem; animation: fadeInUp 1s 1.5s both; }
  .scroll-mouse { width: 26px; height: 40px; border: 2px solid rgba(251, 243, 234, 0.5); border-radius: 13px; position: relative; }
  .scroll-wheel { position: absolute; top: 8px; left: 50%; transform: translateX(-50%); width: 3px; height: 8px; background: var(--cream); border-radius: 3px; animation: scrollWheel 2s ease-in-out infinite; }
  @keyframes scrollWheel { 0% { opacity: 1; transform: translateX(-50%) translateY(0); } 100% { opacity: 0; transform: translateX(-50%) translateY(12px); } }
  .scroll-hint span { font-size: 0.7rem; text-transform: uppercase; letter-spacing: 0.15em; opacity: 0.5; }
  @keyframes fadeInUp { from { opacity: 0; transform: translateX(-50%) translateY(20px); } to { opacity: 1; transform: translateX(-50%) translateY(0); } }

  /* Pita berjalan */
  .band { position: relative; z-index: 2; margin: -1.1rem -4vw -1.1rem; padding: 0.7rem 0; background: var(--maroon); color: var(--cream); transform: rotate(-2deg); overflow: hidden; white-space: nowrap; }
  .band-track { display: inline-flex; animation: slide 34s linear infinite; will-change: transform; }
  .grp { display: inline-flex; }
  .band span { font-size: clamp(1.3rem, 3.2vw, 2.2rem); text-transform: uppercase; padding-right: 1.5rem; }
  @keyframes slide { to { transform: translateX(-50%); } }

  /* Section labels */
  .section-label { display: inline-block; font-weight: 800; font-size: 0.8rem; letter-spacing: 0.15em; text-transform: uppercase; margin: 0 0 0.75rem; padding: 0.4rem 1rem; border-radius: 99px; background: var(--maroon); color: var(--cream); }
  .section-label.light { background: rgba(251, 243, 234, 0.15); color: var(--cream); }

  /* Bagian pilih proyek */
  .pick, .closing, footer { position: relative; z-index: 1; }
  .pick { background: var(--cream); color: var(--maroon); padding: 5rem var(--pad) 5rem; overflow: hidden; }
  .pick-head { margin-bottom: 2.5rem; }
  .pick-head h2 { margin: 0; font-size: clamp(3.4rem, 12vw, 9.5rem); line-height: 0.9; text-transform: uppercase; }
  .pick-sub { margin: 0.75rem 0 0; font-size: clamp(1.05rem, 1.8vw, 1.35rem); font-weight: 600; opacity: 0.7; }

  .tiles { display: grid; grid-template-columns: repeat(auto-fit, minmax(min(100%, 340px), 1fr)); gap: 1.5rem; perspective: 1400px; }
  .tile {
    position: relative; container-type: inline-size; min-height: clamp(340px, 56svh, 560px);
    border-radius: 1.75rem; overflow: hidden; will-change: transform;
    transition: outline-offset 0.2s, box-shadow 0.4s;
  }
  .tile-shimmer {
    position: absolute; inset: 0; z-index: 1; pointer-events: none; opacity: 0;
    background: linear-gradient(105deg, transparent 40%, rgba(255,255,255,0.15) 45%, rgba(255,255,255,0.3) 50%, rgba(255,255,255,0.15) 55%, transparent 60%);
    background-size: 300% 100%; transition: opacity 0.4s;
  }
  .tile.songin { background: var(--maroon); color: var(--cream); --ring: var(--maroon); }
  .tile.petakilat { background: var(--green); color: var(--forest); --ring: var(--forest); }
  .tile.active { outline: 4px solid var(--ring); outline-offset: 6px; }
  .tile-art { position: absolute; right: -3%; bottom: -3%; width: 68%; pointer-events: none; transition: transform 0.6s cubic-bezier(0.16, 1, 0.3, 1); }
  .pick-btn { all: unset; box-sizing: border-box; display: flex; flex-direction: column; align-items: flex-start; gap: 0.35rem; height: 100%; min-height: inherit; padding: clamp(1.25rem, 6cqw, 2.25rem); cursor: pointer; position: relative; z-index: 2; }
  .pick-btn::after { content: ''; position: absolute; inset: 0; border-radius: inherit; }
  .pick-btn:focus-visible::after { outline: 3px solid var(--cream); outline-offset: -8px; }
  .tile.petakilat .pick-btn:focus-visible::after { outline-color: var(--forest); }
  .tile-number { font-size: 0.75rem; font-weight: 800; letter-spacing: 0.1em; opacity: 0.5; }
  .tile-kind { font-weight: 700; opacity: 0.85; font-size: 0.95rem; }
  .tile-name { font-size: clamp(2.8rem, 19cqw, 8rem); line-height: 0.9; text-transform: uppercase; }
  .tile-hint { margin-top: auto; display: inline-flex; align-items: center; gap: 0.5rem; background: var(--cream); color: var(--maroon); font-weight: 700; padding: 0.7rem 1.2rem; border-radius: 999px; transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1); }
  .tile.petakilat .tile-hint { background: var(--forest); color: var(--cream); }
  .arrow-icon { display: inline-block; transition: transform 0.3s; font-style: normal; }
  .tile.active .arrow-icon { transform: rotate(90deg); }
  @media (hover: hover) {
    .tile:hover .tile-hint { transform: translateX(6px); gap: 0.75rem; }
    .tile:hover .tile-art { transform: scale(1.05) rotate(-2deg); }
    .tile:hover .tile-shimmer { opacity: 1; animation: shimmer 1.5s ease-in-out; }
  }
  @keyframes shimmer { 0% { background-position: 100% 0; } 100% { background-position: -100% 0; } }

  /* Panel detail */
  .detail-wrap { scroll-margin-top: 1rem; }
  .detail { margin-top: 1.5rem; display: grid; grid-template-columns: 1fr; gap: 1.5rem; align-items: center; padding: clamp(1.25rem, 4vw, 3.5rem); border-radius: 1.75rem; }
  .detail .art { order: -1; }
  .detail.songin { background: var(--maroon); color: var(--cream); }
  .detail.petakilat { background: var(--green); color: var(--forest); }
  .detail.petakilat .btn { background: var(--forest); color: var(--cream); }
  .detail.petakilat .btn.ghost { background: transparent; color: var(--forest); }
  .copy { display: flex; flex-direction: column; gap: 1.1rem; }
  .kind { margin: 0; font-size: 1.1rem; letter-spacing: 0.04em; opacity: 0.85; }
  .detail h3 { margin: 0; font-size: clamp(3rem, 9vw, 7rem); line-height: 0.9; text-transform: uppercase; }
  .desc { margin: 0; max-width: 40ch; font-size: clamp(1.05rem, 1.7vw, 1.35rem); font-weight: 600; }
  ul { margin: 0; padding: 0 0 0 1.1rem; max-width: 46ch; display: grid; gap: 0.5rem; }
  li { animation: fadeInLi 0.5s cubic-bezier(0.16, 1, 0.3, 1) both; animation-delay: calc(var(--li-d) * 100ms + 300ms); }
  @keyframes fadeInLi { from { opacity: 0; transform: translateX(-10px); } to { opacity: 1; transform: none; } }
  .actions { display: flex; flex-wrap: wrap; gap: 0.75rem; }
  .art { display: grid; place-items: center; }

  /* Ilustrasi 3D berlapis */
  .stage { position: relative; width: min(100%, 460px); aspect-ratio: 360 / 300; perspective: 1100px; }
  .scene { position: absolute; inset: 0; transform-style: preserve-3d; transform: rotateX(var(--base, 0deg)); will-change: transform; }
  .layer { position: absolute; inset: 0; width: 100%; height: 100%; overflow: visible; transform: translateZ(var(--z, 0px)); }
  .layer.lift { filter: drop-shadow(0 16px 16px rgba(0, 0, 0, 0.28)); }

  /* Penutup */
  .closing { background: var(--red); padding: 6rem var(--pad); overflow: hidden; position: relative; }
  .closing::before { content: ''; position: absolute; inset: 0; background-image: var(--grain); opacity: 0.18; mix-blend-mode: multiply; pointer-events: none; }
  .closing .wrap { position: relative; display: flex; flex-direction: column; gap: 1.25rem; }
  .closing h2 { margin: 0; text-transform: uppercase; font-size: clamp(2.6rem, 8vw, 7rem); line-height: 0.95; max-width: 14ch; }
  .closing p { margin: 0; font-size: 1.2rem; opacity: 0.9; }

  /* Footer */
  footer { background: var(--maroon); display: flex; flex-wrap: wrap; justify-content: space-between; align-items: center; gap: 0.75rem; padding: 1.5rem var(--pad); font-size: 0.85rem; opacity: 0.9; }
  .footer-links a { display: inline-flex; align-items: center; gap: 0.4rem; transition: opacity 0.3s; }
  @media (hover: hover) { .footer-links a:hover { opacity: 0.7; } }

  /* ========== RESPONSIVE ========== */

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
    .hero-in { grid-template-columns: 1.15fr 1fr; grid-template-areas: 'text art'; align-items: center; }
    .hero-art { width: min(100%, 560px, calc(52svh * 1.25)); justify-self: end; }
    h1 { font-size: clamp(3.6rem, 10vw, 9.5rem); }
    .scroll-hint { left: var(--pad); transform: none; }
  }
  /* Hero menempel, bagian proyek naik menutupinya */
  @media (min-width: 900px) and (min-height: 600px) {
    .hero { position: sticky; top: 0; z-index: 0; }
    .pick { border-radius: 2rem 2rem 0 0; box-shadow: 0 -20px 60px rgba(0,0,0,0.15); }
  }
  /* HP kecil */
  @media (max-width: 599px) {
    .actions .btn { flex: 1 1 100%; }
    .nav { gap: 0.5rem; }
    .brand-text { font-size: 0.95rem; }
    .brand-logo { width: 34px; height: 34px; }
    .hero { padding: 5.5rem var(--pad) 2.5rem; }
    .hero-foot { gap: 1rem; }
    .hero-actions { flex-direction: column; width: 100%; }
    .hero-actions .btn { width: 100%; justify-content: center; }
    .pick { padding: 3.5rem var(--pad) 3.5rem; }
    .pick-head h2 { font-size: clamp(2.6rem, 14vw, 4rem); }
    .closing { padding: 3.5rem var(--pad); }
    .closing h2 { font-size: clamp(2rem, 9vw, 3.5rem); }
    .detail { padding: clamp(1rem, 4vw, 2rem); }
    .detail h3 { font-size: clamp(2.2rem, 12vw, 4rem); }
    .tiles { grid-template-columns: 1fr; }
    footer { flex-direction: column; align-items: center; text-align: center; padding: 1.2rem var(--pad); }
    .scroll-hint { display: none; }
    .slide-indicators { bottom: -2rem; }
    .orb-1 { width: 300px; height: 300px; }
    .orb-2 { width: 250px; height: 250px; }
    .orb-3 { display: none; }
  }
  /* HP sangat kecil (width <= 360px) */
  @media (max-width: 360px) {
    :global(:root) { --pad: 0.75rem; }
    h1 { font-size: clamp(2.4rem, 16vw, 3.4rem); }
    .brand-text { font-size: 0.85rem; }
    .tile-name { font-size: clamp(2.2rem, 16cqw, 4rem); }
    .pick-btn { padding: clamp(1rem, 4cqw, 1.5rem); }
  }
  @media (max-height: 520px) and (orientation: landscape) {
    .hero-art { display: none; }
    .hero-orbs { display: none; }
  }
  @media (prefers-reduced-motion: reduce) {
    :global(html) { scroll-behavior: auto; }
    h1 span { animation: none; }
    .band-track { animation: none; }
    .btn, .tile, .tile-hint, .arrow-icon { transition: none; }
    .particle, .orb { animation: none; }
    .scroll-hint { display: none; }
    .slide-card { transition: opacity 0.3s; transform: none !important; }
    .slide-card.active { transform: none !important; }
  }
</style>

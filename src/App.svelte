<script lang="ts">
  import { onMount, tick as svelteTick } from 'svelte';
  import { cubicOut } from 'svelte/easing';

  type Project = {
    id: string; name: string; kind: string; desc: string;
    points: string[]; cta: string; url: string; tilt: number;
    accent: string; icon: string; color: string; bg: string;
    price: string; paid: boolean;
  };

  const ig = 'https://www.instagram.com/kalibata.project';

  const projects: Project[] = [
    {
      id:'petakilat', name:'PetaKilat', kind:'Pembuat peta desa', icon:'map',
      desc:'Susun peta administrasi desa atau kelurahan yang siap cetak, tanpa mengatur layout peta secara manual.',
      points:['Pilih wilayah desa atau kelurahan, lalu atur tampilan petanya.','Lengkapi judul, arah utara, skala, legenda, dan metadata.','Ekspor hasilnya ke PNG atau PDF ukuran A4.','Berbasis kredit: tiap peta baru memakai kredit, saldo diisi lewat admin.'],
      cta:'Buat peta', url:'https://petakilat.my.id', tilt:40, accent:'#1fbf6b',
      color:'var(--forest)', bg:'var(--green)', price:'Pakai kredit', paid:true
    },
    {
      id:'songin', name:'SongIn', kind:'Game kartu online', icon:'card',
      desc:'Main kartu Song-In langsung dari browser, gratis. Siapa yang skornya paling kecil, dia yang menang.',
      points:['Tujuan: skor paling kecil di antara semua pemain.','Song (−100 poin): habiskan semua kartu dengan memainkan seri baru.','Leng (−50 poin): habiskan semua kartu dengan cara selain seri baru.'],
      cta:'Mainkan sekarang', url:'https://songin.my.id', tilt:6, accent:'#d4181f',
      color:'var(--cream)', bg:'var(--maroon)', price:'Gratis', paid:false
    }
  ];

  const stats = [
    { val: '2', label: 'Produk Aktif', icon: 'layers' },
    { val: '3', label: 'Pengembang', icon: 'users' },
    { val: '2026', label: 'Tahun Berdiri', icon: 'cal' },
  ];

  // GANTI nama, peran, dan inisial sesuai tim sebenarnya
  const team = [
    { name:'Ricardo Gellael', role:'', init:'N1', hue:'#ff6b35' },
    { name:'Rayhan MA', role:'', init:'N2', hue:'#1fbf6b' },
    { name:'Juan APP', role:'', init:'N3', hue:'#ffd36b' },
  ];

  const steps = [
    { t:'Top-up lewat admin', d:'Hubungi admin dan bayar secara manual.', icon:'send' },
    { t:'Kirim bukti bayar', d:'Kirim screenshot bukti pembayaran ke admin.', icon:'check' },
    { t:'Kredit masuk akun', d:'Admin menambahkan kredit ke akun PetaKilat-mu.', icon:'coin' },
    { t:'Pakai per peta', d:'Setiap peta baru memakai kredit dari saldomu.', icon:'map' },
  ];

  const bandText = projects.map(p => p.name).join(' ✦ ') + ' ✦ Where ideas become projects ✦ ';

  // State
  let mode: 'browse' | 'detail' = 'browse';
  let activeIdx = 0;
  let openedId: string | null = null;
  const SLIDE_MS = 4500;
  let timer: ReturnType<typeof setInterval> | null = null;
  let transitioning = false;

  let bar: HTMLDivElement;
  let navScrolled = false;
  let detailEl: HTMLDivElement;
  let detailScene: HTMLDivElement | null = null;
  let projEl: HTMLElement;
  let heroInner: HTMLDivElement;
  let heroBg: HTMLDivElement;
  let bandEl: HTMLDivElement;
  let showTop = false;
  let touchX = 0;
  let particles: {x:number;y:number;s:number;sp:number;o:number;d:number}[] = [];

  const noMo = () => matchMedia('(prefers-reduced-motion:reduce)').matches;

  function mkPart() { particles = Array.from({length:12},()=>({x:Math.random()*100,y:Math.random()*100,s:Math.random()*3+1.5,sp:Math.random()*25+18,o:Math.random()*0.14+0.04,d:Math.random()*-20})); }

  function startAuto() { stopAuto(); if(noMo()) return; timer=setInterval(()=>{ nextSlide(); }, SLIDE_MS); }
  function stopAuto() { if(timer){clearInterval(timer);timer=null;} }

  function nextSlide() {
    if(transitioning) return;
    transitioning = true;
    setTimeout(()=>{ activeIdx = (activeIdx+1)%projects.length; transitioning=false; }, 50);
  }

  function goSlide(i:number) { activeIdx=i; stopAuto(); startAuto(); }

  async function openProject(id:string) {
    stopAuto(); openedId=id; mode='detail';
    await svelteTick();
    detailEl?.scrollIntoView({behavior:noMo()?'auto':'smooth',block:'start'});
  }

  function backToSlideshow() {
    mode='browse'; openedId=null; activeIdx=0; startAuto();
    projEl?.scrollIntoView({behavior:noMo()?'auto':'smooth',block:'start'});
  }

  $: cur = projects.find(p=>p.id===openedId)??null;

  function rise(n:Element,{duration=700}={}) {
    return {duration:noMo()?0:duration,easing:cubicOut,
      css:(t:number)=>`opacity:${t};transform:perspective(1200px) rotateX(${(1-t)*-14}deg) translateY(${(1-t)*60}px) scale(${.93+t*.07})`};
  }

  function onTS(e:TouchEvent){ touchX=e.touches[0].clientX; }
  function onTE(e:TouchEvent){
    const dx=e.changedTouches[0].clientX-touchX;
    if(Math.abs(dx)>45) goSlide((activeIdx+(dx<0?1:-1)+projects.length)%projects.length);
  }

  // angka naik pelan saat masuk layar
  function count(node:HTMLElement) {
    const target=parseInt(node.textContent||'',10);
    if(noMo()||isNaN(target)||target<10) return{destroy(){}};
    node.textContent='0';
    const io=new IntersectionObserver(es=>{
      if(!es[0].isIntersecting) return; io.disconnect();
      const t0=performance.now(),D=1500;
      const step=(n:number)=>{const t=Math.min((n-t0)/D,1);node.textContent=String(Math.round(target*cubicOut(t)));if(t<1)requestAnimationFrame(step);};
      requestAnimationFrame(step);
    },{threshold:.6});
    io.observe(node);return{destroy(){io.disconnect();}};
  }

  // kartu miring mengikuti kursor + kilau cahaya (hanya perangkat dengan hover)
  function tilt(node:HTMLElement) {
    if(noMo()||!matchMedia('(hover:hover)').matches) return{destroy(){}};
    const mv=(e:PointerEvent)=>{const r=node.getBoundingClientRect(),x=(e.clientX-r.left)/r.width,y=(e.clientY-r.top)/r.height;
      node.style.setProperty('--gx',(x*100).toFixed(1)+'%');node.style.setProperty('--gy',(y*100).toFixed(1)+'%');
      node.style.setProperty('--rx',((.5-y)*9).toFixed(2)+'deg');node.style.setProperty('--ry',((x-.5)*11).toFixed(2)+'deg');};
    const lv=()=>{node.style.setProperty('--rx','0deg');node.style.setProperty('--ry','0deg');};
    node.addEventListener('pointermove',mv);node.addEventListener('pointerleave',lv);
    return{destroy(){node.removeEventListener('pointermove',mv);node.removeEventListener('pointerleave',lv);}};
  }

  function obs(node:HTMLElement) {
    if(noMo()){node.style.opacity='1';node.style.transform='none';return{destroy(){}};}
    const ease='cubic-bezier(.16,1,.3,1)';
    node.style.opacity='0';node.style.transform='translateY(44px) scale(.97)';node.style.filter='blur(6px)';
    node.style.transition=`opacity .8s ${ease},transform .8s ${ease},filter .8s ${ease}`;
    const io=new IntersectionObserver(es=>{es.forEach(e=>{if(e.isIntersecting){
      const d=parseFloat(node.dataset.delay||'0');
      setTimeout(()=>{
        node.style.opacity='1';node.style.transform='translateY(0) scale(1)';node.style.filter='blur(0)';
        setTimeout(()=>{['transform','transition','filter'].forEach(k=>node.style.removeProperty(k));},900);
      },d);
      io.unobserve(node);
    }});},{threshold:.1,rootMargin:'0px 0px -40px 0px'});
    io.observe(node);return{destroy(){io.disconnect();}};
  }

  // tombol sedikit "tertarik" ke kursor (hanya perangkat dengan hover)
  function magnet(node:HTMLElement) {
    if(noMo()||!matchMedia('(hover:hover)').matches) return{destroy(){}};
    const mv=(e:PointerEvent)=>{const r=node.getBoundingClientRect();
      node.style.translate=`${((e.clientX-r.left-r.width/2)*.18).toFixed(1)}px ${((e.clientY-r.top-r.height/2)*.3).toFixed(1)}px`;};
    const lv=()=>{node.style.translate='0 0';};
    node.addEventListener('pointermove',mv);node.addEventListener('pointerleave',lv);
    return{destroy(){node.removeEventListener('pointermove',mv);node.removeEventListener('pointerleave',lv);}};
  }

  onMount(()=>{
    mkPart(); startAuto();
    const onScroll=()=>{navScrolled=scrollY>50;showTop=scrollY>700;};
    addEventListener('scroll',onScroll,{passive:true});
    if(noMo()) return()=>{stopAuto();removeEventListener('scroll',onScroll);};
    const clamp=(v:number,a=0,b=1)=>Math.min(b,Math.max(a,v));
    const doc=document.documentElement;let vh=innerHeight;
    const hov=matchMedia('(hover:hover)').matches;
    const m={nx:0,ny:0,sx:0,sy:0};
    const onR=()=>{vh=innerHeight;};
    const onP=(e:PointerEvent)=>{m.nx=(e.clientX/innerWidth)*2-1;m.ny=(e.clientY/innerHeight)*2-1;};
    addEventListener('resize',onR,{passive:true});
    if(hov) addEventListener('pointermove',onP,{passive:true});
    let raf=0,last=performance.now(),lastY=scrollY,vel=0;
    const frame=(now:number)=>{
      const dt=Math.min(now-last,64);last=now;const k=1-Math.exp(-dt/120);
      const y=scrollY,pr=clamp(y/vh);
      if(bar)bar.style.transform=`scaleX(${clamp(y/Math.max(1,doc.scrollHeight-vh))})`;
      if(heroInner)heroInner.style.setProperty('--p',pr.toFixed(3));
      // parallax hero: teks naik & memudar, orb bergerak lebih lambat
      if(heroInner&&y<vh*1.2){heroInner.style.transform=`translate3d(0,${pr*60}px,0) scale(${1-pr*.07})`;heroInner.style.opacity=String(Math.max(0,1-pr*1.2));}
      if(heroBg&&y<vh*1.2)heroBg.style.transform=`translate3d(0,${y*.22}px,0)`;
      // pita berjalan miring mengikuti kecepatan scroll
      vel+=(clamp((y-lastY)*.3,-7,7)-vel)*k;lastY=y;
      if(projEl)projEl.style.setProperty('--v',vel.toFixed(2));
      if(bandEl)bandEl.style.transform=`rotate(-2deg) skewX(${(-vel).toFixed(2)}deg)`;
      if(!hov){m.nx=Math.sin(now/2600)*.7;m.ny=Math.cos(now/3100)*.5;}
      m.sx+=(m.nx-m.sx)*k;m.sy+=(m.ny-m.sy)*k;
      if(heroBg){heroBg.style.setProperty('--mx',(50+m.sx*45).toFixed(1)+'%');heroBg.style.setProperty('--my',(45+m.sy*45).toFixed(1)+'%');}
      if(detailScene&&cur){const sw=Math.sin(now/2200)*5;detailScene.style.transform=`rotateX(${cur.tilt*.25-(hov?m.sy*5:0)}deg) rotateY(${hov?m.sx*14:sw}deg)`;}
      raf=requestAnimationFrame(frame);
    };
    raf=requestAnimationFrame(frame);
    return()=>{cancelAnimationFrame(raf);stopAuto();removeEventListener('resize',onR);removeEventListener('pointermove',onP);removeEventListener('scroll',onScroll);};
  });
</script>

<svelte:head>
  <title>Kalibata Project — Where Ideas Become Projects</title>
  <meta name="description" content="Kalibata Project membuat produk digital dari ide, mulai dari alat pembuat peta desa sampai game kartu. Since 2026." />
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link href="https://fonts.googleapis.com/css2?family=Anton&family=Archivo:wght@400;600;700;800&display=swap" rel="stylesheet" />
  {@html `<script type="application/ld+json">${JSON.stringify({"@context":"https://schema.org","@type":"Organization","name":"Kalibata Project","url":"https://kalibataproject.my.id","logo":"https://kalibataproject.my.id/logo.png","foundingDate":"2026","description":"Kalibata Project membuat produk digital dari ide, mulai dari alat pembuat peta desa sampai game kartu.","sameAs":["https://www.instagram.com/kalibata.project"]})}</script>`}
</svelte:head>

{#snippet art(id: string)}
  {#if id === 'songin'}
    <svg class="layer" style="--z:0px" viewBox="0 0 360 300"><g transform="rotate(-12 150 150)"><rect x="70" y="30" width="170" height="240" rx="16" fill="#d4181f" stroke="#fbf3ea" stroke-width="5"/><circle cx="155" cy="150" r="34" fill="none" stroke="#fbf3ea" stroke-width="5"/><path d="M155 128v44M133 150h44M139 134l32 32M171 134l-32 32" stroke="#fbf3ea" stroke-width="5"/></g></svg>
    <svg class="layer lift fx" style="--z:60px;--fd:-1.2s" viewBox="0 0 360 300"><g transform="rotate(9 220 150)"><rect x="140" y="30" width="170" height="240" rx="16" fill="#fbf3ea"/><text x="158" y="78" font-family="Anton,sans-serif" font-size="46" fill="#7e0f14">A</text><path d="M170 98l14 18-14 18-14-18z" fill="#7e0f14"/></g></svg>
    <svg class="layer fx" style="--z:120px;--fs:3.2s;--fd:-2.4s" viewBox="0 0 360 300"><g transform="rotate(9 220 150)"><path d="M225 215c-30-26-48-40-48-62 0-30 36-38 48-12 12-26 48-18 48 12 0 22-18 36-48 62z" fill="#d4181f"/><text x="292" y="256" text-anchor="end" font-family="Anton,sans-serif" font-size="32" fill="#7e0f14">−100</text></g></svg>
  {:else}
    <svg class="layer lift" style="--z:0px" viewBox="0 0 360 300"><rect x="20" y="20" width="320" height="260" rx="14" fill="#e9fbf0"/><g class="wave" fill="none" stroke="#1fbf6b" stroke-width="3" stroke-linecap="round"><path d="M40 90c50-40 110 30 170-8s90-10 110 20"/><path d="M40 140c60-30 100 40 170 0s80 0 110 18"/><path d="M40 190c50-20 110 40 170 0s90 4 110 20"/></g></svg>
    <svg class="layer" style="--z:40px" viewBox="0 0 360 300"><path class="route" d="M60 255c40-60 100-30 140-80s60-70 100-90" fill="none" stroke="#06361f" stroke-width="4" stroke-dasharray="2 10" stroke-linecap="round"/><path d="M44 262h90M44 256v12M89 258v8M134 256v12" stroke="#06361f" stroke-width="3"/><ellipse class="ring" cx="205" cy="178" rx="12" ry="4.5" fill="none" stroke="#06361f" stroke-width="3"/></svg>
    <svg class="layer" style="--z:70px" viewBox="0 0 360 300"><text x="318" y="62" text-anchor="end" font-family="Anton,sans-serif" font-size="30" fill="#06361f">U ↑</text></svg>
    <svg class="layer lift fx" style="--z:130px;--fs:2.4s" viewBox="0 0 360 300"><path d="M205 176c-24-31-31-45-31-60a31 31 0 0162 0c0 15-7 29-31 60z" fill="#06361f"/><circle cx="205" cy="116" r="10" fill="#e9fbf0"/></svg>
    <svg class="layer lift fx" style="--z:170px;--fs:3s;--fd:-.8s" viewBox="0 0 360 300"><g transform="translate(292 218)"><circle r="26" fill="#ffd36b" stroke="#06361f" stroke-width="4"/><circle r="17" fill="none" stroke="#06361f" stroke-width="2.5" stroke-dasharray="3 4"/><text y="9" text-anchor="middle" font-family="Anton,sans-serif" font-size="24" fill="#06361f">K</text></g></svg>
  {/if}
{/snippet}

{#snippet icon(n: string)}
  <svg class="ico" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
    {#if n==='card'}<rect x="5" y="3" width="14" height="18" rx="2.5"/><path d="M12 8l3 4-3 4-3-4z"/>
    {:else if n==='map'}<path d="M9 4L3 6v14l6-2 6 2 6-2V4l-6 2z"/><path d="M9 4v14M15 6v14"/>
    {:else if n==='layers'}<path d="M12 3l9 5-9 5-9-5z"/><path d="M3 13l9 5 9-5"/>
    {:else if n==='cal'}<rect x="3" y="5" width="18" height="16" rx="2"/><path d="M3 10h18M8 3v4M16 3v4"/>
    {:else if n==='bulb'}<path d="M9 18h6M10 21h4M12 3a6 6 0 00-3.5 10.9c.6.5 1 1.2 1 2.1h5c0-.9.4-1.6 1-2.1A6 6 0 0012 3z"/>
    {:else if n==='bolt'}<path d="M13 2L4 14h7l-1 8 9-12h-7z"/>
    {:else if n==='pen'}<path d="M4 20l1-4L16 5l3 3L8 19z"/><path d="M14 7l3 3"/>
    {:else if n==='coin'}<circle cx="12" cy="12" r="9"/><path d="M12 7v10M9.5 9.5c0-1 1.1-1.7 2.5-1.7s2.5.7 2.5 1.7-1 1.5-2.5 1.8-2.5.8-2.5 1.8 1.1 1.7 2.5 1.7 2.5-.7 2.5-1.7"/>
    {:else if n==='users'}<circle cx="9" cy="8" r="3.5"/><path d="M2.5 20c0-3.6 2.9-6 6.5-6s6.5 2.4 6.5 6"/><circle cx="17.5" cy="9" r="2.6"/><path d="M17 14.2c2.8.2 4.5 2.1 4.5 4.8"/>
    {:else if n==='send'}<path d="M21 3L10 14M21 3l-7 18-4-7-7-4z"/>
    {:else if n==='check'}<circle cx="12" cy="12" r="9"/><path d="M8 12.5l3 3 5-6"/>
    {:else}<rect x="3" y="9" width="18" height="12" rx="1.5"/><path d="M12 9v12M3 13h18M12 9C9 9 7 8 7 6s3-2 5 3c2-5 5-5 5-3s-2 3-5 3"/>{/if}
  </svg>
{/snippet}

<div class="particles" aria-hidden="true">{#each particles as p}<div class="particle" style="left:{p.x}%;top:{p.y}%;width:{p.s}px;height:{p.s}px;opacity:{p.o};--sp:{p.sp}s;--dl:{p.d}s"></div>{/each}</div>
<div class="progress" bind:this={bar} aria-hidden="true"></div>

<header class="nav" class:scrolled={navScrolled}>
  <a class="brand" href="#top"><img src="/logo.png" alt="Kalibata Project" class="brand-logo"/><span class="brand-text">Kalibata<br/>Project</span></a>
  <nav>
    <a class="nav-link hide-xs" href="#about">Tentang</a>
    <a class="nav-link" href="#proyek">Proyek</a>
    <a class="nav-link hide-xs" href="#tim">Tim</a>
    <a class="btn sm nav-btn" href={ig} target="_blank" rel="noopener noreferrer">
      <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor"><path d="M12 2.163c3.204 0 3.584.012 4.85.07 3.252.148 4.771 1.691 4.919 4.919.058 1.265.069 1.645.069 4.849 0 3.205-.012 3.584-.069 4.849-.149 3.225-1.664 4.771-4.919 4.919-1.266.058-1.644.07-4.85.07-3.204 0-3.584-.012-4.849-.07-3.26-.149-4.771-1.699-4.919-4.92-.058-1.265-.07-1.644-.07-4.849 0-3.204.013-3.583.07-4.849.149-3.227 1.664-4.771 4.919-4.919 1.266-.057 1.645-.069 4.849-.069zM12 0C8.741 0 8.333.014 7.053.072 2.695.272.273 2.69.073 7.052.014 8.333 0 8.741 0 12c0 3.259.014 3.668.072 4.948.2 4.358 2.618 6.78 6.98 6.98C8.333 23.986 8.741 24 12 24c3.259 0 3.668-.014 4.948-.072 4.354-.2 6.782-2.618 6.979-6.98.059-1.28.073-1.689.073-4.948 0-3.259-.014-3.667-.072-4.947-.196-4.354-2.617-6.78-6.979-6.98C15.668.014 15.259 0 12 0zm0 5.838a6.162 6.162 0 100 12.324 6.162 6.162 0 000-12.324zM12 16a4 4 0 110-8 4 4 0 010 8zm6.406-11.845a1.44 1.44 0 100 2.881 1.44 1.44 0 000-2.881z"/></svg>
      IG
    </a>
  </nav>
</header>

<main id="top">
  <!-- HERO -->
  <section class="hero">
    <div class="hero-bg" aria-hidden="true" bind:this={heroBg}><div class="orb o1"></div><div class="orb o2"></div>
      <svg class="topo" viewBox="0 0 800 600" preserveAspectRatio="xMidYMid slice"><defs><path id="cl" vector-effect="non-scaling-stroke" d="M0-90c50-12 105 14 115 62s-30 90-90 105-140-8-150-62 40-93 125-105z"/></defs>
        <g transform="translate(650 170)"><g class="contour">{#each [.45,.8,1.15,1.55,2,2.5] as k}<use href="#cl" transform="scale({k})"/>{/each}</g></g>
        <g transform="translate(120 470)"><g class="contour c2">{#each [.4,.75,1.1,1.5,1.95] as k}<use href="#cl" transform="scale({k})"/>{/each}</g></g>
      </svg>
    </div>
    <div class="hero-inner" bind:this={heroInner}>
      <p class="tag"><span class="tdot"></span> Since 2026</p>
      <h1><span style="--d:0">Where ideas</span><span style="--d:1">become</span><span style="--d:2">projects.</span></h1>
      <p class="hero-sub">Mengubah ide menjadi produk digital yang bisa langsung dipakai.</p>
      <div class="hero-btns">
        <a class="btn glow" use:magnet href="#proyek">Lihat proyek ↓</a>
        <a class="btn ghost hg" href={ig} target="_blank" rel="noopener noreferrer">Hubungi kami</a>
      </div>
    </div>
    <div class="scroll-h" aria-hidden="true"><div class="smouse"><div class="swheel"></div></div></div>
  </section>

  <div class="band" aria-hidden="true" bind:this={bandEl}><div class="band-t">{#each [0,1] as _}<div class="grp">{#each [0,1,2] as _}<span>{bandText}</span>{/each}</div>{/each}</div></div>

  <!-- ===== ABOUT / PORTFOLIO ===== -->
  <section id="about" class="about">
    <div class="wrap">
      <div class="about-grid">
        <div class="about-text" use:obs>
          <p class="label">Tentang kami</p>
          <h2>Kalibata<br/>Project</h2>
          <p class="about-desc">Kami adalah tim kecil yang percaya bahwa ide bagus layak diwujudkan. Kalibata Project membangun produk digital dari tools pembuat peta sampai game kartu yang bisa langsung dipakai siapa saja. Ada yang gratis, ada yang memakai kredit.</p>
          <p class="about-desc">Setiap proyek kami dimulai dari satu pertanyaan sederhana: <strong>"Bagaimana kalau ini ada?"</strong></p>
        </div>
        <div class="about-stats" use:obs data-delay="200">
          {#each stats as s, i}
            <div class="stat" style="--si:{i}">
              <span class="stat-icon">{@render icon(s.icon)}</span>
              <span class="stat-val" use:count>{s.val}</span>
              <span class="stat-label">{s.label}</span>
            </div>
          {/each}
        </div>
      </div>
      <!-- Mission cards -->
      <div class="mission-row">
        <div class="mission-card" use:obs data-delay="0">
          <span class="mc-icon">{@render icon('bolt')}</span>
          <h4>Cepat & Ringan</h4>
          <p>Produk kami dirancang ringan, cepat diakses langsung dari browser tanpa install.</p>
        </div>
        <div class="mission-card" use:obs data-delay="140">
          <span class="mc-icon">{@render icon('pen')}</span>
          <h4>Desain Jelas</h4>
          <p>UI yang bersih dan intuitif — tidak perlu baca manual untuk mulai menggunakan.</p>
        </div>
        <div class="mission-card" use:obs data-delay="280">
          <span class="mc-icon">{@render icon('coin')}</span>
          <h4>Adil & Transparan</h4>
          <p>SongIn gratis. PetaKilat memakai kredit, jadi kamu membayar sesuai pemakaian.</p>
        </div>
      </div>
    </div>
  </section>

  <!-- ===== PROYEK — SEMUA PROJECT ===== -->
  <section id="proyek" class="proj" bind:this={projEl}>
    <div class="wrap">
      <div class="proj-head" use:obs>
        <p class="label">Proyek kami</p>
        <h2>Semua<br/>proyek.</h2>
        <p class="proj-sub">Pilih kartu untuk lihat detailnya, atau geser untuk proyek lainnya.</p>
      </div>

      {#if mode === 'browse'}
        <div class="clothesline">
          <!-- Tali -->
          <svg class="rope" viewBox="0 0 1000 70" preserveAspectRatio="none" aria-hidden="true">
            <line x1="20" y1="0" x2="20" y2="30" stroke="var(--maroon)" stroke-width="4" opacity=".35"/>
            <line x1="980" y1="0" x2="980" y2="24" stroke="var(--maroon)" stroke-width="4" opacity=".35"/>
            <path d="M20 30 Q280 62 500 42 Q720 22 980 24" fill="none" stroke="var(--maroon)" stroke-width="3" stroke-linecap="round" opacity=".45"/>
            <path d="M20 33 Q280 65 500 45 Q720 25 980 27" fill="none" stroke="rgba(0,0,0,.06)" stroke-width="5" stroke-linecap="round"/>
          </svg>

          <!-- 3D Carousel cards -->
          <div class="carousel" on:touchstart|passive={onTS} on:touchend|passive={onTE}>
            {#each projects as p, i (p.id)}
              <button
                class="ccard"
                class:active={activeIdx===i}
                class:left={activeIdx>i}
                class:right={activeIdx<i}
                on:click={() => activeIdx===i ? openProject(p.id) : goSlide(i)}
                aria-label="{p.name}"
                style="--ci:{i};--rot:{i%2===0?3:-4}deg"
              >
                <!-- Pin -->
                <div class="cpin">
                  <div class="pin-wood"><div class="pin-top"></div><div class="pin-coil"></div><div class="pin-l"></div><div class="pin-r"></div></div>
                </div>
                <div class="cstring"></div>

                <!-- Card -->
                <div class="cbody" use:tilt style="background:{p.bg};color:{p.color}">
                  <div class="cbody-glow"></div>
                  <span class="badge" class:paid={p.paid}>{#if p.paid}{@render icon('coin')}{/if}{p.price}</span>
                  <span class="cemoji">{@render icon(p.icon)}</span>
                  <span class="cnum">0{i+1}</span>
                  <span class="ckind">{p.kind}</span>
                  <h3 class="cname">{p.name}</h3>

                  <!-- Art — only visible when active -->
                  <div class="cart">
                    <div class="stage cstage"><div class="scene" style="--base:{p.tilt*.3}deg">{@render art(p.id)}</div></div>
                  </div>

                  <p class="cdesc">{p.desc}</p>
                  <span class="ccta">
                    {#if activeIdx===i}
                      Klik untuk buka detail →
                    {:else}
                      Klik untuk melihat
                    {/if}
                  </span>
                </div>
              </button>
            {/each}
          </div>

          <!-- Nav arrows + dots -->
          <div class="slide-nav">
            <button class="arrow" on:click={()=>goSlide((activeIdx-1+projects.length)%projects.length)} aria-label="Sebelumnya">
              <svg width="24" height="24" viewBox="0 0 24 24" fill="none"><path d="M15 18l-6-6 6-6" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
            </button>
            <div class="dots">
              {#each projects as p, i}
                <button class="dot" class:on={activeIdx===i} on:click={()=>goSlide(i)} aria-label={p.name}>
                  <span class="dot-fill" class:run={activeIdx===i}></span>
                  <span class="dot-label">{p.name}</span>
                </button>
              {/each}
            </div>
            <button class="arrow" on:click={()=>goSlide((activeIdx+1)%projects.length)} aria-label="Berikutnya">
              <svg width="24" height="24" viewBox="0 0 24 24" fill="none"><path d="M9 6l6 6-6 6" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
            </button>
          </div>
        </div>
      {/if}

      {#if mode==='detail' && cur}
        <div class="det-wrap" bind:this={detailEl}>
          {#key cur.id}
            <article class="det" style="background:{cur.bg};color:{cur.color}" in:rise>
              <div class="det-top">
                <button class="back" on:click={backToSlideshow}>← Semua proyek</button>
              </div>
              <div class="det-body">
                <div class="det-copy">
                  <span class="det-num">0{projects.findIndex(x=>x.id===cur.id)+1}</span>
                  <span class="badge inl" class:paid={cur.paid}>{#if cur.paid}{@render icon('coin')}{/if}{cur.price}</span>
                  <p class="det-kind">{cur.kind}</p>
                  <h3>{cur.name}</h3>
                  <p class="det-desc">{cur.desc}</p>
                  <ul>{#each cur.points as pt,j}<li style="--ld:{j}">{pt}</li>{/each}</ul>
                  {#if cur.paid}
                    <div class="cred">
                      <p class="cred-t">Cara kerja kredit</p>
                      <div class="csteps">
                        {#each steps as st,k}
                          <div class="cstep" style="--ld:{k}">
                            <span class="cstep-ico">{@render icon(st.icon)}</span>
                            <div class="cstep-txt"><strong>{st.t}</strong><span>{st.d}</span></div>
                          </div>
                        {/each}
                      </div>
                    </div>
                  {/if}
                  <div class="det-btns">
                    <a class="btn" href={cur.url} target="_blank" rel="noopener noreferrer">{cur.cta} ↗</a>
                    <button class="btn ghost" on:click={backToSlideshow}>Kembali</button>
                  </div>
                </div>
                <div class="det-art"><div class="stage"><div class="scene" bind:this={detailScene} style="--base:{cur.tilt}deg">{@render art(cur.id)}</div></div></div>
              </div>
            </article>
          {/key}
        </div>
      {/if}
    </div>
  </section>

  <!-- ===== TIM PENGEMBANG ===== -->
  <section id="tim" class="team">
    <div class="wrap">
      <div class="team-head" use:obs>
        <p class="label">Tim pengembang</p>
        <h2>Tiga<br/>kepala.</h2>
        <p class="proj-sub">Orang-orang di balik Kalibata Project.</p>
      </div>
      <div class="team-grid">
        {#each team as m, i}
          <div class="tslot" use:obs data-delay={i*140}>
            <article class="tcard" use:tilt style="--h:{m.hue};--ti:{i}">
              <div class="avatar"><span class="a-ring"></span><span class="a-init">{m.init}</span></div>
              <span class="t-num">0{i+1}</span>
              <h3>{m.name}</h3>
              <p class="t-role">{m.role}</p>
            </article>
          </div>
        {/each}
      </div>
    </div>
  </section>

  <!-- CLOSING -->
  <section class="closing">
    <div class="wrap" use:obs>
      <p class="label light">Kolaborasi</p>
      <h2>Punya ide?<br/>Yuk wujudkan.</h2>
      <p>Ceritakan lewat DM Instagram kami.</p>
      <a class="btn glow" use:magnet href={ig} target="_blank" rel="noopener noreferrer">Kirim DM @kalibata.project ↗</a>
    </div>
  </section>
</main>

<a class="totop" class:show={showTop} href="#top" aria-label="Kembali ke atas" tabindex={showTop?0:-1}>
  <svg width="20" height="20" viewBox="0 0 24 24" fill="none"><path d="M6 15l6-6 6 6" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
</a>

<footer>
  <span>© {new Date().getFullYear()} Kalibata Project</span>
  <a href={ig} target="_blank" rel="noopener noreferrer" class="fig">
    <svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><path d="M12 2.163c3.204 0 3.584.012 4.85.07 3.252.148 4.771 1.691 4.919 4.919.058 1.265.069 1.645.069 4.849 0 3.205-.012 3.584-.069 4.849-.149 3.225-1.664 4.771-4.919 4.919-1.266.058-1.644.07-4.85.07-3.204 0-3.584-.012-4.849-.07-3.26-.149-4.771-1.699-4.919-4.92-.058-1.265-.07-1.644-.07-4.849 0-3.204.013-3.583.07-4.849.149-3.227 1.664-4.771 4.919-4.919 1.266-.057 1.645-.069 4.849-.069zM12 0C8.741 0 8.333.014 7.053.072 2.695.272.273 2.69.073 7.052.014 8.333 0 8.741 0 12c0 3.259.014 3.668.072 4.948.2 4.358 2.618 6.78 6.98 6.98C8.333 23.986 8.741 24 12 24c3.259 0 3.668-.014 4.948-.072 4.354-.2 6.782-2.618 6.979-6.98.059-1.28.073-1.689.073-4.948 0-3.259-.014-3.667-.072-4.947-.196-4.354-2.617-6.78-6.979-6.98C15.668.014 15.259 0 12 0zm0 5.838a6.162 6.162 0 100 12.324 6.162 6.162 0 000-12.324zM12 16a4 4 0 110-8 4 4 0 010 8zm6.406-11.845a1.44 1.44 0 100 2.881 1.44 1.44 0 000-2.881z"/></svg>
    @kalibata.project
  </a>
</footer>

<style>
  :global(:root){--red:#d4181f;--maroon:#7e0f14;--cream:#fbf3ea;--green:#1fbf6b;--forest:#06361f;--pad:clamp(1rem,5vw,4.5rem)}
  :global(html){scroll-behavior:smooth}
  :global(body){margin:0;background:var(--red);color:var(--cream);font-family:'Archivo',system-ui,sans-serif;line-height:1.55;overflow-x:hidden;-webkit-text-size-adjust:100%}
  h1,h2,h3,h4,.brand-text,.cname,.band span{font-family:'Anton','Archivo',sans-serif;font-weight:400}
  a{color:inherit;text-decoration:none}:global(:focus-visible){outline:3px solid var(--cream);outline-offset:3px}
  .wrap{width:100%;max-width:1440px;margin-inline:auto;padding-inline:var(--pad)}
  .particles{position:fixed;inset:0;z-index:0;pointer-events:none;overflow:hidden}
  .particle{position:absolute;border-radius:50%;background:var(--cream);animation:fl var(--sp) var(--dl) linear infinite}
  @keyframes fl{0%{transform:translateY(100vh)}100%{transform:translateY(-80px) rotate(600deg)}}
  .progress{position:fixed;inset:0 0 auto 0;height:3px;z-index:30;background:linear-gradient(90deg,var(--cream),var(--green),var(--cream));transform:scaleX(0);transform-origin:0 50%;pointer-events:none}

  .btn{display:inline-flex;align-items:center;justify-content:center;gap:.4rem;min-height:44px;padding:.7rem 1.4rem;border-radius:99px;background:var(--cream);color:var(--maroon);font:inherit;font-weight:700;font-size:.9rem;border:0;cursor:pointer;transition:all .3s cubic-bezier(.16,1,.3,1)}
  .btn.glow{box-shadow:0 4px 16px rgba(0,0,0,.1)}.btn.sm{min-height:36px;padding:.35rem .9rem;font-size:.8rem}
  .btn.ghost{background:0;color:inherit;box-shadow:inset 0 0 0 2px currentColor}.btn.hg{color:var(--cream)}
  @media(hover:hover){.btn:hover{transform:translateY(-2px);box-shadow:0 0 24px rgba(251,243,234,.15),0 6px 20px rgba(0,0,0,.12)}}

  .nav{position:fixed;inset:0 0 auto 0;z-index:15;display:flex;justify-content:space-between;align-items:center;padding:.75rem var(--pad);transition:all .4s cubic-bezier(.16,1,.3,1)}
  .nav.scrolled{background:rgba(126,15,20,.92);backdrop-filter:blur(16px) saturate(1.4);-webkit-backdrop-filter:blur(16px) saturate(1.4);padding-block:.45rem;box-shadow:0 2px 16px rgba(0,0,0,.1)}
  .brand{display:flex;align-items:center;gap:.45rem}.brand-logo{width:32px;height:32px;border-radius:7px;object-fit:cover;transition:all .3s}
  .nav.scrolled .brand-logo{width:24px;height:24px}.brand-text{font-size:.85rem;line-height:1;text-transform:uppercase;letter-spacing:.02em}
  nav{display:flex;align-items:center;gap:.8rem;font-weight:600;font-size:.82rem}
  .nav-link{padding:.35rem .1rem;position:relative}.nav-link::after{content:'';position:absolute;bottom:.1rem;left:0;right:0;height:2px;background:var(--cream);transform:scaleX(0);transition:transform .3s;transform-origin:right}
  @media(hover:hover){.nav-link:hover::after{transform:scaleX(1);transform-origin:left}}
  .nav-btn{background:rgba(251,243,234,.1);color:var(--cream);border:1px solid rgba(251,243,234,.15)}
  @media(hover:hover){.nav-btn:hover{background:var(--cream);color:var(--maroon)}}

  .hero{position:relative;min-height:100svh;display:flex;align-items:center;justify-content:center;padding:5rem var(--pad) 3rem;overflow:hidden;text-align:center}
  .hero::before{content:'';position:absolute;inset:0;background-image:url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='160' height='160'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'/><feColorMatrix values='0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .5 0'/></filter><rect width='100%' height='100%' filter='url(%23n)'/></svg>");opacity:.12;mix-blend-mode:multiply;pointer-events:none;z-index:1}
  .hero-bg{position:absolute;inset:0;overflow:hidden;z-index:0}
  .orb{position:absolute;border-radius:50%;filter:blur(90px);opacity:.25;animation:ob 24s ease-in-out infinite}
  .o1{width:40vw;max-width:400px;aspect-ratio:1;background:#ff6b35;top:-8%;right:-5%}.o2{width:34vw;max-width:340px;aspect-ratio:1;background:#7e0f14;bottom:-6%;left:-4%;animation-delay:-10s}
  @keyframes ob{0%,100%{transform:translate(0,0) scale(1)}33%{transform:translate(18px,-28px) scale(1.03)}66%{transform:translate(-10px,10px) scale(.97)}}
  .hero-inner{position:relative;z-index:2;max-width:640px}
  .tag{display:inline-flex;align-items:center;gap:.4rem;font-weight:700;font-size:.7rem;letter-spacing:.1em;text-transform:uppercase;opacity:.55;margin:0 0 .6rem;animation:fu .8s .2s both}
  .tdot{width:6px;height:6px;border-radius:50%;background:var(--cream);animation:pu 2s ease-in-out infinite}
  @keyframes pu{0%,100%{opacity:1;transform:scale(1)}50%{opacity:.3;transform:scale(.6)}}
  h1{margin:0;text-transform:uppercase;font-size:clamp(2.4rem,11vw,7rem);line-height:.87;letter-spacing:-.02em;perspective:900px}
  h1 span{display:block;transform-origin:50% 100%;animation:fp 1s cubic-bezier(.2,.8,.2,1) both;animation-delay:calc(var(--d)*130ms)}
  @keyframes fp{from{opacity:0;transform:rotateX(-75deg) translateY(30%)}to{opacity:1;transform:none}}
  .hero-sub{margin:1rem auto 0;max-width:44ch;font-size:clamp(.85rem,1.2vw,.98rem);line-height:1.6;opacity:.7;animation:fu .8s .5s both}
  .hero-btns{display:flex;flex-wrap:wrap;gap:.55rem;justify-content:center;margin-top:1.3rem;animation:fu .8s .7s both}
  @keyframes fu{from{opacity:0;transform:translateY(18px)}to{opacity:1;transform:none}}
  .scroll-h{position:absolute;bottom:1rem;left:50%;transform:translateX(-50%);z-index:3;animation:fu 1s 1.5s both}
  .smouse{width:16px;height:26px;border:2px solid rgba(251,243,234,.28);border-radius:8px;position:relative}
  .swheel{position:absolute;top:5px;left:50%;transform:translateX(-50%);width:2px;height:5px;background:var(--cream);border-radius:2px;animation:sw 2s ease-in-out infinite}
  @keyframes sw{0%{opacity:1;transform:translateX(-50%)}100%{opacity:0;transform:translateX(-50%) translateY(7px)}}

  .hero-inner,.hero-bg,.band{will-change:transform}
  .band{position:relative;z-index:2;margin:-1rem -4vw;padding:.45rem 0;background:var(--maroon);color:var(--cream);transform:rotate(-2deg);overflow:hidden;white-space:nowrap}
  .band-t{display:inline-flex;animation:sl 36s linear infinite}.grp{display:inline-flex}
  .band span{font-size:clamp(.95rem,2.4vw,1.5rem);text-transform:uppercase;padding-right:1.3rem}
  @keyframes sl{to{transform:translateX(-50%)}}

  .label{display:inline-block;font-weight:800;font-size:.68rem;letter-spacing:.12em;text-transform:uppercase;margin:0 0 .4rem;padding:.25rem .75rem;border-radius:99px;background:var(--maroon);color:var(--cream)}
  .label.light{background:rgba(251,243,234,.1)}

  /* ===== ABOUT ===== */
  .about{position:relative;z-index:1;background:var(--cream);color:var(--maroon);padding:4.5rem 0 3rem;overflow:hidden;scroll-margin-top:0}
  .about-grid{display:grid;grid-template-columns:1fr;gap:2.5rem;margin-bottom:2.5rem}
  .about-text h2{margin:0;font-size:clamp(2.8rem,9vw,6.5rem);line-height:.88;text-transform:uppercase}
  .about-desc{margin:.6rem 0 0;max-width:48ch;font-size:clamp(.88rem,1.3vw,1.02rem);line-height:1.6;opacity:.75}
  .about-desc strong{opacity:1;color:var(--red)}
  .about-stats{display:flex;gap:1.5rem;flex-wrap:wrap}
  .stat{display:flex;flex-direction:column;align-items:center;gap:.15rem;padding:1.2rem 1.5rem;border-radius:1rem;background:rgba(126,15,20,.06);flex:1;min-width:120px;text-align:center;animation:fu .6s backwards;animation-delay:calc(var(--si)*.12s + .3s)}
  .stat-icon{font-size:1.6rem}.stat-val{font-family:'Anton',sans-serif;font-size:clamp(1.8rem,4vw,2.8rem);line-height:1}.stat-label{font-size:.72rem;font-weight:700;opacity:.5;text-transform:uppercase;letter-spacing:.06em}

  .mission-row{display:grid;grid-template-columns:repeat(auto-fit,minmax(min(100%,240px),1fr));gap:1rem}
  .mission-card{padding:1.5rem;border-radius:1rem;background:rgba(126,15,20,.05);border:1px solid rgba(126,15,20,.08);transition:transform .3s,box-shadow .3s}
  .mc-icon{font-size:1.5rem;display:block;margin-bottom:.4rem}
  .mission-card h4{margin:0 0 .3rem;font-size:1.05rem}
  .mission-card p{margin:0;font-size:.82rem;opacity:.65;line-height:1.5}
  @media(hover:hover){.mission-card:hover{transform:translateY(-3px);box-shadow:0 8px 24px rgba(0,0,0,.06)}}

  /* ===== PROJ ===== */
  .proj{position:relative;z-index:1;background:var(--cream);color:var(--maroon);padding:2rem 0 4rem;overflow:visible;scroll-margin-top:0}
  .proj-head{margin-bottom:1.5rem}.proj-head h2{margin:0;font-size:clamp(2.6rem,9vw,6.5rem);line-height:.88;text-transform:uppercase}
  .proj-sub{margin:.4rem 0 0;font-size:clamp(.85rem,1.3vw,1rem);font-weight:600;opacity:.45}

  /* ===== CLOTHESLINE ===== */
  .clothesline{position:relative;padding:1rem 0 0}
  .rope{display:block;width:100%;height:70px;margin-bottom:-30px;position:relative;z-index:1}

  /* 3D CAROUSEL */
  .carousel{position:relative;z-index:2;display:flex;gap:clamp(1rem,3vw,2.5rem);justify-content:center;align-items:flex-start;padding:0 var(--pad);perspective:1000px}

  .ccard{
    all:unset;cursor:pointer;position:relative;display:flex;flex-direction:column;align-items:center;
    width:clamp(220px,38vw,360px);
    transform-origin:top center;
    transition:all .7s cubic-bezier(.34,1.56,.64,1);
    animation:sway 5.5s ease-in-out infinite;animation-delay:calc(var(--ci)*.8s);
  }
  @keyframes sway{0%,100%{transform:rotate(var(--rot)) translateY(0)}50%{transform:rotate(calc(var(--rot)*-.5)) translateY(2px)}}

  /* ACTIVE card — center, expanded */
  .ccard.active{animation:none;transform:rotate(0) translateY(-22px) scale(1.05);z-index:5}
  /* LEFT of active — pushed back left */
  .ccard.left{animation:none;transform:rotate(8deg) translateX(-20px) translateY(10px) scale(.88);opacity:.35;filter:blur(.8px)}
  /* RIGHT of active — pushed back right */
  .ccard.right{animation:none;transform:rotate(-8deg) translateX(20px) translateY(10px) scale(.88);opacity:.35;filter:blur(.8px)}

  .cpin{position:relative;z-index:3;width:18px;height:34px;margin-bottom:-5px;transition:transform .5s cubic-bezier(.34,1.56,.64,1)}
  .pin-wood{position:relative;width:100%;height:100%}
  .pin-top{position:absolute;top:0;left:2px;right:2px;height:13px;background:linear-gradient(135deg,#d4a76a,#c49154);border-radius:3px 3px 1px 1px;box-shadow:0 1px 2px rgba(0,0,0,.12)}
  .pin-coil{position:absolute;top:11px;left:4px;right:4px;height:5px;background:linear-gradient(90deg,#999,#bbb,#999);border-radius:2px}
  .pin-l,.pin-r{position:absolute;top:14px;width:6px;height:20px;background:linear-gradient(135deg,#d4a76a,#b8824a);border-radius:0 0 2px 2px}
  .pin-l{left:1px}.pin-r{right:1px}
  .ccard.active .cpin{transform:translateY(-3px) scale(1.06)}

  .cstring{width:1px;height:10px;background:rgba(126,15,20,.25);margin-bottom:-1px}

  .cbody{position:relative;width:100%;border-radius:1.15rem;overflow:hidden;transition:all .6s cubic-bezier(.34,1.56,.64,1);box-shadow:0 5px 20px rgba(0,0,0,.06)}
  .ccard.active .cbody{box-shadow:0 16px 48px rgba(0,0,0,.14),0 4px 12px rgba(0,0,0,.06);border-radius:1.3rem}
  .cbody-glow{position:absolute;inset:0;background:radial-gradient(circle at var(--gx,30%) var(--gy,20%),rgba(255,255,255,.24),transparent 60%);pointer-events:none;z-index:0}

  .cemoji{position:relative;z-index:1;font-size:2.5rem;line-height:1;padding-top:1.2rem;text-align:center;display:block;filter:drop-shadow(0 2px 4px rgba(0,0,0,.1));transition:transform .5s cubic-bezier(.34,1.56,.64,1)}
  .ccard.active .cemoji{transform:scale(1.15) translateY(-2px)}
  .cnum{position:relative;z-index:1;display:block;text-align:center;font-size:.6rem;font-weight:800;letter-spacing:.1em;opacity:.3;margin-top:.3rem}
  .ckind{position:relative;z-index:1;display:block;text-align:center;font-size:.72rem;font-weight:700;opacity:.6;text-transform:uppercase;letter-spacing:.05em}
  .cname{position:relative;z-index:1;margin:0;text-align:center;font-size:clamp(1.8rem,6vw,2.8rem);line-height:.92;text-transform:uppercase;padding:.1rem .8rem}

  .cart{position:relative;z-index:1;width:70%;margin:.5rem auto 0;max-height:0;overflow:hidden;opacity:0;transition:all .6s cubic-bezier(.16,1,.3,1)}
  .ccard.active .cart{max-height:300px;opacity:1}
  .cstage{width:100%;aspect-ratio:360/300;perspective:700px;position:relative}
  .cdesc{position:relative;z-index:1;margin:0;padding:0 clamp(.6rem,2vw,1.2rem);font-size:.8rem;line-height:1.4;opacity:0;max-height:0;overflow:hidden;transition:all .5s cubic-bezier(.16,1,.3,1);text-align:left}
  .ccard.active .cdesc{opacity:.75;max-height:120px;margin-top:.4rem}
  .ccta{position:relative;z-index:1;display:block;text-align:center;font-size:.72rem;font-weight:700;text-transform:uppercase;letter-spacing:.04em;opacity:0;padding:.6rem .5rem .8rem;transition:opacity .4s}
  .ccard.active .ccta{opacity:.8}

  @media(hover:hover){
    .ccard.left:hover,.ccard.right:hover{opacity:.6;filter:blur(0);transform:scale(.92) translateY(5px)}
    .ccard.active:hover .ccta{opacity:1;text-decoration:underline}
  }

  .slide-nav{display:flex;align-items:center;justify-content:center;gap:1rem;margin-top:1.8rem}
  .arrow{all:unset;cursor:pointer;display:grid;place-items:center;width:40px;height:40px;border-radius:50%;background:rgba(126,15,20,.08);color:var(--maroon);transition:all .3s}
  @media(hover:hover){.arrow:hover{background:var(--maroon);color:var(--cream);transform:scale(1.08)}}
  .dots{display:flex;gap:.4rem}
  .dot{all:unset;cursor:pointer;position:relative;width:42px;height:4px;border-radius:4px;background:rgba(126,15,20,.1);overflow:hidden}
  .dot-fill{position:absolute;inset:0;background:var(--maroon);border-radius:inherit;transform:scaleX(0);transform-origin:left}
  .dot-fill.run{animation:df 4.5s linear forwards}
  @keyframes df{to{transform:scaleX(1)}}
  .dot-label{position:absolute;bottom:8px;left:50%;transform:translateX(-50%);font-size:.6rem;font-weight:700;text-transform:uppercase;letter-spacing:.04em;white-space:nowrap;opacity:0;transition:opacity .3s;color:var(--maroon)}
  .dot:hover .dot-label,.dot.on .dot-label{opacity:1}

  /* DETAIL */
  .det-wrap{scroll-margin-top:1rem;margin-top:1.5rem}
  .det{border-radius:1.5rem;overflow:hidden;transform-origin:50% 0}
  .det-top{padding:.9rem clamp(1rem,3vw,2rem);border-bottom:1px solid rgba(255,255,255,.06)}
  .back{all:unset;cursor:pointer;display:inline-flex;align-items:center;gap:.3rem;font:inherit;font-weight:700;font-size:.78rem;padding:.35rem .8rem;border-radius:99px;background:rgba(255,255,255,.08);transition:background .3s}
  @media(hover:hover){.back:hover{background:rgba(255,255,255,.15)}}
  .det-body{display:grid;grid-template-columns:1fr;gap:1.5rem;padding:clamp(1rem,3vw,2rem);align-items:center}
  .det-copy{display:flex;flex-direction:column;gap:.7rem}
  .det-num{font-size:.6rem;font-weight:800;letter-spacing:.1em;opacity:.25}.det-kind{margin:0;font-size:.85rem;letter-spacing:.04em;opacity:.65}
  .det h3{margin:0;font-size:clamp(2.2rem,7vw,5rem);line-height:.9;text-transform:uppercase}
  .det-desc{margin:0;max-width:40ch;font-size:clamp(.88rem,1.3vw,1.05rem);font-weight:600}
  ul{margin:0;padding:0 0 0 1rem;max-width:42ch;display:grid;gap:.3rem}
  li{animation:fli .5s cubic-bezier(.16,1,.3,1) both;animation-delay:calc(var(--ld)*100ms + 300ms)}
  @keyframes fli{from{opacity:0;transform:translateX(-8px)}to{opacity:1;transform:none}}
  .det-btns{display:flex;flex-wrap:wrap;gap:.55rem}
  .det-art{display:grid;place-items:center}

  .stage{position:relative;width:min(100%,420px);aspect-ratio:360/300;perspective:1100px}
  .scene{position:absolute;inset:0;transform-style:preserve-3d;transform:rotateX(var(--base,0));will-change:transform}
  .layer{position:absolute;inset:0;width:100%;height:100%;overflow:visible;transform:translateZ(var(--z,0))}
  .layer.lift{filter:drop-shadow(0 10px 10px rgba(0,0,0,.18))}

  .closing{position:relative;z-index:1;background:var(--red);padding:4rem 0;overflow:hidden}
  .closing::before{content:'';position:absolute;inset:0;background-image:url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='160' height='160'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'/><feColorMatrix values='0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .5 0'/></filter><rect width='100%' height='100%' filter='url(%23n)'/></svg>");opacity:.12;mix-blend-mode:multiply;pointer-events:none}
  .closing .wrap{position:relative;display:flex;flex-direction:column;gap:.85rem}
  .closing h2{margin:0;text-transform:uppercase;font-size:clamp(1.8rem,6vw,4.5rem);line-height:.93;max-width:14ch}
  .closing p{margin:0;font-size:.95rem;opacity:.7}
  footer{background:var(--maroon);display:flex;flex-wrap:wrap;justify-content:space-between;align-items:center;gap:.4rem;padding:1rem var(--pad);font-size:.72rem;opacity:.7}
  .fig{display:inline-flex;align-items:center;gap:.25rem;transition:opacity .3s}@media(hover:hover){.fig:hover{opacity:.5}}

  .totop{position:fixed;right:max(1rem,env(safe-area-inset-right));bottom:max(1rem,env(safe-area-inset-bottom));z-index:20;display:grid;place-items:center;width:44px;height:44px;border-radius:50%;background:var(--maroon);color:var(--cream);box-shadow:0 6px 20px rgba(0,0,0,.25);opacity:0;transform:translateY(14px) scale(.8);pointer-events:none;transition:opacity .35s,transform .35s cubic-bezier(.34,1.56,.64,1)}
  .totop.show{opacity:1;transform:none;pointer-events:auto}
  @media(hover:hover){.totop.show:hover{transform:translateY(-3px)}}
  .stat{transition:transform .3s}@media(hover:hover){.stat:hover{transform:translateY(-4px) rotate(-1deg)}}
  .nav{padding-top:max(.75rem,env(safe-area-inset-top))}
  footer{padding-bottom:max(1rem,env(safe-area-inset-bottom))}

  .ico{width:1em;height:1em;stroke-width:1.7;overflow:visible}
  .cemoji .ico,.stat-icon .ico{margin-inline:auto}.mc-icon{color:var(--red)}
  h1 span:nth-child(odd){translate:calc(var(--p,0)*-9vw) 0}
  h1 span:nth-child(even){translate:calc(var(--p,0)*9vw) 0}
  .rope{transform-origin:50% 0;transform:scaleY(calc(1 + var(--v,0)*.025))}
  @media(min-width:600px){.carousel{transform-origin:50% 0;transform:rotate(calc(var(--v,0)*.3deg))}}

  /* ===== FX ===== */
  .hero-bg::after{content:'';position:absolute;inset:0;background:radial-gradient(520px circle at var(--mx,50%) var(--my,40%),rgba(255,214,170,.28),transparent 65%);pointer-events:none}
  .topo{position:absolute;inset:0;width:100%;height:100%;fill:none;stroke:var(--cream);stroke-width:1.2;opacity:.2}
  .contour{transform-box:fill-box;transform-origin:center;animation:cspin 90s linear infinite}.contour.c2{animation-duration:120s;animation-direction:reverse}
  @keyframes cspin{to{transform:rotate(360deg)}}
  h1 span:last-child{background:linear-gradient(100deg,var(--cream) 35%,#ffc98a 50%,var(--cream) 65%) 0 0/250% 100%;-webkit-background-clip:text;background-clip:text;color:transparent;animation:fp 1s cubic-bezier(.2,.8,.2,1) both,shine 5s linear 1.6s infinite;animation-delay:calc(var(--d)*130ms),1.6s}
  @keyframes shine{from{background-position:120% 0}to{background-position:-120% 0}}

  .btn{position:relative;overflow:hidden}
  .btn::after{content:'';position:absolute;inset:0;background:linear-gradient(110deg,transparent 35%,rgba(255,255,255,.4) 50%,transparent 65%);transform:translateX(-130%);pointer-events:none}
  @media(hover:hover){.btn:hover::after{transform:translateX(130%);transition:transform .8s cubic-bezier(.16,1,.3,1)}}

  .ccard.active .cbody{transform:perspective(900px) rotateX(var(--rx,0deg)) rotateY(var(--ry,0deg));transition:transform .18s ease-out,box-shadow .6s,border-radius .6s}

  .fx{animation:fb var(--fs,4s) ease-in-out var(--fd,0s) infinite}
  @keyframes fb{50%{translate:0 -8px}}
  .route{animation:flow 1.4s linear infinite}
  @keyframes flow{to{stroke-dashoffset:-24}}
  .wave{animation:wave 5s ease-in-out infinite alternate}
  @keyframes wave{from{transform:translateX(-7px)}to{transform:translateX(7px)}}
  .ring{transform-box:fill-box;transform-origin:center;animation:ring 2.4s ease-out infinite}
  @keyframes ring{0%{transform:scale(.3);opacity:.9}100%{transform:scale(3);opacity:0}}
  @media(prefers-reduced-motion:reduce){.fx,.route,.wave,.ring,.contour,h1 span:last-child{animation:none}.btn::after{display:none}.ccard.active .cbody{transition:none}}

  /* RESPONSIVE */
  @media(min-width:768px){.about-grid{grid-template-columns:1.2fr 1fr;align-items:center}}
  @media(min-width:900px){.det-body{grid-template-columns:1.1fr 1fr}}
  @media(min-width:900px) and (min-height:600px){.hero{position:sticky;top:0;z-index:0}.about{border-radius:2rem 2rem 0 0;box-shadow:0 -8px 30px rgba(0,0,0,.06)}}
  @media(min-width:1000px){.ccard{width:clamp(260px,28vw,400px)}}

  @media(max-width:599px){
    .nav{gap:.2rem}.brand-text{font-size:.75rem}.brand-logo{width:24px;height:24px}
    .hero{padding:4rem var(--pad) 2rem}h1{font-size:clamp(2rem,12vw,3.2rem)}
    .hero-btns{flex-direction:column;width:100%;align-items:stretch}.hero-btns .btn{justify-content:center}
    .scroll-h{display:none}
    .about{padding:3rem 0 2rem}.about-text h2{font-size:clamp(2.2rem,11vw,3.2rem)}.about-stats{gap:.8rem}
    .stat{padding:.9rem 1rem;min-width:90px}.stat-val{font-size:clamp(1.4rem,5vw,2rem)}
    .proj{padding:1.5rem 0 3rem}.proj-head h2{font-size:clamp(2rem,10vw,3rem)}
    .carousel{flex-direction:column;align-items:center;gap:1rem;perspective:none}
    .ccard{width:min(100%,340px);animation:none!important;transform:rotate(0)!important}
    .ccard.active{transform:translateY(-8px) scale(1.02)!important}
    .ccard.left,.ccard.right{transform:scale(.95)!important;opacity:.4;filter:blur(.3px)}
    .rope{display:none}.cpin{display:none}.cstring{display:none}
    .cbody{border-radius:.85rem}
    .det-body{padding:clamp(.7rem,3vw,1rem)}.det h3{font-size:clamp(1.7rem,9vw,2.5rem)}.det-btns .btn{flex:1 1 100%}
    .closing{padding:2.5rem 0}.closing h2{font-size:clamp(1.5rem,7vw,2.2rem)}
    footer{flex-direction:column;align-items:center;text-align:center}
    .arrow{width:34px;height:34px}
  }
  @media(max-width:360px){:global(:root){--pad:.55rem}h1{font-size:clamp(1.7rem,13vw,2.4rem)}.brand-text{font-size:.68rem}.cname{font-size:clamp(1.4rem,8vw,1.8rem)}}
  @media(max-width:899px){.det-art{order:-1}.stage{width:min(100%,340px)}}
  @media(max-width:599px){
    .about-stats{display:grid;grid-template-columns:repeat(3,1fr);gap:.6rem}
    .stat{min-width:0;padding:.8rem .3rem}.stat-icon{font-size:1.3rem}.stat-label{font-size:.58rem;letter-spacing:.03em}
    .arrow{width:42px;height:42px}.dots{gap:.3rem}.dot{width:34px}
    .mission-card{padding:1.1rem}
  }
  @media(max-width:380px){.hide-xs{display:none}}
  @media(min-width:1600px){.wrap{max-width:1600px}.ccard{width:440px}h1{font-size:8.5rem}.about-desc{font-size:1.1rem}}
  @media(max-height:520px) and (orientation:landscape){.hero{min-height:auto;padding-block:3rem 1.5rem}.hero-bg{display:none}}
  @media(prefers-reduced-motion:reduce){.totop,.stat{transition:none}:global(html){scroll-behavior:auto}h1 span{animation:none}.band-t{animation:none}.btn,.ccard,.cbody,.cart,.cdesc,.ccta,.cpin,.cemoji,.stat{transition:none}.particle,.orb{animation:none}.scroll-h{display:none}.ccard{animation:none!important}}

  /* ===== BADGE HARGA ===== */
  .badge{position:absolute;top:.75rem;right:.75rem;z-index:2;display:inline-flex;align-items:center;gap:.3rem;padding:.25rem .65rem;border-radius:99px;font-size:.64rem;font-weight:800;letter-spacing:.06em;text-transform:uppercase;background:var(--cream);color:var(--maroon);box-shadow:0 2px 8px rgba(0,0,0,.15)}
  .badge.paid{background:linear-gradient(100deg,#ffd36b 35%,#fff4c9 50%,#ffd36b 65%) 0 0/250% 100%;color:#4a2c00;animation:shine 3s linear infinite}
  .badge.inl{position:static;align-self:flex-start}
  .badge .ico{width:.95em;height:.95em}

  /* ===== TIM ===== */
  .team{position:relative;z-index:1;background:var(--cream);color:var(--maroon);padding:4.5rem 0 4rem}
  .team-head{margin-bottom:2rem}.team-head h2{margin:0;font-size:clamp(2.6rem,9vw,6.5rem);line-height:.88;text-transform:uppercase}
  .team-grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(min(100%,250px),1fr));gap:1.2rem}
  @media(min-width:900px){.tslot:nth-child(2){margin-top:2rem}}
  .tcard{position:relative;overflow:hidden;text-align:center;padding:2rem 1.2rem 1.6rem;border-radius:1.3rem;background:var(--maroon);color:var(--cream);box-shadow:0 10px 30px rgba(126,15,20,.18);transform:perspective(900px) rotateX(var(--rx,0deg)) rotateY(var(--ry,0deg));transition:transform .18s ease-out,box-shadow .4s}
  .tcard::before{content:'';position:absolute;inset:0;background:radial-gradient(circle at var(--gx,30%) var(--gy,20%),rgba(255,255,255,.2),transparent 60%);pointer-events:none}
  .tcard>*{position:relative;z-index:1}
  .avatar{position:relative;width:104px;aspect-ratio:1;margin:0 auto 1rem;display:grid;place-items:center;animation:fb 4s ease-in-out infinite;animation-delay:calc(var(--ti)*-.9s)}
  .a-ring{position:absolute;inset:-7px;border-radius:50%;background:conic-gradient(var(--h),var(--cream),var(--h));animation:cspin 6s linear infinite;-webkit-mask:radial-gradient(farthest-side,transparent calc(100% - 4px),#000 calc(100% - 3px));mask:radial-gradient(farthest-side,transparent calc(100% - 4px),#000 calc(100% - 3px))}
  .a-init{width:100%;height:100%;border-radius:50%;display:grid;place-items:center;font-family:'Anton',sans-serif;font-size:2.2rem;background:radial-gradient(circle at 30% 25%,var(--h),#3a0508 90%);color:var(--cream);text-shadow:0 2px 6px rgba(0,0,0,.35)}
  .t-num{display:block;font-size:.6rem;font-weight:800;letter-spacing:.1em;opacity:.35}
  .tcard h3{margin:.2rem 0 0;font-size:1.7rem;line-height:1;text-transform:uppercase}
  .t-role{margin:.5rem 0 0;font-size:.78rem;font-weight:700;opacity:.65;text-transform:uppercase;letter-spacing:.05em}
  @media(hover:hover){.tcard:hover{box-shadow:0 18px 44px rgba(126,15,20,.3)}}

  @media(max-width:599px){.team{padding:3rem 0 2.5rem}}
  @media(prefers-reduced-motion:reduce){.a-ring,.avatar,.badge.paid,.cstep{animation:none}.tcard{transition:none}}

  /* kredit di detail proyek */
  .cred{margin-top:.4rem;padding:.9rem 1rem;border-radius:1rem;background:rgba(6,54,31,.1);max-width:46ch}
  .cred-t{margin:0 0 .5rem;font-size:.68rem;font-weight:800;letter-spacing:.1em;text-transform:uppercase;opacity:.7}
  .csteps{display:grid;gap:.5rem}
  .cstep{display:flex;align-items:center;gap:.65rem;animation:fli .5s cubic-bezier(.16,1,.3,1) both;animation-delay:calc(var(--ld)*100ms + 600ms)}
  .cstep-ico{display:grid;place-items:center;flex:none;width:2rem;height:2rem;border-radius:50%;background:linear-gradient(135deg,#ffd36b,#e8a317);color:#4a2c00;font-size:1rem;animation:cpop 2.8s ease-in-out infinite;animation-delay:calc(var(--ld)*.35s)}
  @keyframes cpop{0%,70%,100%{transform:none}80%{transform:scale(1.18) rotate(-8deg)}}
  .cstep-txt{display:flex;flex-direction:column;line-height:1.3}.cstep-txt strong{font-size:.85rem}.cstep-txt span{font-size:.74rem;opacity:.7}
</style>
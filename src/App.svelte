<script lang="ts">
  import { onMount, tick as svelteTick } from 'svelte';
  import { cubicOut, cubicInOut } from 'svelte/easing';
  import { fly } from 'svelte/transition';

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

  // === TEAM MEMBER DATA ===
  type TeamMember = {
    name: string;
    role: string;
    photo: string;
    hue: string;
    tagline: string;
    about: string;
    skills: string[];
    contribution: string;
    socials: { platform: string; url: string; label: string }[];
  };

  const team: TeamMember[] = [
    {
      name: 'Ricardo Gellael',
      role: 'Member',
      photo: '/team/ricardo-gellael.jpeg',
      hue: '#ff6b35',
      tagline: 'Kalibata Project team member.',
      about: 'Bagian dari tim Kalibata Project yang membangun produk digital dari ide menjadi kenyataan.',
      skills: ['Development', 'Problem Solving', 'Collaboration'],
      contribution: 'Berperan aktif dalam pengembangan dan pertumbuhan produk-produk Kalibata Project.',
      socials: [
        { platform: 'instagram', url: 'https://www.instagram.com/ricardo_gellael', label: 'Ricardo Gellael' }
      ],
    },
    {
      name: 'Rayhan MA',
      role: 'Member',
      photo: '/team/rayhan-ma.jpeg',
      hue: '#1fbf6b',
      tagline: 'Kalibata Project team member.',
      about: 'Bagian dari tim Kalibata Project yang membangun produk digital dari ide menjadi kenyataan.',
      skills: ['Development', 'Problem Solving', 'Collaboration'],
      contribution: 'Berperan aktif dalam pengembangan dan pertumbuhan produk-produk Kalibata Project.',
      socials: [
        { platform: 'instagram', url: 'https://www.instagram.com/rayhanma__', label: 'Rayhan MA' }
      ],
    },
    {
      name: 'Juan AP',
      role: 'Member',
      photo: '/team/juan-ap.jpg',
      hue: '#ffd36b',
      tagline: 'Kalibata Project team member.',
      about: 'Bagian dari tim Kalibata Project yang membangun produk digital dari ide menjadi kenyataan.',
      skills: ['Development', 'Problem Solving', 'Collaboration'],
      contribution: 'Berperan aktif dalam pengembangan dan pertumbuhan produk-produk Kalibata Project.',
      socials: [
        { platform: 'instagram', url: 'https://www.instagram.com/juan_agraprana', label: 'Juan AP' }
      ],
    },
  ];

  const steps = [
    { t:'Top-up lewat admin', d:'Hubungi admin dan bayar secara manual.', icon:'send' },
    { t:'Kirim bukti bayar', d:'Kirim screenshot bukti pembayaran ke admin.', icon:'check' },
    { t:'Kredit masuk akun', d:'Admin menambahkan kredit ke akun PetaKilat-mu.', icon:'coin' },
    { t:'Pakai per peta', d:'Setiap peta baru memakai kredit dari saldomu.', icon:'map' },
  ];

  const bandText = projects.map(p => p.name).join(' ✦ ') + ' ✦ Where ideas become projects ✦ ';

  // === DITHER SHADER (Bayer 8x8, duotone) ===
  type DitherOpts = { hue: string; photo?: string; cell?: number };

  const BAYER = (() => {
    let m: number[][] = [[0]];
    for (let n = 1; n < 8; n *= 2) {
      const s = m.length;
      const nx: number[][] = Array.from({ length: s * 2 }, () => Array(s * 2).fill(0));
      for (let y = 0; y < s; y++) for (let x = 0; x < s; x++) {
        const v = m[y][x] * 4;
        nx[y][x] = v; nx[y][x + s] = v + 2; nx[y + s][x] = v + 3; nx[y + s][x + s] = v + 1;
      }
      m = nx;
    }
    return m.flat().map(v => (v + .5) / 64);
  })();

  const hex2rgb = (h: string): number[] => { const n = parseInt(h.slice(1), 16); return [(n >> 16) & 255, (n >> 8) & 255, n & 255]; };
  const smooth = (a: number, b: number, x: number) => { const t = Math.min(1, Math.max(0, (x - a) / (b - a))); return t * t * (3 - 2 * t); };
  const DARK = hex2rgb('#2a0306');

  // blur kotak sederhana (dua arah) untuk kontras lokal pada foto
  function boxBlur(a: Float32Array, W: number, H: number, r: number) {
    const t = new Float32Array(a.length), o = new Float32Array(a.length), k = 2 * r + 1;
    const cl = (v: number, m: number) => Math.min(m, Math.max(0, v));
    for (let y = 0; y < H; y++) {
      let acc = 0;
      for (let x = -r; x <= r; x++) acc += a[y * W + cl(x, W - 1)];
      for (let x = 0; x < W; x++) { t[y * W + x] = acc / k; acc += a[y * W + cl(x + r + 1, W - 1)] - a[y * W + cl(x - r, W - 1)]; }
    }
    for (let x = 0; x < W; x++) {
      let acc = 0;
      for (let y = -r; y <= r; y++) acc += t[cl(y, H - 1) * W + x];
      for (let y = 0; y < H; y++) { o[y * W + x] = acc / k; acc += t[cl(y + r + 1, H - 1) * W + x] - t[cl(y - r, H - 1) * W + x]; }
    }
    return o;
  }

  // Action: gambar siluet potret ber-dither. Kalau ada foto, foto itu yang di-dither.
  function dither(node: HTMLElement, opts: DitherOpts) {
    const canvas = document.createElement('canvas');
    node.appendChild(canvas);
    const ctx = canvas.getContext('2d', { alpha: false })!;
    let o = opts, w = 8, h = 8, img = ctx.createImageData(8, 8);
    let src: Float32Array | null = null;
    let cl = hex2rgb(o.hue), to = cl;
    const reduce = noMo();
    let pulse = reduce ? 0 : 1, px = .5, py = .38, tx = .5, ty = .38;
    let visible = true, raf = 0, last = 0;

    const loadPhoto = () => {
      src = null;
      if (!o.photo) return;
      const im = new Image(); im.crossOrigin = 'anonymous';
      im.onload = () => {
        try {
          const oc = document.createElement('canvas'); oc.width = w; oc.height = h;
          const c2 = oc.getContext('2d')!;
          c2.imageSmoothingQuality = 'high';
          const k = Math.max(w / im.width, h / im.height), dw = im.width * k, dh = im.height * k;
          c2.drawImage(im, (w - dw) / 2, (h - dh) / 2, dw, dh);
          const d = c2.getImageData(0, 0, w, h).data, a = new Float32Array(w * h);
          for (let i = 0; i < a.length; i++) a[i] = (d[i * 4] * .299 + d[i * 4 + 1] * .587 + d[i * 4 + 2] * .114) / 255;
          // auto-levels (persentil 3–97), angkat midtone, lalu kontras lokal agar wajah tetap terbaca
          const sorted = Float32Array.from(a).sort();
          const lo = sorted[Math.floor(a.length * .03)], hi = sorted[Math.floor(a.length * .97)];
          for (let i = 0; i < a.length; i++) a[i] = Math.pow(Math.min(1, Math.max(0, (a[i] - lo) / Math.max(hi - lo, .05))), .8);
          const bl = boxBlur(a, w, h, Math.max(3, Math.round(w * .07)));
          for (let i = 0; i < a.length; i++) a[i] = Math.min(1, Math.max(0, a[i] + .9 * (a[i] - bl[i])));
          src = a;
        } catch { src = null; }
        if (reduce) draw(performance.now());
      };
      im.src = o.photo;
    };

    const size = () => {
      const c = o.cell ?? 4;
      w = Math.max(8, Math.round(node.clientWidth / c));
      h = Math.max(8, Math.round(node.clientHeight / c));
      canvas.width = w; canvas.height = h;
      img = ctx.createImageData(w, h);
      loadPhoto();
      draw(performance.now());
    };

    function draw(now: number) {
      const t = now / 1000, a = w / h, d = img.data;
      pulse *= .94;
      px += (tx - px) * .12; py += (ty - py) * .12;
      for (let i = 0; i < 3; i++) cl[i] += (to[i] - cl[i]) * .12;
      const gx = (px - .5) * a;
      for (let y = 0; y < h; y++) {
        const Y = y / h;
        for (let x = 0; x < w; x++) {
          const X = (x / w - .5) * a;
          let L: number;
          if (src) { const sw = (t * .22) % 1.5 - .25; L = (src[y * w + x] ?? 0) + .16 * Math.exp(-Math.pow((Y - sw) * 9, 2)); }
          else {
            const head = Math.hypot(X, Y - .42) - .17;
            const body = (Math.hypot(X / .42, (Y - 1.05) / .36) - 1) * .36;
            const m = 1 - smooth(-.012, .02, Math.min(head, body));
            const pl = .5 + .5 * Math.sin(X * 7 + t * .7 + Math.sin(Y * 5 - t * .5) * 1.4) * Math.cos(Y * 6 - t * .4 + X * 2);
            L = m * ((.62 - Y * .22) + .18 * pl) + (1 - m) * (.08 + .3 * pl * pl);
          }
          const gd = (X - gx) * (X - gx) + (Y - py) * (Y - py);
          L += .55 * Math.exp(-gd * 26);
          const B = BAYER[((y & 7) << 3) | (x & 7)];
          const wipe = pulse > .01 ? pulse * (.3 + .7 * BAYER[((y & 7) << 3) | ((x * 5 + y * 3) & 7)]) : 0;
          const c = L + (B - .5) * .9 - wipe > .5 ? cl : DARK;
          const p = (y * w + x) * 4;
          d[p] = c[0]; d[p + 1] = c[1]; d[p + 2] = c[2]; d[p + 3] = 255;
        }
      }
      ctx.putImageData(img, 0, 0);
    }

    const loop = (now: number) => {
      raf = requestAnimationFrame(loop);
      if (!visible || now - last < 33) return;
      last = now; draw(now);
    };

    const rect = () => node.getBoundingClientRect();
    const mv = (e: PointerEvent) => { const r = rect(); tx = (e.clientX - r.left) / r.width; ty = (e.clientY - r.top) / r.height; if (reduce) draw(performance.now()); };
    const lv = () => { tx = .5; ty = .38; };
    node.addEventListener('pointermove', mv); node.addEventListener('pointerleave', lv);
    const ro = new ResizeObserver(size); ro.observe(node);
    const io = new IntersectionObserver(es => { visible = es[0].isIntersecting; }); io.observe(node);
    size();
    if (!reduce) raf = requestAnimationFrame(loop);

    return {
      update(n: DitherOpts) {
        const hue = n.hue !== o.hue, ph = n.photo !== o.photo, ce = n.cell !== o.cell;
        o = n; to = hex2rgb(o.hue);
        if (hue || ph) pulse = reduce ? 0 : 1;
        if (ce) size(); else if (ph) loadPhoto();
        if (reduce) { cl = to; draw(performance.now()); }
      },
      destroy() {
        cancelAnimationFrame(raf); ro.disconnect(); io.disconnect();
        node.removeEventListener('pointermove', mv); node.removeEventListener('pointerleave', lv);
        canvas.remove();
      }
    };
  }

  // === TEAM: ROSTER + PROFILE SHEET ===
  const STAGE_CELL = 4;
  const PIXELS = [{ n: 'Halus', c: 3 }, { n: 'Sedang', c: 5 }, { n: 'Kasar', c: 8 }];
  const TABS = [{ id: 'about', l: 'Tentang' }, { id: 'skills', l: 'Keahlian' }, { id: 'work', l: 'Kontribusi' }] as const;

  let activeTeam = 0;
  let teamModalOpen = false;
  let teamModalIdx = -1;
  let tab: 'about' | 'skills' | 'work' = 'about';
  let pixIdx = 1;
  let dir = 1;
  let modalX = 0, modalY = 0;
  let tmEl: HTMLDivElement | null = null;
  let closeBtn: HTMLButtonElement | null = null;
  let lastFocus: HTMLElement | null = null;
  $: teamModalMember = teamModalIdx >= 0 ? team[teamModalIdx] : null;
  $: tabIdx = TABS.findIndex(t => t.id === tab);

  function openTeamModal(i: number, e?: MouseEvent) {
    lastFocus = (e?.currentTarget as HTMLElement) ?? null;
    modalX = e && e.clientX ? e.clientX : innerWidth / 2;
    modalY = e && e.clientY ? e.clientY : innerHeight / 2;
    teamModalIdx = i; activeTeam = i; tab = 'about'; dir = 1;
    teamModalOpen = true;
    document.body.style.overflow = 'hidden';
    svelteTick().then(() => closeBtn?.focus());
  }

  function closeTeamModal() {
    teamModalOpen = false;
    document.body.style.overflow = '';
    lastFocus?.focus({ preventScroll: true });
  }

  function stepMember(d: number) {
    dir = d; tab = 'about';
    teamModalIdx = (teamModalIdx + d + team.length) % team.length;
    activeTeam = teamModalIdx;
  }

  function onModalKeydown(e: KeyboardEvent) {
    if (!teamModalOpen) return;
    if (e.key === 'Escape') { closeTeamModal(); return; }
    if (e.key === 'ArrowRight') { stepMember(1); return; }
    if (e.key === 'ArrowLeft') { stepMember(-1); return; }
    if (e.key === 'Tab' && tmEl) {
      const f = [...tmEl.querySelectorAll<HTMLElement>('button,a[href]')].filter(el => el.offsetParent !== null);
      if (!f.length) return;
      const first = f[0], lastEl = f[f.length - 1];
      if (e.shiftKey && document.activeElement === first) { e.preventDefault(); lastEl.focus(); }
      else if (!e.shiftKey && document.activeElement === lastEl) { e.preventDefault(); first.focus(); }
    }
  }

  // Lembar profil membuka seperti lingkaran yang melebar dari titik klik
  function reveal(node: Element, { x = innerWidth / 2, y = innerHeight / 2, duration = 650 } = {}) {
    return {
      duration: noMo() ? 0 : duration,
      easing: cubicInOut,
      css: (t: number) => `clip-path:circle(${t * 150}% at ${x}px ${y}px)`
    };
  }

  function stagger(node: Element, { duration = 450, delay = 0 } = {}) {
    return {
      duration: noMo() ? 0 : duration,
      delay: noMo() ? 0 : delay,
      easing: cubicOut,
      css: (t: number) => `opacity:${t};transform:translateY(${(1 - t) * 22}px)`
    };
  }

  // Di layar sentuh tidak ada hover, jadi sorotan berpindah sendiri
  onMount(() => {
    if (matchMedia('(hover:hover)').matches || noMo()) return;
    const id = setInterval(() => { if (!teamModalOpen) activeTeam = (activeTeam + 1) % team.length; }, 3200);
    return () => clearInterval(id);
  });

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

  // === MINIGAME FOOTER: kuda melompati rintangan ===
  function kfGame(node: HTMLElement) {
    const svg = node.querySelector('svg') as SVGSVGElement;
    const jumpG = node.querySelector('.kf-jump') as SVGGElement;
    const layer = node.querySelector('.kf-obs') as SVGGElement;
    const scoreEl = node.querySelector('.kf-score') as HTMLElement;
    const hint = node.querySelector('.kf-hint') as HTMLElement;
    const NS = 'http://www.w3.org/2000/svg';
    const GROUND = 305, HX0 = 546, HX1 = 594, GRAV = 1500, V0 = 520;
    type Ob = { el: SVGGElement; x: number; w: number; h: number };
    type Dash = { el: SVGPathElement; x: number; y: number; f: number };
    let state: 'idle' | 'run' | 'over' = 'idle';
    let obs: Ob[] = [], dashes: Dash[] = [];
    let y = 0, vy = 0, speed = 320, score = 0, best = 0, gap = 0, raf = 0, last = 0, overAt = 0;
    try { best = parseInt(localStorage.getItem('kalibata-kuda-best') || '0', 10) || 0; } catch {}

    const pad = (n: number) => String(Math.floor(n)).padStart(5, '0');
    const put = (el: Element, x: number, yy: number | string) => el.setAttribute('transform', `translate(${x.toFixed(1)} ${yy})`);
    const showHint = (t: string | null) => { hint.hidden = t === null; if (t) hint.textContent = t; };
    const hud = () => { scoreEl.textContent = state === 'idle' && !best ? '' : `Skor ${pad(score)} · Rekor ${pad(best)}`; };
    // rentang x (satuan viewBox) yang benar-benar terlihat, karena svg memakai "slice"
    const view = () => {
      const r = svg.getBoundingClientRect();
      const vw = r.width / Math.max(r.width / 1440, r.height / 360);
      return { x0: (1440 - vw) / 2, x1: (1440 + vw) / 2 };
    };

    const mkDash = (x: number): Dash => {
      const yy = 246 + Math.random() * 100;
      const el = document.createElementNS(NS, 'path') as SVGPathElement;
      el.setAttribute('d', `M0 0h${(20 + Math.random() * 50).toFixed(0)}`);
      el.setAttribute('stroke', '#7e0f14'); el.setAttribute('stroke-width', '1.6');
      el.setAttribute('stroke-linecap', 'round'); el.setAttribute('opacity', '.2');
      put(el, x, yy.toFixed(1));
      layer.appendChild(el);
      return { el, x, y: yy, f: 0.6 + ((yy - 236) / 124) * 0.6 };
    };

    const spawn = (x: number) => {
      const g = document.createElementNS(NS, 'g') as SVGGElement;
      const r = Math.random();
      let w = 24, h = 16;
      if (r < 0.4) {
        g.innerHTML = '<path d="M-13 0L-10-9-3-15 6-13 13-6 15 0Z" fill="#7e0f14" stroke="#7e0f14" stroke-width="2" stroke-linejoin="round"/>';
      } else {
        const n = r < 0.8 ? 1 : 2;
        w = n === 1 ? 22 : 46; h = 26;
        for (let i = 0; i < n; i++) {
          const u = document.createElementNS(NS, 'use');
          u.setAttribute('href', '#kf-tuft');
          u.setAttribute('transform', `translate(${n === 1 ? 0 : i ? 13 : -13} 0) scale(1.15)`);
          g.appendChild(u);
        }
      }
      put(g, x, GROUND);
      layer.appendChild(g);
      obs.push({ el: g, x, w, h });
    };

    const clearObs = () => { obs.forEach(o => o.el.remove()); obs = []; };
    const resetHorse = () => { jumpG.style.translate = '0 0'; jumpG.style.rotate = ''; node.classList.remove('kf-run', 'kf-over', 'kf-air'); };

    const reset = () => {
      cancelAnimationFrame(raf); state = 'idle'; y = vy = 0;
      clearObs(); resetHorse(); showHint('Ketuk atau tekan Spasi untuk main'); hud();
    };

    const end = (now: number) => {
      state = 'over'; overAt = now; cancelAnimationFrame(raf);
      if (Math.floor(score) > best) {
        best = Math.floor(score);
        try { localStorage.setItem('kalibata-kuda-best', String(best)); } catch {}
      }
      node.classList.remove('kf-run', 'kf-air'); node.classList.add('kf-over');
      jumpG.style.transformBox = 'fill-box'; jumpG.style.transformOrigin = '50% 100%'; jumpG.style.rotate = '-10deg';
      showHint('Game over · ketuk untuk ulang'); hud();
    };

    const loop = (now: number) => {
      raf = requestAnimationFrame(loop);
      const dt = Math.min((now - last) / 1000, 0.05); last = now;
      const { x0, x1 } = view();
      speed = Math.min(620, speed + 7 * dt);
      score += dt * speed * 0.03;

      if (y > 0 || vy > 0) {
        vy -= GRAV * dt; y += vy * dt;
        if (y <= 0) { y = 0; vy = 0; node.classList.remove('kf-air'); }
      }
      jumpG.style.translate = `0 ${(-y).toFixed(1)}px`;

      gap -= speed * dt;
      if (gap <= 0) { spawn(x1 + 40); gap = speed * (0.95 + Math.random() * 0.9) + 80; }

      for (const o of obs) { o.x -= speed * dt; put(o.el, o.x, GROUND); }
      for (let i = obs.length - 1; i >= 0; i--) if (obs[i].x < x0 - 60) { obs[i].el.remove(); obs.splice(i, 1); }
      for (const d of dashes) {
        d.x -= speed * dt * d.f;
        if (d.x < x0 - 90) d.x = x1 + Math.random() * 80;
        put(d.el, d.x, d.y.toFixed(1));
      }
      hud();

      for (const o of obs) {
        if (o.x + o.w / 2 > HX0 && o.x - o.w / 2 < HX1 && y < o.h - 3) { end(now); return; }
      }
    };

    const start = () => {
      clearObs(); resetHorse();
      y = vy = 0; speed = 320; score = 0; gap = 520; last = performance.now();
      node.classList.add('kf-run'); state = 'run'; showHint(null); hud();
      const { x0, x1 } = view();
      if (!dashes.length) for (let i = 0; i < 14; i++) dashes.push(mkDash(x0 + Math.random() * (x1 - x0)));
      cancelAnimationFrame(raf); raf = requestAnimationFrame(loop);
    };

    const press = () => {
      if (state === 'idle') start();
      else if (state === 'over') { if (performance.now() - overAt > 350) start(); }
      else if (y === 0) { vy = V0; y = 0.01; node.classList.add('kf-air'); }
    };

    const onDown = () => { if (state === 'run') press(); };
    const onClick = () => { node.focus({ preventScroll: true }); if (state !== 'run') press(); };
    const onKey = (e: KeyboardEvent) => {
      if (state === 'run' && (e.code === 'Space' || e.code === 'ArrowUp')) { e.preventDefault(); press(); return; }
      if (document.activeElement === node && (e.code === 'Space' || e.code === 'ArrowUp' || e.code === 'Enter')) { e.preventDefault(); press(); }
    };

    node.addEventListener('pointerdown', onDown);
    node.addEventListener('click', onClick);
    addEventListener('keydown', onKey);
    const io = new IntersectionObserver(es => { if (!es[0].isIntersecting && state === 'run') reset(); });
    io.observe(node);
    showHint('Ketuk atau tekan Spasi untuk main'); hud();

    return {
      destroy() {
        cancelAnimationFrame(raf); io.disconnect();
        node.removeEventListener('pointerdown', onDown); node.removeEventListener('click', onClick);
        removeEventListener('keydown', onKey);
      }
    };
  }
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
    {:else if n==='user'}<circle cx="12" cy="8" r="4"/><path d="M4 21c0-4.4 3.6-7 8-7s8 2.6 8 7"/>
    {:else if n==='send'}<path d="M21 3L10 14M21 3l-7 18-4-7-7-4z"/>
    {:else if n==='check'}<circle cx="12" cy="12" r="9"/><path d="M8 12.5l3 3 5-6"/>
    {:else}<rect x="3" y="9" width="18" height="12" rx="1.5"/><path d="M12 9v12M3 13h18M12 9C9 9 7 8 7 6s3-2 5 3c2-5 5-5 5-3s-2 3-5 3"/>{/if}
  </svg>
{/snippet}

{#snippet socialIcon(platform: string)}
  {#if platform === 'instagram'}
    <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true"><path d="M12 2.163c3.204 0 3.584.012 4.85.07 3.252.148 4.771 1.691 4.919 4.919.058 1.265.069 1.645.069 4.849 0 3.205-.012 3.584-.069 4.849-.149 3.225-1.664 4.771-4.919 4.919-1.266.058-1.644.07-4.85.07-3.204 0-3.584-.012-4.849-.07-3.26-.149-4.771-1.699-4.919-4.92-.058-1.265-.07-1.644-.07-4.849 0-3.204.013-3.583.07-4.849.149-3.227 1.664-4.771 4.919-4.919 1.266-.057 1.645-.069 4.849-.069zM12 0C8.741 0 8.333.014 7.053.072 2.695.272.273 2.69.073 7.052.014 8.333 0 8.741 0 12c0 3.259.014 3.668.072 4.948.2 4.358 2.618 6.78 6.98 6.98C8.333 23.986 8.741 24 12 24c3.259 0 3.668-.014 4.948-.072 4.354-.2 6.782-2.618 6.979-6.98.059-1.28.073-1.689.073-4.948 0-3.259-.014-3.667-.072-4.947-.196-4.354-2.617-6.78-6.979-6.98C15.668.014 15.259 0 12 0zm0 5.838a6.162 6.162 0 100 12.324 6.162 6.162 0 000-12.324zM12 16a4 4 0 110-8 4 4 0 010 8zm6.406-11.845a1.44 1.44 0 100 2.881 1.44 1.44 0 000-2.881z"/></svg>
  {:else if platform === 'github'}
    <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true"><path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"/></svg>
  {:else if platform === 'linkedin'}
    <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true"><path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433a2.062 2.062 0 01-2.063-2.065 2.064 2.064 0 112.063 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/></svg>
  {:else}
    <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><circle cx="12" cy="12" r="10"/><line x1="2" y1="12" x2="22" y2="12"/><path d="M12 2a15.3 15.3 0 014 10 15.3 15.3 0 01-4 10 15.3 15.3 0 01-4-10 15.3 15.3 0 014-10z"/></svg>
  {/if}
{/snippet}

<svelte:window on:keydown={onModalKeydown} />

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
      <h1><span style="--d:0">Where ideas</span><span style="--d:1">become</span><span style="--d:2">projects.</span></h1>
      <p class="hero-sub">Mengubah ide menjadi produk digital yang bisa langsung dipakai.</p>
      <div class="hero-btns">
        <a class="btn glow" use:magnet href="#proyek">Lihat proyek</a>
        <a class="btn ghost hg" href={ig} target="_blank" rel="noopener noreferrer">Hubungi kami</a>
      </div>
    </div>
    <div class="scroll-h" aria-hidden="true"><div class="smouse"><div class="swheel"></div></div></div>
  </section>

  <div class="band-wrap" aria-hidden="true"><div class="band" bind:this={bandEl}><div class="band-t">{#each [0,1] as _}<div class="grp">{#each [0,1,2] as _}<span>{bandText}</span>{/each}</div>{/each}</div></div></div>

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
          <p>UI yang bersih dan intuitif tidak perlu baca manual untuk mulai menggunakan.</p>
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
                    <div class="cstage"><div class="scene" style="--base:{p.tilt*.3}deg">{@render art(p.id)}</div></div>
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
      <div class="roster">
        <div class="stage" use:obs aria-hidden="true">
          <div class="stage-screen" use:dither={{ hue: team[activeTeam].hue, photo: team[activeTeam].photo, cell: STAGE_CELL }}></div>
          <span class="hud hud-tl">Fig. 0{activeTeam + 1}</span>
          <span class="hud hud-tr">Bayer 8×8</span>
          <span class="hud hud-bl">{team[activeTeam].name}</span>
          <span class="hud hud-br">{STAGE_CELL}px</span>
        </div>
        <ol class="rows">
          {#each team as m, i}
            <li use:obs data-delay={i * 120}>
              <button class="row" class:on={activeTeam === i} style="--h:{m.hue}"
                on:pointerenter={() => (activeTeam = i)} on:focus={() => (activeTeam = i)}
                on:click={(e) => openTeamModal(i, e)} aria-label="Lihat profil {m.name}">
                <span class="row-name">{m.name}</span>
                <span class="row-role">{m.role || 'Member'}</span>
                <span class="row-go" aria-hidden="true">
                  <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="M5 12h14M13 6l6 6-6 6"/></svg>
                </span>
              </button>
            </li>
          {/each}
        </ol>
      </div>
    </div>
  </section>

  <!-- ===== PROFIL TIM (LEMBAR PENUH) ===== -->
  {#if teamModalOpen && teamModalMember}
    <div class="tm" role="dialog" aria-modal="true" aria-label="Profil {teamModalMember.name}"
      bind:this={tmEl} style="--h:{teamModalMember.hue}"
      transition:reveal={{ x: modalX, y: modalY }}>
      <div class="tm-visual">
        <div class="tm-screen" use:dither={{ hue: teamModalMember.hue, photo: teamModalMember.photo, cell: PIXELS[pixIdx].c }}></div>
        <button class="tm-pixel" on:click={() => (pixIdx = (pixIdx + 1) % PIXELS.length)} aria-label="Ganti ukuran piksel">
          Piksel · {PIXELS[pixIdx].n}
        </button>
      </div>

      <div class="tm-panel">
        <div class="tm-top">
          <span>Kalibata Project / Tim</span>
          <button class="tm-close" bind:this={closeBtn} on:click={closeTeamModal} aria-label="Tutup profil">
            <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M18 6L6 18M6 6l12 12"/></svg>
          </button>
        </div>

        {#key teamModalIdx}
          <div class="tm-body" in:fly={{ x: dir * 48, duration: 480, easing: cubicOut }}>
            <p class="tm-role">{teamModalMember.role || 'Team Member'}</p>
            <h2 class="tm-name">{teamModalMember.name}</h2>
            <p class="tm-tagline">{teamModalMember.tagline}</p>

            <div class="tm-tabs" role="tablist" style="--i:{tabIdx}">
              {#each TABS as t}
                <button class="tm-tab" role="tab" aria-selected={tab === t.id} on:click={() => (tab = t.id)}>{t.l}</button>
              {/each}
              <span class="tm-ind" aria-hidden="true"></span>
            </div>

            {#key tab}
              <div class="tm-pane" role="tabpanel" in:stagger={{ duration: 380 }}>
                {#if tab === 'about'}
                  <p>{teamModalMember.about}</p>
                {:else if tab === 'skills'}
                  <ul class="tm-skills">
                    {#each teamModalMember.skills as skill, si}
                      <li in:stagger={{ delay: si * 70, duration: 380 }}>{skill}</li>
                    {/each}
                  </ul>
                {:else}
                  <p>{teamModalMember.contribution}</p>
                  <div class="tm-links">
                    {#each projects as p}
                      <a class="tm-link" href={p.url} target="_blank" rel="noopener noreferrer">{p.name} ↗</a>
                    {/each}
                  </div>
                {/if}
              </div>
            {/key}

            <div class="tm-links">
              {#each teamModalMember.socials as s}
                <a class="tm-link" href={s.url} target="_blank" rel="noopener noreferrer" aria-label={s.label}>
                  {@render socialIcon(s.platform)}<span>{s.label}</span>
                </a>
              {:else}
                <a class="tm-link" href={ig} target="_blank" rel="noopener noreferrer">
                  {@render socialIcon('instagram')}<span>@kalibata.project</span>
                </a>
              {/each}
            </div>
          </div>
        {/key}

        <nav class="tm-nav" aria-label="Navigasi anggota tim">
          <button class="tm-step" on:click={() => stepMember(-1)}>
            <small>← Sebelumnya</small><span>{team[(teamModalIdx + team.length - 1) % team.length].name}</span>
          </button>
          <div class="tm-count" aria-hidden="true">{#each team as _, i}<i class:on={i === teamModalIdx}></i>{/each}</div>
          <button class="tm-step next" on:click={() => stepMember(1)}>
            <small>Berikutnya →</small><span>{team[(teamModalIdx + 1) % team.length].name}</span>
          </button>
        </nav>
      </div>
    </div>
  {/if}

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

<footer class="kf">
  <div class="wrap kf-main">
    <div class="kf-brand">
      <a class="kf-logo" href="#top"><img src="/logo.png" alt="" /><span>Kalibata<br/>Project</span></a>
      <p class="kf-tag">Where ideas become projects. Produk digital dari ide, langsung bisa dipakai siapa saja.</p>
      <ul class="kf-contact">
        <li><a href={ig} target="_blank" rel="noopener noreferrer">{@render socialIcon('instagram')}<span>@kalibata.project</span></a></li>
        <li><a href="https://kalibataproject.my.id" target="_blank" rel="noopener noreferrer">{@render socialIcon('web')}<span>kalibataproject.my.id</span></a></li>
      </ul>
    </div>

    <div class="kf-col">
      <h4>Jelajahi</h4>
      <ul>
        <li><a href="#about">Tentang kami</a></li>
        <li><a href="#proyek">Semua proyek</a></li>
        <li><a href="#tim">Tim pengembang</a></li>
      </ul>
    </div>

    <div class="kf-col">
      <h4>Proyek</h4>
      <ul>
        {#each projects as p}
          <li><a href={p.url} target="_blank" rel="noopener noreferrer">{p.name}</a></li>
        {/each}
      </ul>
    </div>

    <div class="kf-col">
      <h4>Kolaborasi</h4>
      <p class="kf-note">Punya ide atau pertanyaan? Ceritakan lewat DM Instagram kami.</p>
      <a class="kf-cta" href={ig} target="_blank" rel="noopener noreferrer">Kirim DM <span aria-hidden="true">→</span></a>
    </div>

    <div class="kf-legal">
      <span>© {new Date().getFullYear()} Kalibata Project</span>
      <i aria-hidden="true"></i>
      <span>Since 2026</span>
    </div>
  </div>

  <!-- Lanskap bergaya ukiran: bukit berarsir, dataran krem, penunggang kuda -->
  <div class="kf-game" use:kfGame role="button" tabindex="0" aria-label="Minigame: lompatkan kuda melewati rintangan. Ketuk atau tekan Spasi.">
  <svg class="kf-land" viewBox="0 0 1440 360" preserveAspectRatio="xMidYMax slice" aria-hidden="true">
    <defs>
      <pattern id="kf-h1" width="8" height="5" patternUnits="userSpaceOnUse"><path d="M0 2.5h8" stroke="#fbf3ea" stroke-width="1.1" opacity=".55"/></pattern>
      <pattern id="kf-h2" width="8" height="3.4" patternUnits="userSpaceOnUse"><path d="M0 1.7h8" stroke="#fbf3ea" stroke-width="1.15"/></pattern>
      <pattern id="kf-h3" width="8" height="2.4" patternUnits="userSpaceOnUse"><path d="M0 1.2h8" stroke="#fbf3ea" stroke-width="1.2"/></pattern>
      <linearGradient id="kf-fade" x1="0" y1="0" x2="0" y2="1"><stop offset="0" stop-color="#d4181f" stop-opacity=".85"/><stop offset=".7" stop-color="#d4181f" stop-opacity="0"/></linearGradient>
      <g id="kf-tuft" fill="none" stroke="#7e0f14" stroke-width="1.7" stroke-linecap="round"><path d="M0 0C-1-9-5-16-12-22M0 0C0-10-1-18 1-27M0 0C2-9 6-15 13-21M0 0C-3-6-9-9-17-11M0 0C3-6 9-8 17-8M0 0C-1-5-3-8-6-10"/></g>
    </defs>

    <!-- punggung bukit jauh -->
    <path d="M0 190C70 150 150 148 235 172S395 160 520 172 700 150 800 166L960 150C1010 118 1050 92 1110 88C1170 84 1215 112 1262 140S1390 158 1440 148V236H0Z" fill="url(#kf-h1)" stroke="#fbf3ea" stroke-width="1.6" stroke-linejoin="round"/>
    <!-- mesa kanan -->
    <path d="M1010 150C1050 112 1075 82 1125 76C1175 72 1205 100 1250 138L1262 148Z" fill="url(#kf-h2)" stroke="#fbf3ea" stroke-width="1.4"/>
    <!-- bukit tengah -->
    <path d="M0 214C110 184 250 190 380 206S610 192 770 202 1040 186 1200 196 1380 190 1440 200V246H0Z" fill="url(#kf-h3)" stroke="#fbf3ea" stroke-width="1.5" stroke-linejoin="round"/>
    <path d="M0 214C110 184 250 190 380 206S610 192 770 202 1040 186 1200 196 1380 190 1440 200V246H0Z" fill="url(#kf-fade)"/>

    <!-- dataran -->
    <path d="M0 236C200 226 420 234 700 229S1220 227 1440 231V360H0Z" fill="#fbf3ea"/>
    <g stroke="#7e0f14" stroke-linecap="round" opacity=".22" fill="none">
      <path d="M30 250h70M260 258h110M470 246h60M700 252h120M900 262h90M1110 250h110M1290 258h70M150 286h90M380 300h120M820 292h100M1180 296h120"/>
    </g>

    <!-- rintangan & garis tanah (diisi minigame) -->
    <g class="kf-obs"></g>

    <!-- penunggang kuda -->
    <g transform="translate(560 270) scale(1.08)">
      <ellipse cx="0" cy="33" rx="42" ry="4" fill="#7e0f14" opacity=".22"/>
      <g class="kf-jump"><g class="kf-rider" fill="#7e0f14" stroke="#7e0f14" stroke-linecap="round" stroke-linejoin="round">
        <ellipse cx="0" cy="0" rx="26" ry="11" stroke="none"/>
        <path d="M17-7L29-30 38-28 29-2Z" stroke-width="2"/>
        <ellipse cx="40" cy="-21" rx="10" ry="4.6" transform="rotate(48 40 -21)" stroke="none"/>
        <path d="M30-31l1-6 4 5Z" stroke-width="1"/>
        <path d="M18 7L28 31M23 7L11 27M-20 7L-30 31M-15 7L-5 29" fill="none" stroke-width="4"/>
        <path d="M-25-4C-40 0-43 14-38 25" fill="none" stroke-width="4"/>
        <path d="M0-11L2-32M0-8L10 6" fill="none" stroke-width="6"/>
        <path d="M2-30L14-23" fill="none" stroke-width="3"/>
        <circle cx="3" cy="-40" r="4.6" stroke="none"/>
        <ellipse cx="3" cy="-44" rx="10" ry="2.2" stroke="none"/>
        <path d="M-1-44l2-8h8l2 8Z" stroke="none"/>
      </g></g>
    </g>

    <!-- rumpun rumput -->
    <g>{#each [[218,244,0.39],[1369,246,0.4],[1259,247,0.41],[962,248,0.42],[731,250,0.43],[297,250,0.43],[1106,250,0.43],[1010,251,0.43],[97,251,0.44],[131,252,0.44],[147,253,0.46],[913,255,0.47],[321,257,0.5],[357,258,0.5],[170,260,0.52],[704,260,0.52],[169,262,0.54],[334,263,0.55],[838,264,0.56],[234,267,0.59],[827,270,0.63],[1175,277,0.71],[843,278,0.72],[217,279,0.74],[76,281,0.76],[381,281,0.76],[1273,283,0.78],[177,283,0.79],[816,284,0.8],[963,286,0.82],[1255,286,0.83],[1090,289,0.87],[1191,290,0.88],[956,295,0.95],[889,300,1.02],[1260,301,1.03],[211,301,1.04],[1409,301,1.04],[90,303,1.07],[1272,303,1.07],[39,303,1.06],[378,308,1.14],[214,310,1.18],[1365,312,1.2],[536,313,1.22],[104,314,1.24],[1430,314,1.24],[616,317,1.29],[78,317,1.29],[856,319,1.32],[415,323,1.38],[825,327,1.45],[1007,330,1.5],[1244,332,1.55],[410,333,1.55],[232,334,1.56],[1360,335,1.59],[1430,336,1.6],[417,337,1.62],[1149,338,1.65],[452,339,1.65],[1123,341,1.7],[217,347,1.81],[994,347,1.8],[67,349,1.85],[170,350,1.86]] as t}<use href="#kf-tuft" transform="translate({t[0]} {t[1]}) scale({t[2]})"/>{/each}</g>
  </svg>
    <div class="kf-hud"><span class="kf-score"></span></div>
    <div class="kf-hint" hidden></div>
  </div>
</footer>

<style>
  :global(:root){--red:#d4181f;--maroon:#7e0f14;--cream:#fbf3ea;--green:#1fbf6b;--forest:#06361f;--pad:clamp(1rem,5vw,4.5rem)}
  :global(html){scroll-behavior:smooth;overflow-x:clip}
  :global(body){margin:0;background:var(--red);color:var(--cream);font-family:'Archivo',system-ui,sans-serif;line-height:1.55;overflow-x:hidden;overflow-x:clip;-webkit-text-size-adjust:100%}
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
  .band-wrap{position:relative;z-index:2;margin:-2.6rem 0;padding:1.6rem 0;overflow-x:clip;pointer-events:none}
  .band{position:relative;margin:0 -6vw;padding:.45rem 0;background:var(--maroon);color:var(--cream);transform:rotate(-2deg);overflow:hidden;white-space:nowrap}
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

  .totop{position:fixed;right:max(1rem,env(safe-area-inset-right));bottom:max(1rem,env(safe-area-inset-bottom));z-index:20;display:grid;place-items:center;width:44px;height:44px;border-radius:50%;background:var(--maroon);color:var(--cream);box-shadow:0 6px 20px rgba(0,0,0,.25);opacity:0;transform:translateY(14px) scale(.8);pointer-events:none;transition:opacity .35s,transform .35s cubic-bezier(.34,1.56,.64,1)}
  .totop.show{opacity:1;transform:none;pointer-events:auto}
  @media(hover:hover){.totop.show:hover{transform:translateY(-3px)}}
  .stat{transition:transform .3s}@media(hover:hover){.stat:hover{transform:translateY(-4px) rotate(-1deg)}}
  .nav{padding-top:max(.75rem,env(safe-area-inset-top))}

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

  /* ===== TIM: ROSTER + DITHER ===== */
  .team{position:relative;z-index:1;background:var(--red);color:var(--cream);padding:5rem 0 4.5rem;overflow:hidden}
  .team::before{content:'';position:absolute;inset:0;background:radial-gradient(55% 45% at 90% 5%,rgba(255,255,255,.16),transparent 70%),radial-gradient(60% 50% at 5% 100%,rgba(126,15,20,.45),transparent 70%);pointer-events:none}
  .team .wrap{position:relative}
  .team .label{background:var(--cream);color:var(--maroon)}
  .team .proj-sub{opacity:.85}
  .team-head{margin-bottom:2.5rem}.team-head h2{margin:0;font-size:clamp(2.6rem,9vw,6.5rem);line-height:.88;text-transform:uppercase}
  .roster{display:grid;grid-template-columns:1fr;gap:1.6rem;align-items:start}
  @media(min-width:900px){.roster{grid-template-columns:minmax(300px,.75fr) 1.4fr;gap:3.5rem}.stage{position:sticky;top:5rem}}

  .stage{position:relative;width:min(100%,340px);margin-inline:auto;border-radius:1.2rem;overflow:hidden;border:1px solid rgba(251,243,234,.16);background:#240407;box-shadow:0 24px 60px rgba(58,5,8,.55)}
  .stage-screen{position:relative;aspect-ratio:4/5;width:100%;cursor:crosshair;touch-action:pan-y}
  .stage-screen :global(canvas),.tm-screen :global(canvas){position:absolute;inset:0;width:100%;height:100%;display:block;image-rendering:pixelated}
  .hud{position:absolute;z-index:2;font-size:.6rem;font-weight:800;letter-spacing:.14em;text-transform:uppercase;color:var(--cream);mix-blend-mode:difference;pointer-events:none}
  .hud-tl{top:.8rem;left:.9rem}.hud-tr{top:.8rem;right:.9rem}.hud-bl{bottom:.8rem;left:.9rem}.hud-br{bottom:.8rem;right:.9rem}

  .rows{list-style:none;margin:0;padding:0;border-top:1px solid rgba(251,243,234,.35)}
  .row{all:unset;box-sizing:border-box;position:relative;isolation:isolate;overflow:hidden;cursor:pointer;width:100%;display:grid;grid-template-columns:1fr auto;grid-template-areas:"name go" "role go";align-items:center;column-gap:1.3rem;padding:1.3rem .4rem;border-bottom:1px solid rgba(251,243,234,.35)}
  .row::before{content:'';position:absolute;inset:0;z-index:-1;background:linear-gradient(90deg,rgba(126,15,20,.55),transparent 78%);transform:scaleX(0);transform-origin:left;transition:transform .6s cubic-bezier(.16,1,.3,1)}
  .row.on::before{transform:scaleX(1)}
  .row-name{grid-area:name;font-family:'Anton','Archivo',sans-serif;font-size:clamp(2rem,6.2vw,4.6rem);line-height:1.02;text-transform:uppercase;color:transparent;-webkit-text-stroke:1.4px rgba(251,243,234,.8);transition:color .35s,-webkit-text-stroke-color .35s,translate .55s cubic-bezier(.16,1,.3,1)}
  .row.on .row-name{color:var(--cream);-webkit-text-stroke-color:var(--cream);translate:.9rem 0}
  .row-role{grid-area:role;font-size:.72rem;font-weight:700;letter-spacing:.1em;text-transform:uppercase;opacity:.8;transition:translate .55s cubic-bezier(.16,1,.3,1)}
  .row.on .row-role{translate:.9rem 0}
  .row-go{grid-area:go;display:grid;place-items:center;width:3rem;height:3rem;border-radius:50%;border:1px solid rgba(251,243,234,.55);transition:all .45s cubic-bezier(.16,1,.3,1)}
  .row.on .row-go{background:var(--h);border-color:var(--h);color:#1c0204;rotate:-45deg}
  .row:focus-visible,.tm button:focus-visible,.tm a:focus-visible{outline:3px solid var(--cream);outline-offset:-3px}

  /* ===== PROFIL TIM (LEMBAR PENUH) ===== */
  .tm{position:fixed;inset:0;z-index:100;display:grid;grid-template-columns:1fr;grid-template-rows:minmax(0,32dvh) minmax(0,1fr);background:#1c0204;color:var(--cream);overflow:hidden}
  @media(min-width:900px){.tm{grid-template-columns:minmax(0,1.05fr) minmax(0,1fr);grid-template-rows:1fr}}
  .tm-visual{position:relative;overflow:hidden;background:#240407}
  .tm-screen{position:absolute;inset:0;cursor:crosshair;touch-action:pan-y}
  .tm-pixel{all:unset;cursor:pointer;position:absolute;z-index:3;top:1rem;left:1rem;padding:.5rem .9rem;border-radius:99px;background:rgba(28,2,4,.7);border:1px solid rgba(251,243,234,.3);backdrop-filter:blur(6px);font-size:.68rem;font-weight:800;letter-spacing:.1em;text-transform:uppercase;transition:all .3s}
  @media(hover:hover){.tm-pixel:hover{background:var(--cream);color:#1c0204}}

  .tm-panel{position:relative;display:flex;flex-direction:column;min-height:0;padding:clamp(.9rem,2.5vw,2rem) clamp(1.1rem,4vw,3.2rem) 0;overflow-y:auto;overscroll-behavior:contain}
  .tm-top{display:flex;justify-content:space-between;align-items:center;gap:1rem;font-size:.68rem;font-weight:800;letter-spacing:.14em;text-transform:uppercase}
  .tm-top span{opacity:.5}
  .tm-close{all:unset;cursor:pointer;display:grid;place-items:center;width:44px;height:44px;border-radius:50%;border:1px solid rgba(251,243,234,.28);transition:all .4s cubic-bezier(.16,1,.3,1)}
  @media(hover:hover){.tm-close:hover{background:var(--cream);color:#1c0204;rotate:90deg}}
  .tm-body{display:flex;flex-direction:column;gap:1.1rem;padding:1.2rem 0 1rem}
  .tm-role{align-self:flex-start;margin:0;padding:.3rem .85rem;border-radius:99px;background:var(--h);color:#1c0204;font-size:.7rem;font-weight:800;letter-spacing:.1em;text-transform:uppercase}
  .tm-name{margin:0;font-family:'Anton','Archivo',sans-serif;font-weight:400;font-size:clamp(2.6rem,7vw,5.4rem);line-height:.92;text-transform:uppercase}
  .tm-tagline{margin:0;opacity:.6;font-style:italic;max-width:42ch}
  .tm-tabs{position:relative;display:grid;grid-template-columns:repeat(3,1fr);border-bottom:1px solid rgba(251,243,234,.2)}
  .tm-tab{all:unset;cursor:pointer;text-align:center;padding:.85rem .3rem;font-size:.72rem;font-weight:800;letter-spacing:.1em;text-transform:uppercase;opacity:.5;transition:opacity .3s}
  .tm-tab[aria-selected=true]{opacity:1}
  @media(hover:hover){.tm-tab:hover{opacity:.9}}
  .tm-ind{position:absolute;left:0;bottom:-1px;width:calc(100%/3);height:3px;background:var(--h);transform:translateX(calc(var(--i)*100%));transition:transform .5s cubic-bezier(.16,1,.3,1)}
  .tm-pane{display:flex;flex-direction:column;gap:1rem;min-height:9.5rem}
  .tm-pane p{margin:0;line-height:1.75;opacity:.85;max-width:52ch}
  .tm-skills{display:grid;gap:.55rem;margin:0;padding:0;list-style:none}
  .tm-skills li{display:flex;align-items:center;gap:.9rem;padding:.75rem 1rem;border-radius:.85rem;border:1px solid rgba(251,243,234,.16);font-weight:700;transition:border-color .3s,background .3s,translate .4s cubic-bezier(.16,1,.3,1)}
  @media(hover:hover){.tm-skills li:hover{border-color:var(--h);background:color-mix(in srgb,var(--h) 16%,transparent);translate:.35rem 0}}
  .tm-links{display:flex;flex-wrap:wrap;gap:.5rem}
  .tm-link{display:inline-flex;align-items:center;gap:.45rem;padding:.55rem .95rem;border-radius:99px;border:1px solid rgba(251,243,234,.28);font-size:.78rem;font-weight:700;transition:all .3s}
  @media(hover:hover){.tm-link:hover{background:var(--cream);color:#1c0204;translate:0 -2px}}
  .tm-nav{position:sticky;bottom:0;margin-top:auto;display:grid;grid-template-columns:1fr auto 1fr;align-items:center;gap:.8rem;padding:1rem 0 calc(1rem + env(safe-area-inset-bottom));background:linear-gradient(to top,#1c0204 72%,transparent)}
  .tm-step{all:unset;cursor:pointer;display:flex;flex-direction:column;gap:.15rem;min-width:0;padding:.4rem .3rem}
  .tm-step small{font-size:.6rem;letter-spacing:.14em;text-transform:uppercase;font-weight:800;opacity:.5}
  .tm-step span{font-weight:700;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
  .tm-step.next{text-align:right}
  @media(hover:hover){.tm-step:hover small{opacity:1;color:var(--h)}}
  .tm-count{display:flex;gap:.35rem}
  .tm-count i{width:1.2rem;height:3px;border-radius:2px;background:rgba(251,243,234,.25);transition:all .45s cubic-bezier(.16,1,.3,1)}
  .tm-count i.on{width:2.4rem;background:var(--h)}

  @media(prefers-reduced-motion:reduce){.row::before,.row-name,.row-role,.row-go,.tm-ind,.tm-close,.tm-skills li,.tm-link{transition:none}.row.on .row-name,.row.on .row-role{translate:none}}

  /* kredit di detail proyek */
  .cred{margin-top:.4rem;padding:.9rem 1rem;border-radius:1rem;background:rgba(6,54,31,.1);max-width:46ch}
  .cred-t{margin:0 0 .5rem;font-size:.68rem;font-weight:800;letter-spacing:.1em;text-transform:uppercase;opacity:.7}
  .csteps{display:grid;gap:.5rem}
  .cstep{display:flex;align-items:center;gap:.65rem;animation:fli .5s cubic-bezier(.16,1,.3,1) both;animation-delay:calc(var(--ld)*100ms + 600ms)}
  .cstep-ico{display:grid;place-items:center;flex:none;width:2rem;height:2rem;border-radius:50%;background:linear-gradient(135deg,#ffd36b,#e8a317);color:#4a2c00;font-size:1rem;animation:cpop 2.8s ease-in-out infinite;animation-delay:calc(var(--ld)*.35s)}
  @keyframes cpop{0%,70%,100%{transform:none}80%{transform:scale(1.18) rotate(-8deg)}}
  .cstep-txt{display:flex;flex-direction:column;line-height:1.3}.cstep-txt strong{font-size:.85rem}.cstep-txt span{font-size:.74rem;opacity:.7}

  /* ===== FOOTER LANSKAP ===== */
  footer.kf{position:relative;overflow:hidden;text-align:left;background:var(--red);color:var(--cream);padding:4.2rem 0 0}
  footer.kf::before{content:'';position:absolute;inset:0;background-image:url("data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='160' height='160'><filter id='n'><feTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'/><feColorMatrix values='0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .5 0'/></filter><rect width='100%' height='100%' filter='url(%23n)'/></svg>");opacity:.12;mix-blend-mode:multiply;pointer-events:none}
  footer.kf::after{content:'';position:absolute;inset:0 0 auto 0;height:1px;background:rgba(251,243,234,.18)}

  .kf-main{position:relative;z-index:1;display:grid;grid-template-columns:1fr;gap:2rem 2.5rem;padding-bottom:2.2rem}
  .kf-logo{display:inline-flex;align-items:center;gap:.6rem}
  .kf-logo img{width:40px;height:40px;border-radius:9px;object-fit:cover}
  .kf-logo span{font-family:'Anton','Archivo',sans-serif;font-size:1.15rem;line-height:1;text-transform:uppercase;letter-spacing:.02em}
  .kf-tag{margin:.9rem 0 0;max-width:34ch;font-style:italic;font-size:.86rem;line-height:1.6;opacity:.85}

  footer.kf ul{list-style:none;margin:0;padding:0;max-width:none;display:grid;gap:.55rem}
  footer.kf li{animation:none}
  .kf-contact{margin-top:1.1rem!important}
  .kf-contact a{display:inline-flex;align-items:center;gap:.6rem;font-size:.84rem;opacity:.9}

  .kf-col h4{margin:0 0 1.15rem;padding-bottom:.7rem;position:relative;font-family:'Archivo',system-ui,sans-serif;font-weight:700;font-size:.72rem;letter-spacing:.16em;text-transform:uppercase}
  .kf-col h4::after{content:'';position:absolute;left:0;bottom:0;width:26px;height:1px;background:var(--cream);opacity:.7}
  .kf-col li a{position:relative;font-size:.86rem;opacity:.9;transition:opacity .3s}
  .kf-note{margin:0 0 1rem;max-width:30ch;font-style:italic;font-size:.84rem;line-height:1.6;opacity:.85}
  .kf-cta{display:inline-flex;align-items:center;gap:.5rem;padding:.6rem 1.15rem;border:1px solid rgba(251,243,234,.7);font-size:.82rem;font-weight:700;transition:background .3s,color .3s}
  .kf-cta span{transition:translate .35s cubic-bezier(.16,1,.3,1)}

  .kf-legal{display:flex;align-items:center;gap:.8rem;font-size:.72rem;font-style:italic;opacity:.8}
  .kf-legal i{width:1px;height:.9em;background:var(--cream);opacity:.6}

  .kf-game{position:relative;z-index:0;display:block;margin-top:.5rem;cursor:pointer;touch-action:pan-y;-webkit-tap-highlight-color:transparent;-webkit-user-select:none;user-select:none}
  .kf-game:focus-visible{outline:2px dashed var(--cream);outline-offset:-6px}
  .kf-land{display:block;width:100%;height:clamp(200px,25vw,380px)}
  .kf-hud{position:absolute;top:.7rem;right:var(--pad);font-size:.7rem;font-weight:800;letter-spacing:.12em;text-transform:uppercase;font-variant-numeric:tabular-nums;pointer-events:none}
  .kf-hint{position:absolute;left:50%;top:.9rem;translate:-50% 0;padding:.45rem .95rem;border-radius:99px;background:var(--maroon);color:var(--cream);font-size:.68rem;font-weight:800;letter-spacing:.08em;text-transform:uppercase;white-space:nowrap;pointer-events:none;box-shadow:0 4px 14px rgba(0,0,0,.18)}
  :global(.kf-run) .kf-rider{animation-duration:.3s}
  :global(.kf-air) .kf-rider{animation-play-state:paused}
  :global(.kf-over) .kf-rider{animation:none}
  .kf-rider{animation:kfride 1.1s ease-in-out infinite}
  @keyframes kfride{50%{translate:0 -2.5px}}

  @media(hover:hover){
    .kf-contact a:hover,.kf-col li a:hover{opacity:1}
    .kf-col li a::after{content:'';position:absolute;left:0;right:0;bottom:-2px;height:1px;background:var(--cream);transform:scaleX(0);transform-origin:right;transition:transform .35s cubic-bezier(.16,1,.3,1)}
    .kf-col li a:hover::after{transform:scaleX(1);transform-origin:left}
    .kf-cta:hover{background:var(--cream);color:var(--maroon)}
    .kf-cta:hover span{translate:4px 0}
  }
  @media(min-width:600px){.kf-main{grid-template-columns:repeat(2,1fr)}.kf-brand{grid-column:1/-1}}
  @media(min-width:960px){
    .kf-main{grid-template-columns:1.5fr 1fr 1fr 1.3fr;column-gap:3rem}
    .kf-brand{grid-column:auto}
    .kf-legal{grid-column:1/-1;justify-content:flex-end;margin-top:.6rem}
  }
  @media(max-width:599px){footer.kf{padding-top:3rem;text-align:left}.kf-land{height:190px}}
  @media(prefers-reduced-motion:reduce){.kf-rider{animation:none}.kf-cta,.kf-cta span,.kf-col li a,.kf-col li a::after{transition:none}}
</style>
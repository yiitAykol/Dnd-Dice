<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';
  import { World, Body, Box, Plane, Vec3, Material as CMaterial, ContactMaterial } from 'cannon-es';

  let host: HTMLDivElement;
  let width = 0, height = 0;

  // UI durumu
  export let count = 1;        // kaç adet d6
  export let autoClear = true; // yeni atışta eskileri temizle
  let rolling = false;
  let results: number[] = [];
  let sum = 0;

  // Panel / stage layout durumu
  let panelWidth = 320;
  const minPanelWidth = 220;
  const maxPanelWidth = 560;
  const stageMinWidth = 320;
  let resizing = false;

  // THREE
  let renderer: THREE.WebGLRenderer;
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let dirLight: THREE.DirectionalLight;
  let raf = 0;

  // CANNON
  let world: World;
  let groundBody: Body;

  // Zar tutucu
  type Dice = {
    mesh: THREE.Mesh,
    body: Body,
    getValue: () => number,
    landed: boolean
  };
  let diceList: Dice[] = [];

  // ---------- Helpers ----------
  function makeNumberTexture(n: number) {
    const size = 256;
    const canvas = document.createElement('canvas');
    canvas.width = size; canvas.height = size;
    const ctx = canvas.getContext('2d')!;
    // arka plan
    ctx.fillStyle = '#f5f5f5'; ctx.fillRect(0,0,size,size);
    // kenarlık
    ctx.strokeStyle = '#ddd'; ctx.lineWidth = 8; ctx.strokeRect(4,4,size-8,size-8);
    // numara
    ctx.fillStyle = '#111';
    ctx.font = 'bold 170px system-ui, Arial, sans-serif';
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillText(String(n), size/2, size/2 + 10);
    const tex = new THREE.CanvasTexture(canvas);
    tex.anisotropy = 4; tex.needsUpdate = true;
    return tex;
  }

  // BoxGeometry malzeme sırası: [px, nx, py, ny, pz, nz]
  // Standart zar dizilimi (karşıt yüzlerin toplamı 7): +Y=1, -Y=6, +X=3, -X=4, +Z=2, -Z=5
  const faceValues = [3, 4, 1, 6, 2, 5];
  const faceNormalsCannon = [
    new Vec3( 1, 0, 0), // px
    new Vec3(-1, 0, 0), // nx
    new Vec3( 0, 1, 0), // py (üst)
    new Vec3( 0,-1, 0), // ny (alt)
    new Vec3( 0, 0, 1), // pz
    new Vec3( 0, 0,-1)  // nz
  ];

  function createD6(): Dice {
    // THREE mesh
    const geom = new THREE.BoxGeometry(1,1,1);
    const materials = faceValues.map(v => new THREE.MeshStandardMaterial({
      map: makeNumberTexture(v),
      roughness: 0.6,
      metalness: 0.0
    }));
    const mesh = new THREE.Mesh(geom, materials);
    mesh.castShadow = true; mesh.receiveShadow = false;
    scene.add(mesh);

    // CANNON body
    const shape = new Box(new Vec3(0.5,0.5,0.5));
    const body = new Body({
      mass: 1,
      shape,
      position: new Vec3(
        (Math.random()-0.5) * 2.0,
        3 + Math.random()*1.0,
        (Math.random()-0.5) * 2.0
      ),
      angularDamping: 0.05,
      linearDamping: 0.01,
      allowSleep: true
    });
    body.sleepSpeedLimit = 0.15;   // hız < 0.15
    body.sleepTimeLimit = 0.25;    // 0.25 sn sonra uyut
    // ilk hız/torque (atış hissi)
    body.velocity.set(
      (Math.random()-0.5) * 4,
      2 + Math.random()*3,
      (Math.random()-0.5) * 4
    );
    body.angularVelocity.set(
      (Math.random()-0.5) * 20,
      (Math.random()-0.5) * 20,
      (Math.random()-0.5) * 20
    );
    world.addBody(body);

    // çarpma efekti (hafif ekran sarsıntısı & ışık flaşı)
    body.addEventListener('collide', (e: any) => {
      if (!host) return;
      host.classList.add('shake');
      dirLight.intensity = 1.3;
      setTimeout(() => { host?.classList.remove('shake'); dirLight.intensity = 1.0; }, 120);
    });

    const getValue = () => {
      const up = new Vec3(0,1,0);
      let best = -Infinity, idx = 0;
      for (let i=0; i<6; i++){
        const n = faceNormalsCannon[i];
        const worldN = body.quaternion.vmult(n);
        const dot = worldN.dot(up);
        if (dot > best){ best = dot; idx = i; }
      }
      return faceValues[idx];
    };

    return { mesh, body, getValue, landed: false };
  }

  function syncMeshes() {
    for (const d of diceList) {
      d.mesh.position.set(d.body.position.x, d.body.position.y, d.body.position.z);
      d.mesh.quaternion.set(d.body.quaternion.x, d.body.quaternion.y, d.body.quaternion.z, d.body.quaternion.w);
    }
  }

  function onResize() {
    if (!host) return;
    const wrap = host.parentElement;
    if (wrap) {
      const wrapRect = wrap.getBoundingClientRect();
      const maxPanel = Math.min(maxPanelWidth, wrapRect.width - stageMinWidth);
      const clampedPanel = Math.max(minPanelWidth, Math.min(panelWidth, maxPanel));
      if (clampedPanel !== panelWidth) panelWidth = clampedPanel;
    }
    const rect = host.getBoundingClientRect();
    width = rect.width;
    height = Math.max(rect.height, 1);
    renderer.setSize(width, height);
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
  }

  function roll() {
    if (rolling) return;
    if (autoClear) clearDice();
    rolling = true;
    results = []; sum = 0;
    for (let i=0; i<count; i++) {
      const d = createD6();
      diceList.push(d);
    }
  }

  function clearDice() {
    // THREE
    for (const d of diceList) { scene.remove(d.mesh); (d.mesh.geometry as any).dispose?.(); (d.mesh.material as any)?.forEach?.((m: any)=>m.dispose?.()); }
    // CANNON
    for (const d of diceList) world.removeBody(d.body);
    diceList = [];
    results = []; sum = 0;
  }

  // ---------- Layout ----------
  function startResize(event: PointerEvent) {
    if (event.button !== 0 || !host || resizing) return;
    resizing = true;
    const body = document.body;
    body.style.cursor = 'col-resize';
    body.style.userSelect = 'none';
    window.addEventListener('pointermove', handleResizeMove);
    window.addEventListener('pointerup', stopResize);
    window.addEventListener('pointercancel', stopResize);
    event.preventDefault();
  }

  function handleResizeMove(event: PointerEvent) {
    if (!resizing || !host) return;
    const wrap = host.parentElement;
    if (!wrap) return;
    const rect = wrap.getBoundingClientRect();
    const offset = event.clientX - rect.left;
    const maxPanel = Math.min(maxPanelWidth, rect.width - stageMinWidth);
    const nextWidth = Math.max(minPanelWidth, Math.min(offset, maxPanel));
    if (!Number.isFinite(nextWidth) || nextWidth === panelWidth) return;
    panelWidth = nextWidth;
    onResize();
    event.preventDefault();
  }

  function stopResize() {
    if (!resizing) return;
    resizing = false;
    const body = document.body;
    body.style.cursor = '';
    body.style.userSelect = '';
    window.removeEventListener('pointermove', handleResizeMove);
    window.removeEventListener('pointerup', stopResize);
    window.removeEventListener('pointercancel', stopResize);
    if (host) requestAnimationFrame(onResize);
  }

  // ---------- Lifecycle ----------
  onMount(() => {
    // Scene
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0e1012);
    camera = new THREE.PerspectiveCamera(50, 1, 0.1, 100);
    camera.position.set(5, 5, 6);
    camera.lookAt(0, 1.2, 0);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.shadowMap.enabled = true;
    host.appendChild(renderer.domElement);

    // ışıklar
    const amb = new THREE.AmbientLight(0xffffff, 0.55);
    scene.add(amb);
    dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
    dirLight.position.set(3,6,3);
    dirLight.castShadow = true;
    scene.add(dirLight);

    // zemin (THREE)
    const g = new THREE.PlaneGeometry(30,30);
    const m = new THREE.MeshStandardMaterial({ color: 0x111317, roughness: 0.9, metalness: 0.0 });
    const groundMesh = new THREE.Mesh(g, m);
    groundMesh.receiveShadow = true;
    groundMesh.rotation.x = -Math.PI/2;
    scene.add(groundMesh);

    // grid çizgiler
    const grid = new THREE.GridHelper(30, 30, 0x333a40, 0x22272e);
    (grid.material as THREE.Material).transparent = true;
    (grid.material as any).opacity = 0.35;
    scene.add(grid);

    // World (CANNON)
    world = new World({ gravity: new Vec3(0,-9.82,0), allowSleep: true });
    const groundMat = new CMaterial('ground');
    const diceMat = new CMaterial('dice');
    world.addContactMaterial(new ContactMaterial(groundMat, diceMat, { friction: 0.2, restitution: 0.35 }));
    // zemin (CANNON)
    groundBody = new Body({ mass: 0, material: groundMat });
    groundBody.addShape(new Plane());
    groundBody.quaternion.setFromEuler(-Math.PI/2, 0, 0);
    world.addBody(groundBody);

    // resize
    onResize(); window.addEventListener('resize', onResize);

    // animasyon döngüsü
    let last = performance.now();
    const step = () => {
      raf = requestAnimationFrame(step);
      const now = performance.now();
      const dt = Math.min((now - last)/1000, 1/30); // clamp
      last = now;

      world.step(1/60, dt, 3);
      syncMeshes();

      // iniş ve sonuç okuma
      if (rolling && diceList.length > 0) {
        let allSleeping = true;
        for (const d of diceList) {
          const speed = d.body.velocity.length();
          const ang = d.body.angularVelocity.length();
          const sleeping = (speed < 0.05 && ang < 0.05 && d.body.position.y < 1.2);
          if (sleeping && !d.landed) {
            d.landed = true;
            const val = d.getValue();
            results.push(val);
            sum = results.reduce((a,b)=>a+b, 0);
          }
          if (!sleeping) allSleeping = false;
        }
        if (allSleeping) rolling = false;
      }

      renderer.render(scene, camera);
    };
    step();

    return () => {
      cancelAnimationFrame(raf);
      window.removeEventListener('resize', onResize);
      clearDice();
      renderer.dispose();
    };
  });

  onDestroy(() => {
    if (resizing) stopResize();
    try { cancelAnimationFrame(raf); } catch {}
  });
</script>

<div class="wrap" class:resizing={resizing} style={`--panel-width: ${panelWidth}px`}>
  <div class="panel">
    <div class="row">
      <label>Adet</label>
      <input type="range" min="1" max="10" bind:value={count}>
      <span class="val">{count}</span>
    </div>
    <div class="row">
      <label><input type="checkbox" bind:checked={autoClear}> Yeni atışta temizle</label>
    </div>
    <div class="buttons">
      <button class="roll" disabled={rolling} on:click={roll}>🎲 Roll d6</button>
      <button class="clear" on:click={clearDice}>Temizle</button>
    </div>
    <div class="row results">
      <div>Sonuçlar: {results.join(', ') || '—'}</div>
      <div>Toplam: <b>{sum || '—'}</b></div>
    </div>
  </div>
  <div
    class="splitter"
    role="separator"
    aria-orientation="vertical"
    on:pointerdown={startResize}
  ></div>
  <div class="stage" bind:this={host}></div>
</div>

<style>
  .wrap { display: flex; align-items: stretch; width: 100%; height: 100%; gap: 0; }
  .wrap.resizing { cursor: col-resize; }
  .panel { flex: 0 0 var(--panel-width, 320px); width: var(--panel-width, 320px); min-width: 220px; max-width: 560px;
           background: rgba(20,22,26,0.7); backdrop-filter: blur(8px);
           border: 1px solid #2a2f36; border-radius: 14px; padding: 12px; color: #e9eef3; box-sizing: border-box; }
  .splitter { flex: 0 0 6px; margin: 0 6px; border-radius: 12px; background: rgba(17,20,25,0.75);
              border: 1px solid #262b32; cursor: col-resize; position: relative; transition: background 0.15s ease; }
  .splitter::before { content: ''; position: absolute; top: 50%; left: 50%; width: 2px; height: calc(100% - 20px);
                      background: #343b45; border-radius: 2px; transform: translate(-50%, -50%); opacity: 0.7;
                      transition: background 0.15s ease, opacity 0.15s ease; }
  .splitter:hover::before, .wrap.resizing .splitter::before { opacity: 1; background: #4c7cf7; }
  .row { display: flex; align-items: center; gap: 8px; margin: 8px 0; }
  .row.results { flex-direction: column; align-items: flex-start; gap: 4px; font-size: 14px; }
  .row label { color: #bfc7d2; font-size: 14px; }
  .val { margin-left: auto; opacity: .8; }
  .buttons { display: flex; gap: 8px; margin-top: 6px; }
  button { cursor: pointer; border-radius: 12px; padding: 10px 12px; border: 1px solid #2b323a; background: #171a1f; color: #e9eef3; }
  button.roll { background: #1f6feb; border-color: #295ebd; }
  button:disabled { opacity: .6; cursor: default; }
  .stage { flex: 1 1 auto; min-width: 0; position: relative; border: 1px solid #2a2f36; border-radius: 14px; }
  .stage canvas { width: 100%; height: 100%; display: block; border-radius: 14px; }
  .wrap, .stage { min-height: 70vh; }

  /* Çarpma anı sarsıntı */
  .stage.shake canvas { animation: shakey 120ms linear 1; }
  @keyframes shakey {
    0% { transform: translate(0,0) }
    20% { transform: translate(2px,-1px) }
    40% { transform: translate(-2px,1px) }
    60% { transform: translate(2px,1px) }
    80% { transform: translate(-2px,-1px) }
    100% { transform: translate(0,0) }
  }
</style>

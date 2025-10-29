<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';
  import { World, Body, Box, Plane, Vec3, Material as CMaterial, ContactMaterial, ConvexPolyhedron } from 'cannon-es';

  let host: HTMLDivElement;
  let width = 0, height = 0;

  // UI durumu
  export let count = 1;        // kaç adet d6
  export let autoClear = true; // yeni atışta eskileri temizle
  let rolling = false;
  let results: number[] = [];
  let sum = 0;

  type DiceTypeKey = 'd6' | 'd8' | 'd12';
  let diceType: DiceTypeKey = 'd6';

  const diceOptions: { key: DiceTypeKey; label: string; sides: number }[] = [
    { key: 'd6', label: 'd6', sides: 6 },
    { key: 'd8', label: 'd8', sides: 8 },
    { key: 'd12', label: 'd12', sides: 12 }
  ];

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
    landed: boolean,
    type: DiceTypeKey
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

  const polyTextureCache = new Map<string, THREE.CanvasTexture>();

  function makePolyFaceTexture(value: number, baseColor: THREE.Color) {
    const key = `${value}_${baseColor.getHexString()}`;
    const cached = polyTextureCache.get(key);
    if (cached) return cached;

    const size = 256;
    const canvas = document.createElement('canvas');
    canvas.width = size; canvas.height = size;
    const ctx = canvas.getContext('2d')!;

    ctx.clearRect(0, 0, size, size);
    const light = baseColor.clone().lerp(new THREE.Color('#ffffff'), 0.6);
    const border = baseColor.clone().lerp(new THREE.Color('#000000'), 0.45);
    const radius = size * 0.38;
    ctx.beginPath();
    ctx.fillStyle = light.getStyle();
    ctx.arc(size / 2, size / 2, radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.lineWidth = size * 0.07;
    ctx.strokeStyle = border.getStyle();
    ctx.stroke();

    ctx.fillStyle = '#0d1117';
    ctx.font = 'bold 150px system-ui, Arial, sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(String(value), size / 2, size / 2 + 6);

    const tex = new THREE.CanvasTexture(canvas);
    tex.anisotropy = 4;
    tex.needsUpdate = true;
    polyTextureCache.set(key, tex);
    return tex;
  }

  // BoxGeometry malzeme sırası: [px, nx, py, ny, pz, nz]
  // Standart zar dizilimi (karşıt yüzlerin toplamı 7): +Y=1, -Y=6, +X=3, -X=4, +Z=2, -Z=5
  const d6FaceValues = [3, 4, 1, 6, 2, 5];
  const d6FaceNormals = [
    new Vec3( 1, 0, 0), // px
    new Vec3(-1, 0, 0), // nx
    new Vec3( 0, 1, 0), // py (üst)
    new Vec3( 0,-1, 0), // ny (alt)
    new Vec3( 0, 0, 1), // pz
    new Vec3( 0, 0,-1)  // nz
  ];
  const worldUp = new Vec3(0, 1, 0);

  function createD6(): Dice {
    // THREE mesh
    const geom = new THREE.BoxGeometry(1,1,1);
    const materials = d6FaceValues.map(v => new THREE.MeshStandardMaterial({
      map: makeNumberTexture(v),
      roughness: 0.6,
      metalness: 0.0
    }));
    const mesh = new THREE.Mesh(geom, materials);
    mesh.castShadow = true; mesh.receiveShadow = false;
    scene.add(mesh);

    const shape = new Box(new Vec3(0.5,0.5,0.5));
    const body = makeBody(shape, 1);
    const probe = new Vec3();

    const getValue = () => {
      let best = -Infinity, idx = 0;
      for (let i=0; i<6; i++){
        const n = d6FaceNormals[i];
        body.quaternion.vmult(n, probe);
        const dot = probe.dot(worldUp);
        if (dot > best){ best = dot; idx = i; }
      }
      return d6FaceValues[idx];
    };

    return { mesh, body, getValue, landed: false, type: 'd6' };
  }

  function attachImpactEffect(body: Body) {
    body.addEventListener('collide', () => {
      if (!host) return;
      host.classList.add('shake');
      dirLight.intensity = 1.3;
      setTimeout(() => {
        host?.classList.remove('shake');
        dirLight.intensity = 1.0;
      }, 120);
    });
  }

  function makeBody(shape: any, mass = 1) {
    const body = new Body({
      mass,
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
    body.sleepSpeedLimit = 0.15;
    body.sleepTimeLimit = 0.25;
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
    attachImpactEffect(body);
    world.addBody(body);
    return body;
  }

  type PolyDiceResources = {
    vertices: number[];
    indices: number[];
    radius: number;
    mass: number;
    color: number | string;
    scaledVertices: number[];
    faces: number[][];
    triangleNormals: Vec3[];
    triangleValues: number[];
    triangleToGroup: number[];
    groups: { normal: Vec3; triangles: number[]; value: number }[];
  };

  function buildPolyDiceResources(config: {
    vertices: number[];
    indices: number[];
    radius: number;
    color: number | string;
    mass?: number;
  }): PolyDiceResources {
    const { vertices, indices, radius, color } = config;
    const mass = config.mass ?? 1;
    const scaledVertices: number[] = [];
    for (let i = 0; i < vertices.length; i += 3) {
      const x = vertices[i];
      const y = vertices[i + 1];
      const z = vertices[i + 2];
      const len = Math.sqrt(x * x + y * y + z * z) || 1;
      const scale = radius / len;
      scaledVertices.push(x * scale, y * scale, z * scale);
    }

    const faces: number[][] = [];
    const triangleToGroup: number[] = [];
    for (let i = 0; i < indices.length; i += 3) {
      faces.push([indices[i], indices[i + 1], indices[i + 2]]);
      triangleToGroup.push(-1);
    }

    const triangleNormals: Vec3[] = [];
    const triangleValues: number[] = [];
    const groups = new Map<string, { normal: Vec3; triangles: number[] }>();
    const vA = new THREE.Vector3();
    const vB = new THREE.Vector3();
    const vC = new THREE.Vector3();
    const edgeAB = new THREE.Vector3();
    const edgeAC = new THREE.Vector3();

    const setVertex = (index: number, target: THREE.Vector3) => {
      const offset = index * 3;
      target.set(
        scaledVertices[offset],
        scaledVertices[offset + 1],
        scaledVertices[offset + 2]
      );
      return target;
    };

    for (let i = 0; i < faces.length; i++) {
      const face = faces[i];
      const a = setVertex(face[0], vA);
      const b = setVertex(face[1], vB);
      const c = setVertex(face[2], vC);
      edgeAB.copy(b).sub(a);
      edgeAC.copy(c).sub(a);
      edgeAB.cross(edgeAC).normalize();
      const normal = new Vec3(edgeAB.x, edgeAB.y, edgeAB.z);
      triangleNormals.push(normal);
      triangleValues.push(0);
      const key = `${Math.round(normal.x * 1000)}_${Math.round(normal.y * 1000)}_${Math.round(normal.z * 1000)}`;
      let group = groups.get(key);
      if (!group) {
        group = { normal, triangles: [] };
        groups.set(key, group);
      }
      group.triangles.push(i);
    }

    const grouped = Array.from(groups.values());
    grouped.sort((a, b) => {
      if (Math.abs(a.normal.y - b.normal.y) > 1e-3) return b.normal.y - a.normal.y;
      if (Math.abs(a.normal.z - b.normal.z) > 1e-3) return b.normal.z - a.normal.z;
      return b.normal.x - a.normal.x;
    });
    const groupsWithValues = grouped.map((group, idx) => {
      const value = idx + 1;
      for (const tri of group.triangles) {
        triangleValues[tri] = value;
        triangleToGroup[tri] = idx;
      }
      return { normal: group.normal, triangles: [...group.triangles], value };
    });

    return { vertices, indices, radius, mass, color, scaledVertices, faces, triangleNormals, triangleValues, triangleToGroup, groups: groupsWithValues };
  }

  const d8Resources = buildPolyDiceResources({
    vertices: [
      1, 0, 0,  -1, 0, 0,  0, 1, 0,
      0,-1, 0,  0, 0, 1,  0, 0,-1
    ],
    indices: [
      0, 2, 4,  0, 4, 3,  0, 3, 5,
      0, 5, 2,  1, 2, 5,  1, 5, 3,
      1, 3, 4,  1, 4, 2
    ],
    radius: 0.92,
    color: '#f2994a',
    mass: 1
  });

  const goldenPhi = (1 + Math.sqrt(5)) / 2;
  const goldenInv = 1 / goldenPhi;

  const d12Resources = buildPolyDiceResources({
    vertices: [
      -1, -1, -1,  -1, -1, 1,
      -1, 1, -1,  -1, 1, 1,
       1, -1, -1,  1, -1, 1,
       1, 1, -1,   1, 1, 1,
       0, -goldenInv, -goldenPhi,  0, -goldenInv, goldenPhi,
       0,  goldenInv, -goldenPhi,  0,  goldenInv, goldenPhi,
      -goldenInv, -goldenPhi, 0,   -goldenInv, goldenPhi, 0,
       goldenInv, -goldenPhi, 0,    goldenInv, goldenPhi, 0,
      -goldenPhi, 0, -goldenInv,    goldenPhi, 0, -goldenInv,
      -goldenPhi, 0,  goldenInv,    goldenPhi, 0,  goldenInv
    ],
    indices: [
      3, 11, 7,   3, 7, 15,   3, 15, 13,
      7, 19, 17,  7, 17, 6,   7, 6, 15,
      17, 4, 8,   17, 8, 10,  17, 10, 6,
      8, 0, 16,   8, 16, 2,   8, 2, 10,
      0, 12, 1,   0, 1, 18,   0, 18, 16,
      6, 10, 2,   6, 2, 13,   6, 13, 15,
      2, 16, 18,  2, 18, 3,   2, 3, 13,
      18, 1, 9,   18, 9, 11,  18, 11, 3,
      4, 14, 12,  4, 12, 0,   4, 0, 8,
      11, 9, 5,   11, 5, 19,  11, 19, 7,
      19, 5, 14,  19, 14, 4,  19, 4, 17,
      1, 12, 14,  1, 14, 5,   1, 5, 9
    ],
    radius: 0.96,
    color: '#4cc9f0',
    mass: 1.3
  });

  function createPolyDice(resources: PolyDiceResources, type: DiceTypeKey): Dice {
    const geometry = new THREE.PolyhedronGeometry(resources.vertices, resources.indices, resources.radius, 0);
    const baseColor = new THREE.Color(resources.color as THREE.ColorRepresentation);
    const meshMaterial = new THREE.MeshStandardMaterial({
      color: baseColor,
      flatShading: true,
      roughness: 0.55,
      metalness: 0.1
    });
    const mesh = new THREE.Mesh(geometry, meshMaterial);
    mesh.castShadow = true;
    mesh.receiveShadow = false;
    scene.add(mesh);

    const labelMeshes: THREE.Mesh[] = [];
    const tempQuat = new THREE.Quaternion();
    const forward = new THREE.Vector3(0, 0, 1);
    const a = new THREE.Vector3();
    const b = new THREE.Vector3();
    const c = new THREE.Vector3();
    const centroid = new THREE.Vector3();
    const normalVec = new THREE.Vector3();

    const setVertex = (index: number, target: THREE.Vector3) => {
      const offset = index * 3;
      target.set(
        resources.scaledVertices[offset],
        resources.scaledVertices[offset + 1],
        resources.scaledVertices[offset + 2]
      );
      return target;
    };

    const labelSize = resources.radius * 0.65;
    const labelOffset = resources.radius * 0.06;
    const triCenter = new THREE.Vector3();

    for (const group of resources.groups) {
      const value = group.value;
      centroid.set(0, 0, 0);
      let triCount = 0;
      for (const triIndex of group.triangles) {
        const face = resources.faces[triIndex];
        setVertex(face[0], a);
        setVertex(face[1], b);
        setVertex(face[2], c);
        triCenter.copy(a).add(b).add(c).multiplyScalar(1 / 3);
        centroid.add(triCenter);
        triCount++;
      }
      if (triCount === 0) continue;
      centroid.multiplyScalar(1 / triCount);
      normalVec.set(group.normal.x, group.normal.y, group.normal.z).normalize();

      const labelTexture = makePolyFaceTexture(value, baseColor);
      const labelMaterial = new THREE.MeshBasicMaterial({ map: labelTexture, transparent: true, depthWrite: false });
      labelMaterial.userData.keepTexture = true;
      const labelGeometry = new THREE.PlaneGeometry(labelSize, labelSize);
      const labelMesh = new THREE.Mesh(labelGeometry, labelMaterial);
      labelMesh.position.copy(centroid).addScaledVector(normalVec, labelOffset);
      tempQuat.setFromUnitVectors(forward, normalVec);
      labelMesh.quaternion.copy(tempQuat);
      mesh.add(labelMesh);
      labelMeshes.push(labelMesh);
    }

    mesh.userData.labelMeshes = labelMeshes;

    const verticesVec3: Vec3[] = [];
    for (let i = 0; i < resources.scaledVertices.length; i += 3) {
      verticesVec3.push(new Vec3(
        resources.scaledVertices[i],
        resources.scaledVertices[i + 1],
        resources.scaledVertices[i + 2]
      ));
    }
    const normalsForShape = resources.triangleNormals.map(n => new Vec3(n.x, n.y, n.z));
    const shape = new ConvexPolyhedron({
      vertices: verticesVec3,
      faces: resources.faces,
      normals: normalsForShape
    });
    shape.computeEdges();
    shape.computeNormals();
    shape.updateBoundingSphereRadius();

    const body = makeBody(shape, resources.mass);
    const probe = new Vec3();

    const getValue = () => {
      let best = -Infinity;
      let value = 1;
      for (let i = 0; i < resources.triangleNormals.length; i++) {
        body.quaternion.vmult(resources.triangleNormals[i], probe);
        const dot = probe.dot(worldUp);
        if (dot > best) {
          best = dot;
          value = resources.triangleValues[i];
        }
      }
      return value;
    };

    return { mesh, body, getValue, landed: false, type };
  }

  function createDiceByType(type: DiceTypeKey): Dice {
    if (type === 'd6') return createD6();
    if (type === 'd8') return createPolyDice(d8Resources, type);
    if (type === 'd12') return createPolyDice(d12Resources, type);
    return createD6();
  }

  function disposeMesh(mesh: THREE.Mesh) {
    (mesh.geometry as THREE.BufferGeometry).dispose();
    const disposeMaterial = (mat: THREE.Material) => {
      const withMap = mat as THREE.Material & { map?: THREE.Texture | null; userData?: Record<string, any> };
      if (!withMap.userData?.keepTexture) withMap.map?.dispose();
      mat.dispose();
    };
    const material = mesh.material;
    if (Array.isArray(material)) {
      material.forEach(disposeMaterial);
    } else {
      disposeMaterial(material as THREE.Material);
    }

    const labelMeshes = mesh.userData.labelMeshes as THREE.Mesh[] | undefined;
    if (labelMeshes) {
      for (const label of labelMeshes) {
        mesh.remove(label);
        (label.geometry as THREE.BufferGeometry).dispose();
        disposeMaterial(label.material as THREE.Material);
      }
      delete mesh.userData.labelMeshes;
    }
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
      const d = createDiceByType(diceType);
      diceList.push(d);
    }
  }

  function clearDice() {
    for (const d of diceList) {
      scene.remove(d.mesh);
      disposeMesh(d.mesh);
      world.removeBody(d.body);
    }
    diceList = [];
    rolling = false;
    results = [];
    sum = 0;
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
      <label for="dice-count">Adet</label>
      <input id="dice-count" type="range" min="1" max="10" bind:value={count}>
      <span class="val">{count}</span>
    </div>
    <div class="row">
      <label for="dice-type">Zar</label>
      <select id="dice-type" bind:value={diceType}>
        {#each diceOptions as opt}
          <option value={opt.key}>{opt.label}</option>
        {/each}
      </select>
    </div>
    <div class="row">
      <label><input type="checkbox" bind:checked={autoClear}> Yeni atışta temizle</label>
    </div>
    <div class="buttons">
      <button class="roll" disabled={rolling} on:click={roll}>🎲 Roll {diceType}</button>
      <button class="clear" on:click={clearDice}>Temizle</button>
    </div>
    <div class="row results">
      <div>Sonuclar ({diceType}): {results.join(', ') || '-'}</div>
      <div>Toplam: <b>{sum || '-'}</b></div>
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
  .row select { flex: 1 1 auto; margin-left: auto; background: #15181d; color: #e9eef3; border: 1px solid #2b323a; border-radius: 10px; padding: 6px 8px; }
  .row select:focus { outline: none; border-color: #4c7cf7; }
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

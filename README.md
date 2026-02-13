<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Hyper Flight (Web 3D)</title>
  <style>
    html, body { margin:0; padding:0; overflow:hidden; background:#000; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, "Noto Sans KR", sans-serif; }
    #wrap { position:fixed; inset:0; }
    #fx {
      position:fixed; inset:0;
      width:100vw; height:100vh;
      pointer-events:none;
      mix-blend-mode: screen;
    }
    #ui {
      position:fixed; left:14px; top:14px;
      color:rgba(255,255,255,0.92);
      font-size:14px; line-height:1.35;
      text-shadow: 0 1px 8px rgba(0,0,0,0.6);
      user-select:none;
      padding:10px 12px;
      background: rgba(0,0,0,0.22);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 10px;
      backdrop-filter: blur(6px);
      max-width: 360px;
    }
    #ui b { font-weight: 700; }
    #hint {
      position:fixed; inset:auto 14px 14px auto;
      color:rgba(255,255,255,0.92);
      font-size:13px;
      user-select:none;
      padding:10px 12px;
      background: rgba(0,0,0,0.22);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 10px;
      backdrop-filter: blur(6px);
      text-align:right;
      max-width: 380px;
    }
    .bar {
      width: 240px; height: 8px;
      background: rgba(255,255,255,0.15);
      border-radius: 99px;
      overflow:hidden;
      margin-top: 6px;
      border: 1px solid rgba(255,255,255,0.12);
    }
    .bar > div {
      height:100%;
      width: 0%;
      background: rgba(255,255,255,0.85);
      border-radius: 99px;
    }
    #centerMsg {
      position:fixed; inset:0;
      display:flex;
      align-items:center; justify-content:center;
      pointer-events:none;
      color: rgba(255,255,255,0.92);
      font-size: 18px;
      text-shadow: 0 1px 18px rgba(0,0,0,0.7);
      opacity: 1;
      transition: opacity 180ms ease;
    }
    #centerMsg.hidden { opacity: 0; }
    #damage {
      position:fixed; inset:0;
      pointer-events:none;
      background: rgba(255,0,0,0.0);
      transition: background 80ms linear;
    }
  </style>
</head>
<body>
  <div id="wrap"></div>
  <canvas id="fx"></canvas>

  <div id="damage"></div>

  <div id="ui">
    <div><b>상태</b>: <span id="status">클릭해서 시작</span></div>
    <div>속도: <span id="speed">0</span> m/s</div>
    <div>고도: <span id="alt">0</span> m</div>
    <div>점수(링): <span id="score">0</span></div>
    <div style="margin-top:8px"><b>아드레날린</b>: <span id="adrTxt">0%</span></div>
    <div class="bar"><div id="adrBar"></div></div>
    <div style="margin-top:10px; opacity:0.92">
      가까이 스칠수록 아드레날린↑ → 최고속/연출↑<br/>
      (건물 사이를 “활공”하듯 파고들수록 더 빨라집니다)
    </div>
  </div>

  <div id="hint">
    <div><b>조작</b></div>
    <div>클릭: 포인터 잠금 / 시작</div>
    <div>마우스: 방향 조준</div>
    <div>W/S: 가속/감속 &nbsp; A/D: 좌/우 스트레이프</div>
    <div>Q/E: 롤(은행) &nbsp; Shift: 부스트(아드레날린 필요)</div>
    <div>R: 리셋</div>
  </div>

  <div id="centerMsg">클릭해서 시작 (마우스 잠금)</div>

  <!-- Three.js -->
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>

  <script>
  (() => {
    if (!window.THREE) {
      alert("Three.js 로드 실패. 인터넷 연결/차단 여부를 확인해주세요.");
      return;
    }

    // ---------- DOM ----------
    const statusEl = document.getElementById('status');
    const speedEl  = document.getElementById('speed');
    const altEl    = document.getElementById('alt');
    const scoreEl  = document.getElementById('score');
    const adrTxtEl = document.getElementById('adrTxt');
    const adrBarEl = document.getElementById('adrBar');
    const centerMsg = document.getElementById('centerMsg');
    const damageEl = document.getElementById('damage');

    // ---------- Renderer / Scene / Camera ----------
    const wrap = document.getElementById('wrap');
    const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
    renderer.setSize(window.innerWidth, window.innerHeight);
    wrap.appendChild(renderer.domElement);

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x86c7ff);
    scene.fog = new THREE.FogExp2(0x86c7ff, 0.00013);

    const camera = new THREE.PerspectiveCamera(78, window.innerWidth/window.innerHeight, 0.1, 20000);

    // Lighting
    const hemi = new THREE.HemisphereLight(0xffffff, 0x334455, 0.85);
    scene.add(hemi);

    const sun = new THREE.DirectionalLight(0xffffff, 1.0);
    sun.position.set(900, 1200, 600);
    sun.castShadow = false;
    scene.add(sun);

    // Ground
    const groundGeo = new THREE.PlaneGeometry(24000, 24000, 1, 1);
    const groundMat = new THREE.MeshStandardMaterial({ color: 0x1b2a35, roughness: 1.0, metalness: 0.0 });
    const ground = new THREE.Mesh(groundGeo, groundMat);
    ground.rotation.x = -Math.PI / 2;
    ground.position.y = 0;
    scene.add(ground);

    // Grid helper (속도감/스케일 기준점)
    const grid = new THREE.GridHelper(24000, 240, 0x244255, 0x244255);
    grid.material.opacity = 0.45;
    grid.material.transparent = true;
    scene.add(grid);

    // ---------- City (Instanced) ----------
    const COUNT_X = 90;
    const COUNT_Z = 90;
    const SPACING = 60;         // 빌딩 간격
    const JITTER  = 18;         // 랜덤 흔들림
    const WORLD_HALF_X = (COUNT_X - 1) * SPACING * 0.5;
    const WORLD_HALF_Z = (COUNT_Z - 1) * SPACING * 0.5;

    const buildingGeo = new THREE.BoxGeometry(1, 1, 1);
    const buildingMat = new THREE.MeshStandardMaterial({
      color: 0x9aa6b7,
      roughness: 0.95,
      metalness: 0.05
    });
    const totalBuildings = COUNT_X * COUNT_Z;
    const buildingsMesh = new THREE.InstancedMesh(buildingGeo, buildingMat, totalBuildings);
    buildingsMesh.frustumCulled = true;
    scene.add(buildingsMesh);

    const dummy = new THREE.Object3D();

    // 빌딩 충돌/근접체크용 데이터
    const buildings = new Array(totalBuildings);
    const cellSize = SPACING * 2;                 // 공간 해시 셀 크기
    const cellMap = new Map();                    // key -> indices[]
    function cellKey(cx, cz) { return cx + "|" + cz; }
    function putInCell(ix, iz, idx) {
      const cx = Math.floor(ix / cellSize);
      const cz = Math.floor(iz / cellSize);
      const key = cellKey(cx, cz);
      const arr = cellMap.get(key);
      if (arr) arr.push(idx);
      else cellMap.set(key, [idx]);
    }

    function rand(min, max) { return min + Math.random() * (max - min); }

    let bi = 0;
    for (let x = 0; x < COUNT_X; x++) {
      for (let z = 0; z < COUNT_Z; z++) {
        const px = (x * SPACING) - WORLD_HALF_X + rand(-JITTER, JITTER);
        const pz = (z * SPACING) - WORLD_HALF_Z + rand(-JITTER, JITTER);

        // 중심부는 더 높게, 외곽은 낮게(도시 느낌)
        const centerFalloff =
          1.0 - Math.min(1.0, Math.sqrt((px*px)/(WORLD_HALF_X*WORLD_HALF_X) + (pz*pz)/(WORLD_HALF_Z*WORLD_HALF_Z)));

        const w = rand(14, 34);
        const d = rand(14, 34);
        const h = rand(16, 90) + rand(0, 220) * Math.pow(centerFalloff, 1.35);

        dummy.position.set(px, h * 0.5, pz);
        dummy.scale.set(w, h, d);
        dummy.rotation.set(0, 0, 0);
        dummy.updateMatrix();
        buildingsMesh.setMatrixAt(bi, dummy.matrix);

        buildings[bi] = {
          minX: px - w * 0.5,
          maxX: px + w * 0.5,
          minZ: pz - d * 0.5,
          maxZ: pz + d * 0.5,
          maxY: h
        };
        putInCell(px, pz, bi);

        bi++;
      }
    }
    buildingsMesh.instanceMatrix.needsUpdate = true;

    // ---------- Rings ----------
    const rings = [];
    const ringCount = 18;
    const ringGeo = new THREE.TorusGeometry(10, 1.2, 10, 28);
    const ringMat = new THREE.MeshStandardMaterial({
      color: 0xffffff,
      roughness: 0.3,
      metalness: 0.0,
      emissive: 0x7fd7ff,
      emissiveIntensity: 1.35
    });

    for (let i = 0; i < ringCount; i++) {
      const ring = new THREE.Mesh(ringGeo, ringMat);
      ring.position.set(rand(-500, 500), rand(60, 260), rand(-500, 500));
      ring.rotation.set(rand(0, Math.PI), rand(0, Math.PI), rand(0, Math.PI));
      scene.add(ring);
      rings.push(ring);
    }

    // ---------- Player / Controls ----------
    const keys = Object.create(null);
    window.addEventListener('keydown', (e) => { keys[e.code] = true; });
    window.addEventListener('keyup',   (e) => { keys[e.code] = false; });

    let pointerLocked = false;
    let mouseDX = 0, mouseDY = 0;

    function requestLock() {
      if (!pointerLocked) document.body.requestPointerLock?.();
    }

    document.body.addEventListener('click', requestLock);
    document.addEventListener('pointerlockchange', () => {
      pointerLocked = (document.pointerLockElement === document.body);
      statusEl.textContent = pointerLocked ? "FLY" : "클릭해서 시작";
      centerMsg.classList.toggle('hidden', pointerLocked);
    });

    document.addEventListener('mousemove', (e) => {
      if (!pointerLocked) return;
      mouseDX += e.movementX || 0;
      mouseDY += e.movementY || 0;
    });

    // Player state
    const player = {
      pos: new THREE.Vector3(0, 140, 900),
      yaw: 0,
      pitch: 0,
      roll: 0,
      quat: new THREE.Quaternion(),
    };

    // Flight tuning
    let speed = 180;             // m/s 느낌의 임의 단위
    let strafe = 0;
    let adrenaline = 0;          // 0..1
    let score = 0;

    let damageFlash = 0;

    const forward = new THREE.Vector3();
    const right   = new THREE.Vector3();
    const up      = new THREE.Vector3(0, 1, 0);

    const clock = new THREE.Clock();

    function clamp(v, a, b) { return Math.max(a, Math.min(b, v)); }
    function lerp(a, b, t) { return a + (b - a) * t; }
    function expLerp(current, target, sharpness, dt) {
      const t = 1 - Math.exp(-sharpness * dt);
      return lerp(current, target, t);
    }

    function placeRing(ring) {
      // 현재 진행방향 기준으로 앞쪽에 배치
      forward.set(0,0,-1).applyQuaternion(player.quat).normalize();
      right.set(1,0,0).applyQuaternion(player.quat).normalize();

      const aheadDist = rand(450, 1100);
      const sideDist  = rand(-360, 360);
      const height    = rand(50, 380);

      const p = player.pos.clone()
        .add(forward.clone().multiplyScalar(aheadDist))
        .add(right.clone().multiplyScalar(sideDist));

      // 도시 범위 안으로 살짝 끌어오기
      p.x = clamp(p.x, -WORLD_HALF_X + 120, WORLD_HALF_X - 120);
      p.z = clamp(p.z, -WORLD_HALF_Z + 120, WORLD_HALF_Z - 120);
      p.y = height;

      ring.position.copy(p);
      ring.rotation.set(rand(0, Math.PI), rand(0, Math.PI), rand(0, Math.PI));
    }

    // 링 초기 재배치(처음부터 너무 가까운 걸 방지)
    for (const ring of rings) placeRing(ring);

    // ---------- Speed FX Canvas ----------
    const fx = document.getElementById('fx');
    const ctx = fx.getContext('2d');

    function resize() {
      const w = window.innerWidth;
      const h = window.innerHeight;

      renderer.setSize(w, h);
      camera.aspect = w / h;
      camera.updateProjectionMatrix();

      const dpr = Math.min(window.devicePixelRatio || 1, 2);
      fx.width  = Math.floor(w * dpr);
      fx.height = Math.floor(h * dpr);
      fx.style.width = w + "px";
      fx.style.height = h + "px";
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }
    window.addEventListener('resize', resize);
    resize();

    function drawSpeedLines(intensity, adr) {
      const w = window.innerWidth;
      const h = window.innerHeight;
      const cx = w * 0.5;
      const cy = h * 0.5;

      ctx.clearRect(0, 0, w, h);

      // 중앙 글로우(속도감 보정)
      const glow = ctx.createRadialGradient(cx, cy, 0, cx, cy, Math.max(w, h) * 0.6);
      glow.addColorStop(0, `rgba(255,255,255,${0.10 * intensity})`);
      glow.addColorStop(1, `rgba(255,255,255,0)`);
      ctx.fillStyle = glow;
      ctx.fillRect(0, 0, w, h);

      const count = Math.floor(18 + 180 * intensity);
      for (let i = 0; i < count; i++) {
        const a = Math.random() * Math.PI * 2;
        const r0 = (Math.random() ** 2) * 24;
        const len = (40 + 520 * intensity) * (0.45 + Math.random() * 0.85);

        const x0 = cx + Math.cos(a) * r0;
        const y0 = cy + Math.sin(a) * r0;
        const x1 = cx + Math.cos(a) * (r0 + len);
        const y1 = cy + Math.sin(a) * (r0 + len);

        const alpha = (0.04 + 0.18 * intensity) * (0.35 + 0.65 * Math.random());
        ctx.strokeStyle = `rgba(255,255,255,${alpha})`;
        ctx.lineWidth = 1 + 2.2 * intensity + 0.8 * adr;
        ctx.beginPath();
        ctx.moveTo(x0, y0);
        ctx.lineTo(x1, y1);
        ctx.stroke();
      }
    }

    // ---------- Building proximity / collision ----------
    function pointAABB_DistXZ(px, pz, b) {
      // XZ 평면 거리(건물 옆을 스칠 때 사용)
      let dx = 0;
      if (px < b.minX) dx = b.minX - px;
      else if (px > b.maxX) dx = px - b.maxX;

      let dz = 0;
      if (pz < b.minZ) dz = b.minZ - pz;
      else if (pz > b.maxZ) dz = pz - b.maxZ;

      return Math.sqrt(dx*dx + dz*dz);
    }

    function checkNearbyBuildings(dt) {
      const px = player.pos.x;
      const py = player.pos.y;
      const pz = player.pos.z;

      const pcx = Math.floor(px / cellSize);
      const pcz = Math.floor(pz / cellSize);

      const nearDist = 14.0; // “스치기” 기준(작을수록 더 스릴)
      let collided = false;

      // 주변 셀만 검사
      for (let ox = -1; ox <= 1; ox++) {
        for (let oz = -1; oz <= 1; oz++) {
          const key = (pcx + ox) + "|" + (pcz + oz);
          const arr = cellMap.get(key);
          if (!arr) continue;

          for (let i = 0; i < arr.length; i++) {
            const b = buildings[arr[i]];
            // 고도가 건물 높이보다 충분히 높으면 무시(최적화)
            if (py > b.maxY + 18) continue;

            // 충돌(점-박스 단순 판정)
            const insideXZ = (px >= b.minX && px <= b.maxX && pz >= b.minZ && pz <= b.maxZ);
            const insideY  = (py >= 1 && py <= b.maxY + 2);
            if (insideXZ && insideY) {
              // 충돌 반응: 위/뒤로 튕기고 감속
              collided = true;

              player.pos.y = Math.max(player.pos.y, b.maxY + 10);
              // 뒤로 약간 밀기
              forward.set(0,0,-1).applyQuaternion(player.quat).normalize();
              player.pos.addScaledVector(forward, -28);

              speed = Math.max(80, speed * 0.42);
              adrenaline = Math.max(0, adrenaline * 0.4);

              damageFlash = 1.0;
              break;
            }

            // 근접 스치기(아드레날린)
            const dist = pointAABB_DistXZ(px, pz, b);
            if (dist < nearDist && py < b.maxY + 10 && py > 6) {
              // 스칠수록 더 많이 오른다
              const gain = (1 - dist / nearDist);
              adrenaline = clamp(adrenaline + gain * dt * 1.05, 0, 1);
            }
          }
          if (collided) break;
        }
        if (collided) break;
      }

      // 바닥 충돌
      if (player.pos.y < 3.5) {
        player.pos.y = 3.5;
        speed = Math.max(60, speed * 0.85);
        damageFlash = Math.max(damageFlash, 0.6);
      }
    }

    // ---------- Main loop ----------
    const camPos = new THREE.Vector3();
    const camTarget = new THREE.Vector3();

    function animate() {
      const dt = Math.min(0.033, clock.getDelta());

      // 마우스 입력 소비(프레임마다 리셋)
      const dx = mouseDX; const dy = mouseDY;
      mouseDX = 0; mouseDY = 0;

      if (pointerLocked) {
        const sens = 0.0021;

        player.yaw   -= dx * sens;
        player.pitch -= dy * sens;

        player.pitch = clamp(player.pitch, -1.35, 1.35);
      }

      // 키 입력
      const accelInput = (keys['KeyW'] ? 1 : 0) - (keys['KeyS'] ? 1 : 0);
      const strafeInput = (keys['KeyD'] ? 1 : 0) - (keys['KeyA'] ? 1 : 0);
      const rollInput = (keys['KeyE'] ? 1 : 0) - (keys['KeyQ'] ? 1 : 0);
      const boostHeld = !!keys['ShiftLeft'] || !!keys['ShiftRight'];

      if (keys['KeyR']) {
        // 리셋
        player.pos.set(0, 140, 900);
        player.yaw = 0; player.pitch = 0; player.roll = 0;
        speed = 180;
        strafe = 0;
        adrenaline = 0;
        damageFlash = 0;
        keys['KeyR'] = false;
      }

      // 회전(롤)
      if (rollInput !== 0) {
        player.roll += rollInput * dt * 2.6;
      } else {
        // 스트레이프/턴에 약간 은행 자동 보정(속도감)
        const autoRollTarget = clamp(-strafeInput * 0.55 + (dx * 0.0012), -1.1, 1.1);
        player.roll = expLerp(player.roll, autoRollTarget, 4.0, dt);
      }
      player.roll = clamp(player.roll, -1.2, 1.2);

      // 쿼터니언 갱신
      player.quat.setFromEuler(new THREE.Euler(player.pitch, player.yaw, player.roll, 'YXZ'));

      // 방향 벡터
      forward.set(0,0,-1).applyQuaternion(player.quat).normalize();
      right.set(1,0,0).applyQuaternion(player.quat).normalize();

      // 속도 튜닝
      const baseMax = 520;
      const extraMax = 320;
      const maxSpeed = baseMax + adrenaline * extraMax;

      const baseAccel = 320;
      const accel = baseAccel * (1 + adrenaline * 0.7);

      // 다이브(하강) 시 가속, 상승 시 감속(활공 느낌)
      const diveBonus = Math.max(0, -forward.y) * (220 + 240 * adrenaline);
      const climbPenalty = Math.max(0, forward.y) * (240);

      speed += accelInput * accel * dt;
      speed += diveBonus * dt;
      speed -= climbPenalty * dt;

      // 기본 드래그(너무 무한가속 방지)
      speed -= speed * (0.018 + 0.018 * Math.abs(player.roll)) * dt;

      // 부스트: 아드레날린이 있을 때만 “폭발”
      if (boostHeld && adrenaline > 0.06) {
        speed += (420 + 520 * adrenaline) * dt;
        adrenaline = Math.max(0, adrenaline - dt * 0.28);
      }

      speed = clamp(speed, 60, maxSpeed);

      // 스트레이프(빌딩 사이 활공 보정)
      const strafeMax = 40 + speed * 0.28;
      const strafeTarget = strafeInput * strafeMax;
      strafe = expLerp(strafe, strafeTarget, 6.0, dt);

      // 이동
      player.pos.addScaledVector(forward, speed * dt);
      player.pos.addScaledVector(right, strafe * dt);

      // 도시 범위 밖으로 나가면 부드럽게 되돌림(벽 느낌)
      const margin = 220;
      if (player.pos.x < -WORLD_HALF_X + margin) { player.pos.x = -WORLD_HALF_X + margin; speed *= 0.92; }
      if (player.pos.x >  WORLD_HALF_X - margin) { player.pos.x =  WORLD_HALF_X - margin; speed *= 0.92; }
      if (player.pos.z < -WORLD_HALF_Z + margin) { player.pos.z = -WORLD_HALF_Z + margin; speed *= 0.92; }
      if (player.pos.z >  WORLD_HALF_Z - margin) { player.pos.z =  WORLD_HALF_Z - margin; speed *= 0.92; }

      // 아드레날린 자연 감소(너무 오래 유지 방지)
      adrenaline = Math.max(0, adrenaline - dt * 0.12);

      // 근접/충돌 체크
      checkNearbyBuildings(dt);

      // 링 체크
      for (const ring of rings) {
        const dist = ring.position.distanceTo(player.pos);
        if (dist < 12.5) {
          score++;
          adrenaline = clamp(adrenaline + 0.28, 0, 1);
          placeRing(ring);
        }

        // 링 약간 회전(가시성)
        ring.rotation.x += dt * 0.45;
        ring.rotation.y += dt * 0.35;
      }

      // 카메라: 속도에 따라 뒤로 빠지고 FOV 확장
      const speed01 = clamp((speed - 60) / 520, 0, 1);
      const baseFov = 78;
      const targetFov = baseFov + 28 * speed01 + 12 * adrenaline;
      camera.fov = expLerp(camera.fov, targetFov, 6.0, dt);
      camera.updateProjectionMatrix();

      const camBack = 18 + speed01 * 22;
      const camUp = 6 + adrenaline * 3.5;

      // 플레이어 뒤/위쪽으로 카메라 오프셋
      const camOffset = new THREE.Vector3(0, camUp, camBack).applyQuaternion(player.quat);
      const desiredCam = player.pos.clone().add(camOffset);

      // 카메라 스프링 스무딩
      camPos.copy(camera.position);
      camPos.lerp(desiredCam, 1 - Math.exp(-dt * 4.5));

      // 약간의 흔들림(고속)
      const shake = (0.05 + 0.25 * speed01 + 0.28 * adrenaline);
      camPos.x += (Math.random() - 0.5) * shake;
      camPos.y += (Math.random() - 0.5) * shake;
      camPos.z += (Math.random() - 0.5) * shake;

      camera.position.copy(camPos);

      camTarget.copy(player.pos).add(forward.clone().multiplyScalar(42 + speed01 * 20));
      camera.lookAt(camTarget);

      // 데미지 플래시
      damageFlash = Math.max(0, damageFlash - dt * 1.8);
      damageEl.style.background = `rgba(255,0,0,${0.22 * damageFlash})`;

      // UI 업데이트
      statusEl.textContent = pointerLocked ? (speed > 520 ? "SONIC" : "FLY") : "클릭해서 시작";
      speedEl.textContent = Math.round(speed);
      altEl.textContent = Math.round(player.pos.y);
      scoreEl.textContent = score;

      adrTxtEl.textContent = Math.round(adrenaline * 100) + "%";
      adrBarEl.style.width = (adrenaline * 100).toFixed(1) + "%";

      // FX: 속도선
      const fxIntensity = clamp(speed01 * 0.95 + adrenaline * 0.55, 0, 1);
      drawSpeedLines(fxIntensity, adrenaline);

      renderer.render(scene, camera);
      requestAnimationFrame(animate);
    }

    animate();
  })();
  </script>
</body>
</html>

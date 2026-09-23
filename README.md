<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Poxel.io Clone - Retro FPS</title>
    <style>
        body, html { margin: 0; padding: 0; overflow: hidden; width: 100%; height: 100%; font-family: 'Courier New', monospace; user-select: none; background: #000; }
        #canvas-container { width: 100%; height: 100%; display: block; }
        
        /* UI Overlay */
        #crosshair { position: absolute; top: 50%; left: 50%; width: 10px; height: 10px; transform: translate(-50%, -50%); pointer-events: none; }
        #crosshair::before, #crosshair::after { content: ''; position: absolute; background: #00ffcc; }
        #crosshair::before { top: 4px; left: -5px; width: 20px; height: 2px; }
        #crosshair::after { top: -5px; left: 4px; width: 2px; height: 20px; }

        #ui-layer { position: absolute; top: 20px; left: 20px; color: #00ffcc; font-size: 18px; text-shadow: 2px 2px #000; pointer-events: none; }
        #instructions { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); color: #fff; text-align: center; background: rgba(0,0,0,0.85); padding: 30px; border: 3px solid #00ffcc; border-radius: 10px; cursor: pointer; }
        #instructions h1 { margin-top: 0; color: #00ffcc; }
        
        .stat-box { margin-bottom: 5px; background: rgba(0,0,0,0.5); padding: 5px 10px; border-left: 3px solid #00ffcc; }
    </style>
    <!-- Include Three.js via CDN -->
    <script src="https://cloudflare.com"></script>
</head>
<body>

    <div id="canvas-container"></div>
    <div id="crosshair"></div>
    
    <div id="ui-layer">
        <div class="stat-box">KILLS: <span id="kill-count">0</span></div>
        <div class="stat-box">SPEED: <span id="speed-meter">0</span> u/s</div>
        <div class="stat-box" style="border-left-color: #ff3366;">HEALTH: 100</div>
    </div>

    <div id="instructions">
        <h1>POXEL.IO ECO-CLONE</h1>
        <p>CLICK TO PLAY</p>
        <p style="font-size: 14px; color: #aaa;">WASD = Move | SPACE = Jump / B-Hop<br>SHIFT = Slide Tech | CLICK = Shoot</p>
    </div>

    <script>
        // --- GAME STATE & CONFIG ---
        let scene, camera, renderer, clock;
        let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false, isGrounded = true, isSliding = false;
        
        // Physics variables mimicking Poxel.io high-speed tech
        const velocity = new THREE.Vector3();
        const direction = new THREE.Vector3();
        let speedMultiplier = 1.0;
        let bHopTimer = 0;
        let score = 0;

        // Pointer Lock Controls setup
        const instructions = document.getElementById('instructions');
        const speedMeter = document.getElementById('speed-meter');
        const killCount = document.getElementById('kill-count');
        let isLocked = false;

        const pitchObject = new THREE.Object3D();
        const yawObject = new THREE.Object3D();

        // Game Objects Collections
        const targets = [];
        const activeTracers = [];

        init();
        animate();

        function init() {
            // 1. Scene & Setup
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x1a1a2e);
            scene.fog = new THREE.FogExp2(0x1a1a2e, 0.015);

            clock = new THREE.Clock();

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            
            // Camera hierarchy for FPS tracking
            pitchObject.add(camera);
            yawObject.position.y = 2; // Player height
            yawObject.add(pitchObject);
            scene.add(yawObject);

            // 2. Lighting (Retro-Stylised Cyber Environment)
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
            scene.add(ambientLight);

            const dirLight = new THREE.DirectionalLight(0x00ffcc, 0.8);
            dirLight.position.set(20, 40, 20);
            scene.add(dirLight);

            // 3. Environment: Pixel-style Map Grid
            const rendererContainer = document.getElementById('canvas-container');
            renderer = new THREE.WebGLRenderer({ antialias: false }); // Crisp pixel edges
            renderer.setSize(window.innerWidth, window.innerHeight);
            rendererContainer.appendChild(renderer.domElement);

            // Floor Grid with 3D Voxel Pillars
            const floorGeo = new THREE.PlaneGeometry(200, 200, 50, 50);
            const floorMat = new THREE.MeshStandardMaterial({ color: 0x111122, wireframe: true });
            const floor = new THREE.Mesh(floorGeo, floorMat);
            floor.rotation.x = -Math.PI / 2;
            scene.add(floor);

            // Generate Map Geometry (Obstacles/Pillars for movement surfing)
            const boxGeo = new THREE.BoxGeometry(4, 12, 4);
            const boxMat = new THREE.MeshStandardMaterial({ color: 0x334466, roughness: 0.8 });
            
            for(let i = 0; i < 40; i++) {
                const obstacle = new THREE.Mesh(boxGeo, boxMat);
                obstacle.position.set(
                    (Math.random() - 0.5) * 160,
                    6,
                    (Math.random() - 0.5) * 160
                );
                scene.add(obstacle);
            }

            // Spawn Initial Bot Targets
            for(let i = 0; i < 8; i++) spawnTarget();

            // 4. Input & Event Listeners
            setupControls();
            window.addEventListener('resize', onWindowResize);
        }

        // --- CONTROLS & POINTER LOCK ---
        function setupControls() {
            instructions.addEventListener('click', () => {
                document.body.requestPointerLock();
            });

            document.addEventListener('pointerlockchange', () => {
                if (document.pointerLockElement === document.body) {
                    instructions.style.display = 'none';
                    isLocked = true;
                } else {
                    instructions.style.display = 'block';
                    isLocked = false;
                }
            });

            document.addEventListener('mousemove', (e) => {
                if (!isLocked) return;
                yawObject.rotation.y -= e.movementX * 0.0022;
                pitchObject.rotation.x -= e.movementY * 0.0022;
                pitchObject.rotation.x = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, pitchObject.rotation.x));
            });

            document.addEventListener('keydown', (e) => {
                switch (e.code) {
                    case 'KeyW': moveForward = true; break;
                    case 'KeyS': moveBackward = true; break;
                    case 'KeyA': moveLeft = true; break;
                    case 'KeyD': moveRight = true; break;
                    case 'Space': 
                        if (isGrounded) {
                            // Bunny-hop boost if timed correctly
                            let jumpBoost = (performance.now() - bHopTimer < 300) ? 1.3 : 1.0;
                            velocity.y = 12; 
                            velocity.x *= jumpBoost;
                            velocity.z *= jumpBoost;
                            isGrounded = false;
                        }
                        break;
                    case 'ShiftLeft':
                        if(!isSliding && isGrounded) {
                            isSliding = true;
                            yawObject.position.y = 1.0; // Crouch/slide down camera
                            speedMultiplier = 1.8;
                            setTimeout(() => { // End slide boost
                                isSliding = false;
                                yawObject.position.y = 2;
                                speedMultiplier = 1.0;
                            }, 400);
                        }
                        break;
                }
            });

            document.addEventListener('keyup', (e) => {
                switch (e.code) {
                    case 'KeyW': moveForward = false; break;
                    case 'KeyS': moveBackward = false; break;
                    case 'KeyA': moveLeft = false; break;
                    case 'KeyD': moveRight = false; break;
                }
            });

            document.addEventListener('mousedown', (e) => {
                if(isLocked && e.button === 0) fireWeapon();
            });
        }

        // --- CORE SHOOTING MECHANICS ---
        function fireWeapon() {
            // Weapon Kickback/Recoil (Alters pitch slightly)
            pitchObject.rotation.x += 0.02;

            // Simple Hitscan Raycasting
            const raycaster = new THREE.Raycaster();
            const center = new THREE.Vector2(0, 0); // Crosshair center
            raycaster.setFromCamera(center, camera);

            const intersects = raycaster.intersectObjects(targets);

            // Draw Hitscan Tracer Line
            const origin = new THREE.Vector3().setFromMatrixPosition(camera.matrixWorld);
            origin.y -= 0.2; // Offset to mimic weapon muzzle
            let endpoint = new THREE.Vector3();

            if (intersects.length > 0) {
                const hitObj = intersects[0].object;
                endpoint.copy(intersects[0].point);
                
                // Process Damage/Kill
                scene.remove(hitObj);
                targets.splice(targets.indexOf(hitObj), 1);
                score++;
                killCount.innerText = score;
                
                // Instantly respawn new bot elsewhere
                spawnTarget();
            } else {
                raycaster.ray.at(100, endpoint);
            }

            createTracer(origin, endpoint);
        }

        function createTracer(start, end) {
Use code with caution.const material = new THREE.LineBasicMaterial({ color: 0x00ffcc });const points = [start, end];const geometry = new THREE.BufferGeometry().setFromPoints(points);const line = new THREE.Line(geometry, material);scene.add(line);activeTracers.push({ mesh: line, spawnTime: performance.now() });}function spawnTarget() {// Box/Pixel style target botsconst targetGeo = new THREE.BoxGeometry(2, 4, 2);const targetMat = new THREE.MeshStandardMaterial({ color: 0xff3366, emissive: 0x441111 });const target = new THREE.Mesh(targetGeo, targetMat);target.position.set((Math.random() - 0.5) * 150,2,(Math.random() - 0.5) * 150);scene.add(target);targets.push(target);}function onWindowResize() {camera.aspect = window.innerWidth / window.innerHeight;camera.updateProjectionMatrix();renderer.setSize(window.innerWidth, window.innerHeight);}// --- GAME LOOP & PHYSICS ENGINE ---function animate() {requestAnimationFrame(animate);if (isLocked) {const delta = clock.getDelta();// Apply friction coefficients based on statelet friction = isGrounded ? (isSliding ? 1.5 : 8.0) : 1.5;velocity.x -= velocity.x * friction * delta;velocity.z -= velocity.z * friction * delta;// Gravity implementationvelocity.y -= 32.0 * delta;direction.z = Number(moveForward) - Number(moveBackward);direction.x = Number(moveRight) - Number(moveLeft);direction.normalize();// Base movement speedsconst currentMoveSpeed = 45.0 * speedMultiplier;if (moveForward || moveBackward) velocity.z -= direction.z * currentMoveSpeed * delta;if (moveLeft || moveRight) velocity.x -= direction.x * currentMoveSpeed * delta;// Apply vectors locally based on player viewpoint orientationyawObject.translateX(-velocity.x * delta);yawObject.translateZ(-velocity.z * delta);yawObject.position.y += velocity.y * delta;// Floor Boundaries Collisionif (yawObject.position.y < 2) {velocity.y = 0;yawObject.position.y = isSliding ? 1.0 : 2;if(!isGrounded) {isGrounded = true;bHopTimer = performance.now(); // Track landing window for B-hopping}}// Render dynamic speedometer tracking UIconst currentSpeedStr = Math.round(Math.sqrt(velocity.x2 + velocity.z2));speedMeter.innerText = currentSpeedStr;// Clean up decaying laser shooting tracersconst now = performance.now();for(let i = activeTracers.length - 1; i >= 0; i--) {if(now - activeTracers[i].spawnTime > 80) {scene.remove(activeTracers[i].mesh);activeTracers.splice(i, 1);}}}renderer.render(scene, camera);}

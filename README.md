<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Three.js 正常版 - 跑车与正常城市</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; }
        canvas { display: block; }
        #ui {
            position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%);
            color: white; font-family: 'Arial', sans-serif; font-size: 16px;
            background: rgba(0, 0, 0, 0.6); padding: 10px 20px; border-radius: 8px;
            pointer-events: none; text-align: center; transition: opacity 0.3s;
        }
        #interaction {
            position: absolute; top: 55%; left: 50%; transform: translate(-50%, -50%);
            color: #ffcc00; font-family: 'Arial', sans-serif; font-size: 24px; font-weight: bold;
            text-shadow: 2px 2px 4px #000; display: none; pointer-events: none;
        }
        #loading {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: white; font-family: 'Arial', sans-serif; font-size: 30px; font-weight: bold;
        }
    </style>
</head>
<body>
    <div id="loading">正在加载...</div>
    <div id="ui">点击画面锁定鼠标 | WASD 移动 | 靠近超跑按 <b>F</b> 驾驶</div>
    <div id="interaction">按 F 上车</div>

    <script type="importmap">
    {
        "imports": {
            "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
            "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
        }
    }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
        import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
        import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
        import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

        // ==========================================
        // 1. 基础场景
        // ==========================================
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87CEEB); 
        // 去掉指数雾，改用线性雾，防止物体发黑
        scene.fog = new THREE.Fog(0x87CEEB, 100, 500);

        const camera = new THREE.PerspectiveCamera(65, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); 
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        document.body.appendChild(renderer.domElement);

        // 加强环境光，确保深色物体也能看清
        const ambientLight = new THREE.AmbientLight(0xffffff, 1.2);
        scene.add(ambientLight);
        const dirLight = new THREE.DirectionalLight(0xfff5e6, 2.0);
        dirLight.position.set(100, 150, 100);
        dirLight.castShadow = true;
        dirLight.shadow.mapSize.width = 2048;
        dirLight.shadow.mapSize.height = 2048;
        dirLight.shadow.camera.left = -100;
        dirLight.shadow.camera.right = 100;
        dirLight.shadow.camera.top = 100;
        dirLight.shadow.camera.bottom = -100;
        dirLight.shadow.camera.far = 400;
        scene.add(dirLight);
        scene.add(dirLight.target);

        // ==========================================
        // 2. 地面与道路
        // ==========================================
        // 草地
        const grassCanvas = document.createElement('canvas');
        grassCanvas.width = 256; grassCanvas.height = 256;
        const gCtx = grassCanvas.getContext('2d');
        gCtx.fillStyle = '#4ca62b'; gCtx.fillRect(0, 0, 256, 256);
        for(let i=0; i<2000; i++) {
            gCtx.fillStyle = Math.random() > 0.5 ? 'rgba(60, 140, 30, 0.4)' : 'rgba(100, 200, 60, 0.4)';
            gCtx.fillRect(Math.random() * 256, Math.random() * 256, Math.random() * 2 + 1, Math.random() * 2 + 1);
        }
        const grassTex = new THREE.CanvasTexture(grassCanvas);
        grassTex.wrapS = THREE.RepeatWrapping; grassTex.wrapT = THREE.RepeatWrapping;
        grassTex.repeat.set(50, 50);

        const ground = new THREE.Mesh(
            new THREE.PlaneGeometry(1000, 1000),
            new THREE.MeshStandardMaterial({ map: grassTex, roughness: 1 })
        );
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);

        // 道路 (简化版，纯色带虚线)
        const roadMat = new THREE.MeshStandardMaterial({ color: 0x333333, roughness: 0.9 });
        const lineMat = new THREE.MeshBasicMaterial({ color: 0xffffff });

        // 南北向
        const roadNS = new THREE.Mesh(new THREE.PlaneGeometry(16, 1000), roadMat);
        roadNS.rotation.x = -Math.PI / 2;
        roadNS.position.set(0, 0.05, 0);
        roadNS.receiveShadow = true;
        scene.add(roadNS);

        // 东西向
        const roadEW = new THREE.Mesh(new THREE.PlaneGeometry(1000, 16), roadMat);
        roadEW.rotation.x = -Math.PI / 2;
        roadEW.position.set(0, 0.05, 0);
        roadEW.receiveShadow = true;
        scene.add(roadEW);

        // 中心线 (简化版)
        for(let i=-400; i<400; i+=20) {
            const line1 = new THREE.Mesh(new THREE.PlaneGeometry(0.5, 8), lineMat);
            line1.rotation.x = -Math.PI / 2; line1.position.set(0, 0.06, i);
            scene.add(line1);
            const line2 = new THREE.Mesh(new THREE.PlaneGeometry(8, 0.5), lineMat);
            line2.rotation.x = -Math.PI / 2; line2.position.set(i, 0.06, 0);
            scene.add(line2);
        }

        // ==========================================
        // 3. 简单建筑 (放弃 InstancedMesh 复杂着色)
        // ==========================================
        const buildingMat = new THREE.MeshStandardMaterial({ color: 0x8899aa, roughness: 0.8 });
        const windowMat = new THREE.MeshBasicMaterial({ color: 0x223344 });

        for (let i = 0; i < 100; i++) {
            const x = (Math.random() - 0.5) * 800;
            const z = (Math.random() - 0.5) * 800;
            
            // 避开中心十字路口
            if (Math.abs(x) < 20 || Math.abs(z) < 20) continue;

            const w = 15 + Math.random() * 10;
            const h = 20 + Math.random() * 40;
            const d = 15 + Math.random() * 10;

            const building = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), buildingMat);
            building.position.set(x, h / 2, z);
            building.castShadow = true; building.receiveShadow = true;
            scene.add(building);

            // 简单加个窗户条
            const win = new THREE.Mesh(new THREE.BoxGeometry(w + 0.1, h * 0.8, d + 0.1), windowMat);
            win.position.set(x, h / 2, z);
            scene.add(win);
        }

        // ==========================================
        // 4. 正常比例的行人
        // ==========================================
        const npcGeo = new THREE.CapsuleGeometry(0.3, 0.8, 4, 8);
        const npcMat = new THREE.MeshStandardMaterial({ color: 0xffaa00 });
        
        for (let i = 0; i < 50; i++) {
            const npc = new THREE.Mesh(npcGeo, npcMat);
            // 让人走在路边
            npc.position.set((Math.random() - 0.5) * 800, 0.7, (Math.random() - 0.5) * 800);
            npc.castShadow = true;
            scene.add(npc);
        }

        // ==========================================
        // 5. 主角与蓝色跑车 (绝对立体的模型)
        // ==========================================
        const playerGroup = new THREE.Group();
        const body = new THREE.Mesh(new THREE.CapsuleGeometry(0.4, 1.0, 4, 8), new THREE.MeshStandardMaterial({ color: 0x222222 }));
        body.position.y = 0.9; body.castShadow = true;
        playerGroup.add(body);

        const head = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.8, 0.8), new THREE.MeshStandardMaterial({ color: 0xe0c0a0 }));
        head.position.y = 1.8; head.castShadow = true;
        playerGroup.add(head);
        
        playerGroup.position.set(-5, 0, -5); 
        scene.add(playerGroup);

        // --- 跑车 ---
        const carGroup = new THREE.Group();
        // 鲜艳的蓝色车漆，确保能看见
        const carBodyMat = new THREE.MeshStandardMaterial({ color: 0x0066ff, metalness: 0.6, roughness: 0.2 });
        const glassMat = new THREE.MeshStandardMaterial({ color: 0x111111, metalness: 0.9, roughness: 0.1 });
        const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 0.9 });

        // 1. 车身
        const chassis = new THREE.Mesh(new THREE.BoxGeometry(2.2, 0.6, 4.5), carBodyMat);
        chassis.position.y = 0.6; chassis.castShadow = true;
        carGroup.add(chassis);

        // 2. 车顶/挡风玻璃
        const cabin = new THREE.Mesh(new THREE.BoxGeometry(1.8, 0.5, 2.0), glassMat);
        cabin.position.set(0, 1.1, 0); cabin.castShadow = true;
        carGroup.add(cabin);

        // 3. 车轮
        const wheelGeo = new THREE.CylinderGeometry(0.4, 0.4, 0.3, 16);
        const wPositions = [{x:-1.2, z:1.5}, {x:1.2, z:1.5}, {x:-1.2, z:-1.5}, {x:1.2, z:-1.5}];
        wPositions.forEach(pos => {
            const w = new THREE.Mesh(wheelGeo, wheelMat);
            w.rotation.z = Math.PI / 2;
            w.position.set(pos.x, 0.4, pos.z);
            w.castShadow = true;
            carGroup.add(w);
        });

        // 把车放在主角旁边
        carGroup.position.set(0, 0, 0); 
        scene.add(carGroup);

        document.getElementById('loading').style.display = 'none';

        // ==========================================
        // 6. 控制逻辑 (保持原样)
        // ==========================================
        let cameraAngleX = 0, cameraAngleY = 0.2;
        let isDriving = false;
        const keys = {};
        const carState = { speed: 0, heading: 0, maxSpeed: 40, acceleration: 20, friction: 5 };
        
        renderer.domElement.addEventListener('click', () => {
            renderer.domElement.requestPointerLock();
            document.getElementById('ui').style.opacity = '0';
        });

        document.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement === renderer.domElement) {
                cameraAngleX -= e.movementX * 0.002;
                cameraAngleY -= e.movementY * 0.002;
                cameraAngleY = Math.max(-0.2, Math.min(1.0, cameraAngleY));
            }
        });

        window.addEventListener('keydown', (e) => {
            keys[e.code] = true;
            if (e.code === 'KeyF') {
                if (!isDriving) {
                    if (playerGroup.position.distanceTo(carGroup.position) < 8) {
                        isDriving = true; playerGroup.visible = false; carState.speed = 0;
                        document.getElementById('interaction').style.display = 'none';
                    }
                } else {
                    isDriving = false; playerGroup.visible = true;
                    playerGroup.position.copy(carGroup.position);
                    playerGroup.position.x += Math.cos(carState.heading) * 4;
                    playerGroup.position.z -= Math.sin(carState.heading) * 4;
                }
            }
        });
        window.addEventListener('keyup', (e) => keys[e.code] = false);

        // ==========================================
        // 7. 主循环
        // ==========================================
        const composer = new EffectComposer(renderer);
        composer.addPass(new RenderPass(scene, camera));
        // 泛光调低，防止画面发白
        composer.addPass(new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 0.1, 0.4, 0.85)); 
        composer.addPass(new OutputPass());

        const clock = new THREE.Clock();

        function animate() {
            requestAnimationFrame(animate);
            const delta = Math.min(clock.getDelta(), 0.1);

            if (isDriving) {
                if (keys['KeyW']) carState.speed += carState.acceleration * delta;
                if (keys['KeyS']) carState.speed -= carState.acceleration * delta;
                if (!keys['KeyW'] && !keys['KeyS']) {
                    if (carState.speed > 0) carState.speed = Math.max(0, carState.speed - carState.friction * delta);
                    if (carState.speed < 0) carState.speed = Math.min(0, carState.speed + carState.friction * delta);
                }
                carState.speed = Math.max(-10, Math.min(carState.maxSpeed, carState.speed));
                
                const turnRate = 2.0 - (Math.abs(carState.speed) / carState.maxSpeed) * 1.2;
                if (keys['KeyA']) carState.heading += turnRate * delta * Math.sign(carState.speed);
                if (keys['KeyD']) carState.heading -= turnRate * delta * Math.sign(carState.speed);

                carGroup.position.x += Math.sin(carState.heading) * carState.speed * delta;
                carGroup.position.z += Math.cos(carState.heading) * carState.speed * delta;
                carGroup.rotation.y = carState.heading;

                const distance = 10;
                camera.position.set(
                    carGroup.position.x - Math.sin(carState.heading) * distance, 
                    carGroup.position.y + 4, 
                    carGroup.position.z - Math.cos(carState.heading) * distance
                );
                camera.lookAt(
                    carGroup.position.x + Math.sin(carState.heading) * 5, 
                    carGroup.position.y + 1, 
                    carGroup.position.z + Math.cos(carState.heading) * 5
                );
            } else {
                const speed = 12 * delta;
                let moveX = 0, moveZ = 0;
                if (keys['KeyW']) moveZ -= 1;
                if (keys['KeyS']) moveZ += 1;
                if (keys['KeyA']) moveX -= 1;
                if (keys['KeyD']) moveX += 1;

                if (moveX !== 0 || moveZ !== 0) {
                    const angle = Math.atan2(moveX, moveZ);
                    const worldAngle = angle + cameraAngleX;
                    playerGroup.position.x += Math.sin(worldAngle) * speed;
                    playerGroup.position.z += Math.cos(worldAngle) * speed;
                    playerGroup.rotation.y = worldAngle + Math.PI;
                }

                const offsetX = Math.sin(cameraAngleX) * Math.cos(cameraAngleY) * 8;
                const offsetZ = Math.cos(cameraAngleX) * Math.cos(cameraAngleY) * 8;
                const offsetY = Math.sin(cameraAngleY) * 8;
                camera.position.set(playerGroup.position.x + offsetX, playerGroup.position.y + offsetY + 1.5, playerGroup.position.z + offsetZ);
                camera.lookAt(playerGroup.position.x, playerGroup.position.y + 1.0, playerGroup.position.z);

                const ui = document.getElementById('interaction');
                if (!isDriving && playerGroup.position.distanceTo(carGroup.position) < 8) {
                    ui.style.display = 'block';
                } else {
                    ui.style.display = 'none';
                }
            }

            const target = isDriving ? carGroup.position : playerGroup.position;
            dirLight.position.set(target.x + 50, target.y + 100, target.z + 50);
            dirLight.target.position.copy(target);

            composer.render();
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
            composer.setSize(window.innerWidth, window.innerHeight);
        });

        animate();
    </script>
</body>
</html>
185.199.108.153 yfgame.github.io,sudo killall -HUP mDNSResponder

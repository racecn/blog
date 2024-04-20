<script lang="ts">
    import { ActionIcon, IconRenderer, Input, InputWrapper } from '@svelteuidev/core';
    import { RangeSlider } from '@skeletonlabs/skeleton';
    import { GithubLogo } from 'radix-icons-svelte';
    import { Button } from '@svelteuidev/core';
    import { browser } from '$app/environment'; 
    import { onMount } from 'svelte';

    import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
    import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
    import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
    import * as THREE from "three"

    let avgFps = 0;
    let fpsCount = 0;
    let fpsTotal = 0;
    let lastTime = performance.now();
    let isMouseLocked = false;
    let isPaused = false;
    let speed = 1;

    let planetDistance = 1;

    let maxPlanetDistance = 50;
    let minPlanetDistance = 0.1;

    let planets: THREE.Mesh<THREE.SphereGeometry, THREE.MeshBasicMaterial>[] = [];
    let planetSpeeds: number[] = [];
    if (browser) {
    let camera : THREE.PerspectiveCamera;
    let scene : THREE.Scene;
    let renderer : THREE.WebGLRenderer;
    let sun : THREE.Mesh<THREE.SphereGeometry, THREE.MeshBasicMaterial>;

    let moveForward = false;
    let moveBackward = false;
    let moveLeft = false;
    let moveRight = false;
    let moveSpeed = 0.005;

    let isWireframe = false;

    const fpsElement = document.getElementById('fps');
    const playPauseButton = document.getElementById('play-pause');
    const speedSlider = document.getElementById('speed-slider');

    onMount(() => {
        if (typeof document !== 'undefined') {
                document.getElementById('addPlanetForm').addEventListener('submit', function (event) {
                event.preventDefault(); // Prevent form submission

                console.log('Form submitted');

                const formData = new FormData(event.target);
                console.log('Form data:', formData);

                const newPlanet = {
                    name: formData.get('planetName'),
                    distance: parseFloat(formData.get('planetDistance')),
                    radius: parseFloat(formData.get('planetRadius')),
                    speed: parseFloat(formData.get('planetSpeed'))
                };
                console.log('New planet:', newPlanet);

                planetData.push(newPlanet); // Update planet data array
                addPlanet(newPlanet); // Add the new planet to the scene
            });

        }
    });

    const deletePlanet = (index: number) => {
        planetData.splice(index, 1); // Remove the planet at the specified index
        updatePlanets(); // Update the planets in the scene
        updatePlanetList(); // Update the list of planets rendered to the user
    };


    const sunRadius = 1; // Set the sun's radius to scale
    const planetData = [
            { name: 'Mercury', distance: 0.39 + sunRadius, radius: 0.03, speed: 0.005 },
            { name: 'Venus', distance: 0.72 + sunRadius, radius: 0.08, speed: 0.004 },
            { name: 'Earth', distance: 1 + sunRadius, radius: 0.08, speed: 0.003 },
            { name: 'Mars', distance: 1.52 + sunRadius, radius: 0.04, speed: 0.002 },
            { name: 'Jupiter', distance: 5.2 + sunRadius, radius: 0.4, speed: 0.001 },
            { name: 'Saturn', distance: 9.58 + sunRadius, radius: 0.35, speed: 0.0005 },
            { name: 'Uranus', distance: 19.22 + sunRadius, radius: 0.15, speed: 0.0003 },
            { name: 'Neptune', distance: 30.05 + sunRadius, radius: 0.15, speed: 0.0002 },
        ];

    const lockMouse = () => {
        const canvas = renderer.domElement;
        canvas.requestPointerLock();
    };

    const unlockMouse = () => {
        document.exitPointerLock();
    };

    const onPointerLockChange = () => {
        const canvas = renderer.domElement;
        isMouseLocked = document.pointerLockElement === canvas;
    };

    function handleToggleWireframe() {
        isWireframe = !isWireframe;
        toggleWireframeAll(planets);
        toggleWireframeAll([sun]);
    }

    function toggleWireframeAll(objects: THREE.Mesh[]) {
        objects.forEach(object => {
            const material = object.material as THREE.Material;
            material.wireframe = !material.wireframe;
        });
    }

    const onMouseMove = (event: MouseEvent) => {
    if (isMouseLocked) {
        const movementX = event.movementX || 0;
        const movementY = event.movementY || 0;

        // Calculate the current direction the camera is facing
        const dir = new THREE.Vector3(0, 0, -1);
        dir.applyQuaternion(camera.quaternion);

        // Calculate the new direction by rotating the current direction
        const axis = new THREE.Vector3(0, 1, 0); // Y-axis is up
        const angle = -movementX * 0.002; // Negative because moving left should turn camera left
        dir.applyAxisAngle(axis, angle);

        // Set the camera's rotation to look at the new direction
        camera.lookAt(camera.position.clone().add(dir));

        // Limit the camera's vertical rotation to between -PI/2 and PI/2
        camera.rotation.x = clamp(
            camera.rotation.x - movementY * 0.002,
            -Math.PI / 2,
            Math.PI / 2
        );
        
        // Wrap the camera's horizontal rotation around 0 to 2*PI
        camera.rotation.y = wrap(camera.rotation.y, -Math.PI, Math.PI);

    }
};

// Clamp function to keep a value within a range
const clamp = (value: number, min: number, max: number) => {
    return Math.min(Math.max(value, min), max);
};

// Wrap function to wrap a value around a range
const wrap = (value: number, min: number, max: number) => {
    const range = max - min;
    const wrapped = ((value - min) % range + range) % range + min;
    return wrapped >= max ? wrapped - range : wrapped;
};


const moveSpeedNormal = 0.005;
const moveSpeedFast = 0.05;


const onKeyDown = (event: KeyboardEvent) => {
    switch (event.code) {
        case 'ShiftLeft':
            moveSpeed = moveSpeedFast;
            break;
        case 'KeyW':
            moveBackward = true;
            break;
        case 'KeyS':
            moveForward = true;
            break;
        case 'KeyA':
            moveLeft = true;
            break;
        case 'KeyD':
            moveRight = true;
            break;
        case 'Space':
            // Move camera up
            camera.position.y += moveSpeed;
            break;
        case 'ControlLeft':
            // Move camera down
            camera.position.y -= moveSpeed;
            break;
    }
};

const onKeyUp = (event: KeyboardEvent) => {
    switch (event.code) {
        case 'ShiftLeft':
            moveSpeed = moveSpeedNormal;
            break;
        case 'KeyW':
            moveBackward = false;
            break;
        case 'KeyS':
            moveForward = false;
            break;
        case 'KeyA':
            moveLeft = false;
            break;
        case 'KeyD':
            moveRight = false;
            break;
    }
};
    const togglePause = () => {
        isPaused = !isPaused;
        playPauseButton.innerText = isPaused ? 'Play' : 'Pause';
    };

    const onSpeedChange = () => {
        speed = parseFloat(speedSlider.value);
    };

    document.addEventListener('mousemove', onMouseMove);
    document.addEventListener('keydown', onKeyDown);
    document.addEventListener('keyup', onKeyUp);
    document.addEventListener('pointerlockchange', onPointerLockChange);
    playPauseButton.addEventListener('click', togglePause);
    speedSlider.addEventListener('input', onSpeedChange);

    const updateCameraPosition = () => {
        const direction = new THREE.Vector3(0, 0, -1);
        direction.applyQuaternion(camera.quaternion);
        direction.normalize();

        if (moveForward) {
            camera.position.add(direction.multiplyScalar(-moveSpeed));
        }
        if (moveBackward) {
            camera.position.add(direction.multiplyScalar(moveSpeed));
        }

        const right = new THREE.Vector3(1, 0, 0);
        right.applyQuaternion(camera.quaternion);
        right.normalize();

        if (moveLeft) {
            camera.position.add(right.multiplyScalar(-moveSpeed));
        }
        if (moveRight) {
            camera.position.add(right.multiplyScalar(moveSpeed));
        }
    };

    
    const initPlanets = () => {
        planetData.forEach(data => {
            const color = new THREE.Color(Math.random(), Math.random(), Math.random());
            const geometry = new THREE.SphereGeometry(data.radius, 32, 32);
            const material = new THREE.MeshBasicMaterial({ color: color });
            const planet = new THREE.Mesh(geometry, material);
            planet.position.set(data.distance, 0, 0);
            planet.userData = data; // Add user data to the planet mesh
            planets.push(planet);
            planetSpeeds.push(data.speed);
            scene.add(planet);
        });
        };


    const updatePlanets = () => {
        planets.forEach((planet, index) => {
            const orbitRadius = planets[index].position.distanceTo(new THREE.Vector3(0, 0, 0));
            const theta = performance.now() * planetSpeeds[index] * speed / orbitRadius; // Calculate angle based on speed and radius
            planet.position.x = Math.cos(theta) * orbitRadius;
            planet.position.z = Math.sin(theta) * orbitRadius;
        });
    };

        const initOrbitLines = () => {
        planets.forEach((planet, index) => {
            const orbitRadius = planets[index].position.distanceTo(new THREE.Vector3(0, 0, 0));
            const orbitLineMaterial = new THREE.LineBasicMaterial({ color: planet.material.color });
            const orbitLineGeometry = new THREE.TorusGeometry(orbitRadius, 0.001, 16, 100);
            const orbitLine = new THREE.Line(orbitLineGeometry, orbitLineMaterial);
            orbitLine.rotation.x = Math.PI / 2; // Rotate the orbit line to be in the x-y plane
            scene.add(orbitLine);
        });
    };


    // Add a new planet to the scene
    const addPlanet = (newPlanetData) => {
        // Add the new planet to the scene without reinitializing all planets
        const color = new THREE.Color(Math.random(), Math.random(), Math.random()); // Random color
        const geometry = new THREE.SphereGeometry(newPlanetData.radius, 32, 32);
        const material = new THREE.MeshBasicMaterial({ color: color });
        const planet = new THREE.Mesh(geometry, material);
        planet.position.set(newPlanetData.distance, 0, 0);
        planets.push(planet);
        planetSpeeds.push(newPlanetData.speed);
        scene.add(planet);

        // Add orbit line for the new planet
        const orbitRadius = newPlanetData.distance;
        const orbitLineGeometry = new THREE.TorusGeometry(orbitRadius, 0.001, 16, 100);
        const orbitLineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff });
        const orbitLine = new THREE.Line(orbitLineGeometry, orbitLineMaterial);
        orbitLine.rotation.x = Math.PI / 2; // Rotate the orbit line to be in the x-y plane
        scene.add(orbitLine);
    };


    let moon: THREE.Mesh<THREE.SphereGeometry, THREE.MeshBasicMaterial>; // Declare moon outside of any reactive context

    const initMoons = () => {
        const moonRadius = 0.02; // Set the moon's radius
        const moonGeometry = new THREE.SphereGeometry(moonRadius, 32, 32); // Use the scaled radius
        const moonMaterial = new THREE.MeshBasicMaterial({ color: 0xaaaaaa }); // Grey color for the moon
        moon = new THREE.Mesh(moonGeometry, moonMaterial);
        const earthPosition = planets[2].position; // Assuming Earth is the third planet in the planets array
        moon.position.set(earthPosition.x + 0.1, 0, earthPosition.z); // Set initial position slightly offset from Earth
        scene.add(moon);
    };

    $: {
        if (moon) {
            const earthPosition = planets[2].position; // Assuming Earth is the third planet in the planets array
            const theta = performance.now() * 0.001; // Adjust speed as needed
            const moonDistance = 0.1; // Assuming moonDistance is 0.1
            const x = Math.cos(theta) * moonDistance + earthPosition.x;
            const z = Math.sin(theta) * moonDistance + earthPosition.z;
            moon.position.set(x, 0, z);
        }
    }




    const init = () => {
        scene = new THREE.Scene();
        camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.up.set(0, 1, 0);

        renderer = new THREE.WebGLRenderer();
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const textureLoader = new THREE.TextureLoader();
        const texture = textureLoader.load('/sun_texture.jpg');

        const sunRadius = 1; // Set the sun's radius to scale
        const sunGeometry = new THREE.SphereGeometry(sunRadius, 32, 32); // Use the scaled radius
        const sunMaterial = new THREE.MeshBasicMaterial({ color: 0xffff00 }); // Yellow color for the sun
        sun = new THREE.Mesh(sunGeometry, sunMaterial); // Assign to global variable
        scene.add(sun);

        // Inside the init function
        const sunLight = new THREE.PointLight(0xffffff, 1); // White light, intensity 1
        sunLight.position.copy(sun.position); // Position the light at the sun's position
        scene.add(sunLight);

        renderer.shadowMap.enabled = true; // Enable shadow mapping

        // Enable shadows for the sun
        sun.castShadow = true;
        sun.receiveShadow = true;

        // Configure shadow properties for the light
        sunLight.castShadow = true;
        sunLight.shadow.mapSize.width = 1024;
        sunLight.shadow.mapSize.height = 1024;
        sunLight.shadow.camera.near = 0.5;
        sunLight.shadow.camera.far = 500;

        // Bloom effect
        const composer = new EffectComposer(renderer);
        const renderPass = new RenderPass(scene, camera);
        composer.addPass(renderPass);

        const bloomPass = new UnrealBloomPass();
        bloomPass.strength = 1; // Adjust the strength of the bloom effect
        bloomPass.radius = 0.5; // Adjust the radius of the bloom effect
        bloomPass.threshold = 0.5; // Adjust the threshold for what is considered bright enough to bloom

        composer.addPass(bloomPass);

        window.composer = composer;

        camera.position.z = 3;

        const canvas = renderer.domElement;
        canvas.addEventListener('click', lockMouse);

        // Start the animation loop
        animate();
    };

    const render = () => {
        renderer.clear()
        renderer.render(scene, camera);
        composer.render();
    };

    const updateMoon = () => {
    if (moon) {
        const earthPosition = planets[2].position; // Assuming Earth is the third planet in the planets array
        const theta = performance.now() * 0.001; // Adjust speed as needed
        const moonDistance = 0.1; // Assuming moonDistance is 0.1
        const x = Math.cos(theta) * moonDistance + earthPosition.x;
        const z = Math.sin(theta) * moonDistance + earthPosition.z;
        moon.position.set(x, 0, z);
    }
};

    const animate = (time: number) => {
        requestAnimationFrame(animate);

        if (!isPaused) {
            updateCameraPosition();
            updatePlanets();
            updateMoon();
        }

        camera.updateMatrixWorld();

        render();

        const delta = time - lastTime;
        const fps = 1 / (delta / 1000);
        fpsTotal += fps;
        fpsCount++;

        if (delta >= 1000) {
            avgFps = fpsTotal / fpsCount;
            fpsTotal = 0;
            fpsCount = 0;
            lastTime = time;
        }

        fpsElement.innerText = `Average FPS (last second): ${avgFps.toFixed(1)}`;
    };


        init();
        initPlanets();
        initMoons();
        initOrbitLines(planetData);
        animate(performance.now());
    }

</script>


<style>
    #fps {
        position: absolute;
        top: 10px;
        left: 10px;
        color: white;
        font-family: Arial, sans-serif;
        font-size: 12px;
        z-index: 9999;
    }

    #menu {
        position: absolute;
        top: 10px;
        right: 10px;
        display: flex;
        flex-direction: column;
        align-items: flex-end;
        gap: 0.5em;
        background-color: white;
        padding: 1em;
    }

    #controls {
        flex-direction: column;
        align-items: flex-end;
        gap: 0.5em;
        background-color: white;
        padding: 1em;
    }

    #help {
        flex-direction: column;
        align-items: flex-end;
        gap: 0.5em;
        background-color: white;
        padding: 1em;
    
    }
</style>

<section id="menu" class="p-4 bg-gray-200 flex flex-col border align-middle items">

    <div id="fps" class="mb-4"></div>

    <div id="controls" class="flex justify-between items-center mb-4">
        <button id="play-pause" class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600">Pause</button>
        <button class="bg-gray-500 text-white px-4 py-2 rounded hover:bg-gray-600" on:click={() => composer.passes[1].enabled = !composer.passes[1].enabled}>Toggle Bloom</button>
        <button class="bg-gray-500 text-white px-4 py-2 rounded hover:bg-gray-600" x-on:click="handleToggleWireframe()">Toggle Wireframe</button>
        <input type="range" id="speed-slider" min="0.1" max="2" step="0.1" x-bind:value="speed" class="w-24">
    </div>

    <div id="help" class="mb-4">
        <p>Click on the simulation to begin! It will lock the mouse.</p>
        <p>WASD to move</p>
        <p>Move your mouse to look around</p>
        <p>Press ESC to un-lock your mouse</p>
        <p>Press Shift to move faster</p>
        <p>Press Space to move up</p>
        <p>Press Ctrl to move down</p>
        <br/>
        <p>Get more help on our GitHub repository!</p>
        <ActionIcon color='gray' variant='hover' root='a'
            href="https://github.com/racecn/S3"
        ><GithubLogo size={64} /></ActionIcon>
    </div>

    <section id="create-planet" class="p-4 flex">
        <h2 class="text-2xl font-bold mb-4">Add a Planet</h2>
        <form id="addPlanetForm" class="space-y-4">
            <div class="flex flex-col">
                <label for="planetName" class="mb-1">Planet Name:</label>
                <input class="input border border-gray-300 rounded px-2 py-1" type="text" id="planetName" name="planetName" required>
            </div>
    
            <div class="flex flex-col">
                <label for="planetDistance" class="mb-1">Distance from Sun:</label>
                <RangeSlider bind:value={planetDistance} max="25" step="1" class="w-full">
                    <div class="flex justify-between items-center">
                        <div class="font-bold">Kilometers</div>
                        <div class="text-xs">{planetDistance} / 25k km</div>
                    </div>
                </RangeSlider>
            </div>
    
            <div class="flex flex-col">
                <label for="planetRadius" class="mb-1">Radius:</label>
                <input class="input border border-gray-300 rounded px-2 py-1" type="number" id="planetRadius" name="planetRadius" step="0.01" required>
            </div>
    
            <div class="flex flex-col">
                <label for="planetSpeed" class="mb-1">Speed:</label>
                <input class="input border border-gray-300 rounded px-2 py-1" type="number" id="planetSpeed" name="planetSpeed" step="0.0001" required>
            </div>
    
            <button type="submit" class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600">Add Planet</button>
        </form>
    </section>
</section>



<svelte:head>
    <title>S3</title>
</svelte:head>

<script lang="ts">
  import * as THREE from "three";
  import { OrbitControls } from "three/addons/controls/OrbitControls.js";
  import { onMount, onDestroy } from "svelte";

  let {
    spaceWidth = 20,
    spaceLength = 20,
    fieldWidth = 10,
    fieldLength = 10,
    gridWidth = 1,
    gridLength = 1,
    cameraFree = false,
  }: {
    spaceWidth?: number;
    spaceLength?: number;
    fieldWidth?: number;
    fieldLength?: number;
    gridWidth?: number;
    gridLength?: number;
    cameraFree?: boolean;
  } = $props();

  let container: HTMLDivElement;
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let controls: OrbitControls;
  let animationId: number;

  // Free camera state
  let flyKeys: Record<string, boolean> = {};
  let flyMouseDragging = false;
  let flyLastX = 0;
  let flyLastY = 0;
  let flyYaw = 0;
  let flyPitch = 0;
  const flySpeed = 0.3;

  function onKeyDown(e: KeyboardEvent) {
    if (!cameraFree) return;
    flyKeys[e.code] = true;
  }
  function onKeyUp(e: KeyboardEvent) {
    if (!cameraFree) return;
    flyKeys[e.code] = false;
  }
  function onFlyMouseDown(e: MouseEvent) {
    if (!cameraFree) return;
    flyMouseDragging = true;
    flyLastX = e.clientX;
    flyLastY = e.clientY;
  }
  function onFlyMouseMove(e: MouseEvent) {
    if (!cameraFree || !flyMouseDragging) return;
    const dx = e.clientX - flyLastX;
    const dy = e.clientY - flyLastY;
    flyLastX = e.clientX;
    flyLastY = e.clientY;
    flyYaw -= dx * 0.005;
    flyPitch -= dy * 0.005;
    flyPitch = Math.max(
      -Math.PI / 2 + 0.01,
      Math.min(Math.PI / 2 - 0.01, flyPitch),
    );
  }
  function onFlyMouseUp() {
    flyMouseDragging = false;
  }

  function updateFlyCamera() {
    if (!cameraFree || !camera) return;
    // Apply rotation from yaw/pitch
    const euler = new THREE.Euler(flyPitch, flyYaw, 0, "YXZ");
    camera.quaternion.setFromEuler(euler);

    // Movement relative to camera direction
    const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(
      camera.quaternion,
    );
    const right = new THREE.Vector3(1, 0, 0).applyQuaternion(camera.quaternion);
    const up = new THREE.Vector3(0, 1, 0);

    const move = new THREE.Vector3();
    if (flyKeys["KeyW"]) move.add(forward);
    if (flyKeys["KeyS"]) move.sub(forward);
    if (flyKeys["KeyA"]) move.sub(right);
    if (flyKeys["KeyD"]) move.add(right);
    if (flyKeys["Space"]) move.add(up);
    if (flyKeys["ShiftLeft"] || flyKeys["ShiftRight"]) move.sub(up);

    if (move.lengthSq() > 0) {
      move.normalize().multiplyScalar(flySpeed);
      camera.position.add(move);
    }
  }

  function setupCameraMode() {
    if (!controls || !camera) return;
    if (cameraFree) {
      controls.enabled = false;
      // Sync yaw/pitch from current camera orientation
      const euler = new THREE.Euler().setFromQuaternion(
        camera.quaternion,
        "YXZ",
      );
      flyYaw = euler.y;
      flyPitch = euler.x;
    } else {
      controls.enabled = true;
    }
  }

  let spaceEdges: THREE.LineSegments;
  let spaceWalls: THREE.Mesh;
  let fieldMesh: THREE.Mesh;
  let gridLines: THREE.LineSegments;

  function initScene() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a1a2e);

    camera = new THREE.PerspectiveCamera(
      60,
      container.clientWidth / container.clientHeight,
      0.1,
      1000,
    );
    const camDist = Math.max(spaceWidth, spaceLength);
    camera.position.set(camDist * 1.5, camDist * 1.2, camDist * 1.5);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(container.clientWidth, container.clientHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);

    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.maxDistance = 200;
    controls.minDistance = 2;

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(20, 30, 20);
    scene.add(directionalLight);

    buildScene();
    animate();
  }

  function buildScene() {
    if (spaceEdges) {
      scene.remove(spaceEdges);
      spaceEdges.geometry.dispose();
    }
    if (spaceWalls) {
      scene.remove(spaceWalls);
      spaceWalls.geometry.dispose();
    }
    if (fieldMesh) {
      scene.remove(fieldMesh);
      fieldMesh.geometry.dispose();
    }
    if (gridLines) {
      scene.remove(gridLines);
      gridLines.geometry.dispose();
    }

    const sw = Math.max(1, spaceWidth);
    const sl = Math.max(1, spaceLength);
    const sh = Math.max(sw, sl);
    const fw = Math.max(1, Math.min(fieldWidth, sw));
    const fl = Math.max(1, Math.min(fieldLength, sl));
    const gw = Math.max(0.1, gridWidth);
    const gl = Math.max(0.1, gridLength);

    // Space — box with walls
    const boxGeo = new THREE.BoxGeometry(sw, sh, sl);
    const edgesGeo = new THREE.EdgesGeometry(boxGeo);
    spaceEdges = new THREE.LineSegments(
      edgesGeo,
      new THREE.LineBasicMaterial({ color: 0x6a6a9a }),
    );
    spaceEdges.position.y = sh / 2;
    scene.add(spaceEdges);

    spaceWalls = new THREE.Mesh(
      boxGeo,
      new THREE.MeshBasicMaterial({
        color: 0x2a2a4e,
        transparent: true,
        opacity: 0.1,
        side: THREE.BackSide,
      }),
    );
    spaceWalls.position.y = sh / 2;
    scene.add(spaceWalls);

    // Field — plane on the floor
    const fieldGeo = new THREE.PlaneGeometry(fw, fl);
    fieldMesh = new THREE.Mesh(
      fieldGeo,
      new THREE.MeshStandardMaterial({
        color: 0x3a5a3a,
        side: THREE.DoubleSide,
      }),
    );
    fieldMesh.rotation.x = -Math.PI / 2;
    fieldMesh.position.y = 0.01;
    scene.add(fieldMesh);

    // Custom grid on the field (rectangular cells)
    const halfFw = fw / 2;
    const halfFl = fl / 2;
    const colsX = Math.max(1, Math.round(fw / gw));
    const colsZ = Math.max(1, Math.round(fl / gl));
    const pts: number[] = [];
    for (let i = 0; i <= colsX; i++) {
      const x = -halfFw + (i * fw) / colsX;
      pts.push(x, 0, -halfFl, x, 0, halfFl);
    }
    for (let j = 0; j <= colsZ; j++) {
      const z = -halfFl + (j * fl) / colsZ;
      pts.push(-halfFw, 0, z, halfFw, 0, z);
    }
    const gridGeo = new THREE.BufferGeometry();
    gridGeo.setAttribute("position", new THREE.Float32BufferAttribute(pts, 3));
    gridLines = new THREE.LineSegments(
      gridGeo,
      new THREE.LineBasicMaterial({ color: 0x888888 }),
    );
    gridLines.position.y = 0.02;
    scene.add(gridLines);
  }

  function animate() {
    animationId = requestAnimationFrame(animate);
    if (cameraFree) {
      updateFlyCamera();
    } else {
      controls.update();
    }
    renderer.render(scene, camera);
  }

  function onResize() {
    if (!container || !renderer) return;
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  }

  onMount(() => {
    initScene();
    window.addEventListener("resize", onResize);
    window.addEventListener("keydown", onKeyDown);
    window.addEventListener("keyup", onKeyUp);
    window.addEventListener("mousemove", onFlyMouseMove);
    window.addEventListener("mouseup", onFlyMouseUp);
  });

  onDestroy(() => {
    if (typeof cancelAnimationFrame !== "undefined")
      cancelAnimationFrame(animationId);
    if (typeof window !== "undefined") {
      window.removeEventListener("resize", onResize);
      window.removeEventListener("keydown", onKeyDown);
      window.removeEventListener("keyup", onKeyUp);
      window.removeEventListener("mousemove", onFlyMouseMove);
      window.removeEventListener("mouseup", onFlyMouseUp);
    }
    if (renderer) {
      renderer.dispose();
      if (renderer.domElement.parentNode === container) {
        container.removeChild(renderer.domElement);
      }
    }
  });

  $effect(() => {
    if (scene) {
      buildScene();
    }
  });

  $effect(() => {
    setupCameraMode();
  });
</script>

<div
  bind:this={container}
  class="w-full h-full"
  onmousedown={onFlyMouseDown}
></div>

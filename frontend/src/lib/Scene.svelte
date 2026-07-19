<script lang="ts">
  import * as THREE from "three";
  import { OrbitControls } from "three/addons/controls/OrbitControls.js";
  import { onMount, onDestroy } from "svelte";

  interface AssetData {
    id: string;
    type: "box";
    name: string;
    x: number;
    y: number;
    z: number;
    width: number;
    height: number;
    depth: number;
  }

  let {
    spaceWidth = 20,
    spaceLength = 20,
    fieldWidth = 10,
    fieldLength = 10,
    gridWidth = 1,
    gridLength = 1,
    gridOpacity = 0.5,
    cameraFree = false,
    assets = [] as AssetData[],
    selectedId = null as string | null,
    onSelect = (_id: string | null) => {},
    onMove = (_id: string, _x: number, _y: number, _z: number) => {},
  }: {
    spaceWidth?: number;
    spaceLength?: number;
    fieldWidth?: number;
    fieldLength?: number;
    gridWidth?: number;
    gridLength?: number;
    gridOpacity?: number;
    cameraFree?: boolean;
    assets?: AssetData[];
    selectedId?: string | null;
    onSelect?: (id: string | null) => void;
    onMove?: (id: string, x: number, y: number, z: number) => void;
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

  const flyKeyCodes = new Set([
    "KeyW",
    "KeyA",
    "KeyS",
    "KeyD",
    "Space",
    "ShiftLeft",
    "ShiftRight",
    "ControlLeft",
    "ControlRight",
    "ArrowUp",
    "ArrowDown",
    "ArrowLeft",
    "ArrowRight",
  ]);

  function onKeyDown(e: KeyboardEvent) {
    if (!cameraFree) return;
    if (flyKeyCodes.has(e.code)) {
      e.preventDefault();
      flyKeys[e.code] = true;
    }
  }
  function onKeyUp(e: KeyboardEvent) {
    if (!cameraFree) return;
    if (flyKeyCodes.has(e.code)) {
      e.preventDefault();
      flyKeys[e.code] = false;
    }
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
      flyKeys = {};
    }
  }

  let spaceEdges: THREE.LineSegments;
  let spaceWalls: THREE.Mesh;
  let fieldMesh: THREE.Mesh;
  let gridLines: THREE.LineSegments;

  // Asset meshes
  let assetGroup: THREE.Group;
  let assetMeshes = new Map<string, THREE.Mesh>();
  let selectionRing: THREE.Mesh;

  // Gizmo (axis arrows)
  let gizmoGroup: THREE.Group;
  let gizmoArrows: THREE.Mesh[] = [];
  let gizmoAxis: "x" | "y" | "z" | null = null;
  let gizmoStartPos = new THREE.Vector3();
  let gizmoDragId: string | null = null;

  // Interaction state
  let raycaster = new THREE.Raycaster();
  let pointer = new THREE.Vector2();
  let isDragging = false;
  let dragId: string | null = null;
  let dragPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
  let dragOffset = new THREE.Vector3();
  let pointerDownPos = { x: 0, y: 0 };
  let pointerDownTime = 0;

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
    controls.enablePan = false;

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(20, 30, 20);
    scene.add(directionalLight);

    buildScene();
    animate();
    buildAssets();
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
      new THREE.LineBasicMaterial({
        color: 0x888888,
        transparent: true,
        opacity: Math.max(0, Math.min(1, gridOpacity)),
      }),
    );
    gridLines.position.y = 0.02;
    scene.add(gridLines);
  }

  function buildAssets() {
    if (!assetGroup) {
      assetGroup = new THREE.Group();
      scene.add(assetGroup);
    }

    // Remove old meshes
    for (const [id, mesh] of assetMeshes) {
      assetGroup.remove(mesh);
      mesh.geometry.dispose();
    }
    assetMeshes.clear();

    // Create new meshes (boxes)
    for (const a of assets) {
      const w = Math.max(0.01, a.width);
      const h = Math.max(0.01, a.height);
      const d = Math.max(0.01, a.depth);
      const geo = new THREE.BoxGeometry(w, h, d);
      const mat = new THREE.MeshStandardMaterial({
        color: 0xffffff,
        emissive: selectedId === a.id ? 0x442200 : 0x000000,
        metalness: 0.1,
        roughness: 0.5,
      });
      const mesh = new THREE.Mesh(geo, mat);
      mesh.position.set(a.x, a.y, a.z);
      mesh.userData.assetId = a.id;
      assetGroup.add(mesh);
      assetMeshes.set(a.id, mesh);
    }

    // Selection ring
    if (selectionRing) {
      assetGroup.remove(selectionRing);
      selectionRing.geometry.dispose();
      selectionRing = undefined as any;
    }
    if (selectedId) {
      const a = assets.find((a) => a.id === selectedId);
      if (a) {
        const ringR = Math.max(0.01, Math.max(a.width, a.depth) / 2 + 0.05);
        const ringGeo = new THREE.RingGeometry(ringR, ringR + 0.03, 32);
        const ringMat = new THREE.MeshBasicMaterial({
          color: 0xff8800,
          side: THREE.DoubleSide,
          transparent: true,
          opacity: 0.8,
        });
        selectionRing = new THREE.Mesh(ringGeo, ringMat);
        selectionRing.rotation.x = -Math.PI / 2;
        selectionRing.position.set(a.x, 0.03, a.z);
        assetGroup.add(selectionRing);
      }
    }

    buildGizmo();
  }

  function buildGizmo() {
    if (gizmoGroup) {
      assetGroup.remove(gizmoGroup);
      for (const arrow of gizmoArrows) {
        arrow.geometry.dispose();
      }
      gizmoArrows = [];
    }

    if (!selectedId) return;
    const a = assets.find((a) => a.id === selectedId);
    if (!a) return;

    gizmoGroup = new THREE.Group();

    const arrowLen = Math.max(a.width, a.height, a.depth) * 0.8 + 0.3;
    const arrowRadius = 0.02;
    const headRadius = 0.05;
    const headLen = 0.12;

    const axes: { dir: THREE.Vector3; color: number; name: "x" | "y" | "z" }[] =
      [
        { dir: new THREE.Vector3(1, 0, 0), color: 0xff3333, name: "x" },
        { dir: new THREE.Vector3(0, 1, 0), color: 0x33ff33, name: "y" },
        { dir: new THREE.Vector3(0, 0, 1), color: 0x3333ff, name: "z" },
      ];

    for (const axis of axes) {
      // Arrow shaft (cylinder)
      const shaftGeo = new THREE.CylinderGeometry(
        arrowRadius,
        arrowRadius,
        arrowLen,
        8,
      );
      const shaftMat = new THREE.MeshBasicMaterial({ color: axis.color });
      const shaft = new THREE.Mesh(shaftGeo, shaftMat);

      // Arrow head (cone)
      const headGeo = new THREE.ConeGeometry(headRadius, headLen, 12);
      const headMat = new THREE.MeshBasicMaterial({ color: axis.color });
      const head = new THREE.Mesh(headGeo, headMat);

      // Group for this axis
      const arrowGroup = new THREE.Group();
      shaft.position.y = arrowLen / 2;
      head.position.y = arrowLen + headLen / 2;
      arrowGroup.add(shaft);
      arrowGroup.add(head);

      // Orient along axis direction
      if (axis.name === "x") {
        arrowGroup.rotation.z = -Math.PI / 2;
      } else if (axis.name === "z") {
        arrowGroup.rotation.x = Math.PI / 2;
      }

      arrowGroup.userData.axis = axis.name;
      arrowGroup.userData.isGizmo = true;
      gizmoGroup.add(arrowGroup);
      gizmoArrows.push(shaft, head);
    }

    gizmoGroup.position.set(a.x, a.y, a.z);
    assetGroup.add(gizmoGroup);
  }

  function getPointerCoords(e: MouseEvent) {
    const rect = renderer.domElement.getBoundingClientRect();
    pointer.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    pointer.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  }

  function pickAsset(): THREE.Mesh | null {
    raycaster.setFromCamera(pointer, camera);
    const meshes = Array.from(assetMeshes.values());
    const hits = raycaster.intersectObjects(meshes, false);
    return hits.length > 0 ? (hits[0].object as THREE.Mesh) : null;
  }

  function pickGizmo(): "x" | "y" | "z" | null {
    if (!gizmoGroup) return null;
    raycaster.setFromCamera(pointer, camera);
    const hits = raycaster.intersectObjects(gizmoArrows, false);
    if (hits.length > 0) {
      // Walk up to find the group with axis data
      let obj: THREE.Object3D | null = hits[0].object;
      while (obj) {
        if (obj.userData && obj.userData.axis) {
          return obj.userData.axis as "x" | "y" | "z";
        }
        obj = obj.parent;
      }
    }
    return null;
  }

  function onSceneMouseDown(e: MouseEvent) {
    if (cameraFree) {
      onFlyMouseDown(e);
      return;
    }
    if (e.button !== 0) return;
    getPointerCoords(e);
    pointerDownPos = { x: e.clientX, y: e.clientY };
    pointerDownTime = Date.now();

    // Check gizmo first
    const axis = pickGizmo();
    if (axis && selectedId) {
      gizmoAxis = axis;
      gizmoDragId = selectedId;
      isDragging = false;
      const a = assets.find((a) => a.id === selectedId);
      if (a) {
        gizmoStartPos.set(a.x, a.y, a.z);
        // Set drag plane perpendicular to the axis, facing camera
        const axisVec = new THREE.Vector3(
          axis === "x" ? 1 : 0,
          axis === "y" ? 1 : 0,
          axis === "z" ? 1 : 0,
        );
        // Use a plane that contains the axis and is most facing the camera
        const camDir = new THREE.Vector3();
        camera.getWorldDirection(camDir);
        // Pick plane normal as the axis direction projected onto camera-facing
        // Simplest: use a plane whose normal is the axis itself
        dragPlane.setFromNormalAndCoplanarPoint(axisVec, gizmoStartPos);
        raycaster.setFromCamera(pointer, camera);
        const intersect = new THREE.Vector3();
        raycaster.ray.intersectPlane(dragPlane, intersect);
        dragOffset.copy(gizmoStartPos).sub(intersect);
      }
      controls.enabled = false;
      return;
    }

    // Check asset
    const hit = pickAsset();
    if (hit) {
      const id = hit.userData.assetId as string;
      onSelect(id);
      dragId = id;
      isDragging = false;

      const a = assets.find((a) => a.id === id);
      if (a) {
        dragPlane.set(new THREE.Vector3(0, 1, 0), -a.y);
        raycaster.setFromCamera(pointer, camera);
        const intersect = new THREE.Vector3();
        raycaster.ray.intersectPlane(dragPlane, intersect);
        dragOffset.set(a.x - intersect.x, 0, a.z - intersect.z);
      }
      controls.enabled = false;
    }
  }

  function onSceneMouseMove(e: MouseEvent) {
    if (cameraFree) {
      onFlyMouseMove(e);
      return;
    }

    // Gizmo drag
    if (gizmoAxis && gizmoDragId) {
      const dx = e.clientX - pointerDownPos.x;
      const dy = e.clientY - pointerDownPos.y;
      if (!isDragging && Math.sqrt(dx * dx + dy * dy) > 3) {
        isDragging = true;
      }
      if (!isDragging) return;

      getPointerCoords(e);
      raycaster.setFromCamera(pointer, camera);
      const intersect = new THREE.Vector3();
      if (raycaster.ray.intersectPlane(dragPlane, intersect)) {
        const newCoord = intersect.add(dragOffset);
        const a = assets.find((a) => a.id === gizmoDragId);
        if (a) {
          if (gizmoAxis === "x") {
            onMove(gizmoDragId, newCoord.x, a.y, a.z);
          } else if (gizmoAxis === "y") {
            onMove(gizmoDragId, a.x, newCoord.y, a.z);
          } else if (gizmoAxis === "z") {
            onMove(gizmoDragId, a.x, a.y, newCoord.z);
          }
        }
      }
      return;
    }

    // Free drag (XZ plane)
    if (!dragId) return;
    const dx = e.clientX - pointerDownPos.x;
    const dy = e.clientY - pointerDownPos.y;
    if (!isDragging && Math.sqrt(dx * dx + dy * dy) > 3) {
      isDragging = true;
    }
    if (!isDragging) return;

    getPointerCoords(e);
    raycaster.setFromCamera(pointer, camera);
    const intersect = new THREE.Vector3();
    if (raycaster.ray.intersectPlane(dragPlane, intersect)) {
      const newX = intersect.x + dragOffset.x;
      const newZ = intersect.z + dragOffset.z;
      const a = assets.find((a) => a.id === dragId);
      if (a) {
        onMove(dragId, newX, a.y, newZ);
      }
    }
  }

  function onSceneMouseUp(e: MouseEvent) {
    if (cameraFree) {
      onFlyMouseUp();
      return;
    }

    if (gizmoAxis) {
      gizmoAxis = null;
      gizmoDragId = null;
      isDragging = false;
      controls.enabled = true;
      return;
    }

    if (dragId) {
      dragId = null;
      isDragging = false;
      controls.enabled = true;
      return;
    }

    // Click on empty space — deselect if quick click
    const dt = Date.now() - pointerDownTime;
    const dx = e.clientX - pointerDownPos.x;
    const dy = e.clientY - pointerDownPos.y;
    if (dt < 300 && Math.sqrt(dx * dx + dy * dy) < 5) {
      getPointerCoords(e);
      const hit = pickAsset();
      if (!hit) onSelect(null);
    }
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
    window.addEventListener("mousemove", onSceneMouseMove);
    window.addEventListener("mouseup", onSceneMouseUp);
  });

  onDestroy(() => {
    if (typeof cancelAnimationFrame !== "undefined")
      cancelAnimationFrame(animationId);
    if (typeof window !== "undefined") {
      window.removeEventListener("resize", onResize);
      window.removeEventListener("keydown", onKeyDown);
      window.removeEventListener("keyup", onKeyUp);
      window.removeEventListener("mousemove", onSceneMouseMove);
      window.removeEventListener("mouseup", onSceneMouseUp);
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

  $effect(() => {
    if (scene) {
      buildAssets();
    }
  });
</script>

<div
  bind:this={container}
  class="w-full h-full"
  onmousedown={onSceneMouseDown}
></div>

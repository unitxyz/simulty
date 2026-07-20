<script lang="ts">
  import * as THREE from "three";
  import { OrbitControls } from "three/addons/controls/OrbitControls.js";
  import { onMount, onDestroy } from "svelte";

  type CharacterGender = "male" | "female";

  interface AssetData {
    id: string;
    type: "box" | "character";
    name: string;
    x: number;
    y: number;
    z: number;
    width: number;
    height: number;
    depth: number;
    rotX: number;
    rotY: number;
    rotZ: number;
    gender?: CharacterGender;
    headRotX?: number;
    leftArmRotX?: number;
    rightArmRotX?: number;
    leftForearmRotX?: number;
    rightForearmRotX?: number;
    leftLegRotX?: number;
    rightLegRotX?: number;
    leftShinRotX?: number;
    rightShinRotX?: number;
  }

  let {
    spaceWidth = 10,
    spaceLength = 10,
    fieldWidth = 8,
    fieldLength = 8,
    gridWidth = 1,
    gridLength = 1,
    gridOpacity = 0.5,
    gridOffsetY = 0,
    grid3D = false,
    gridVisible = true,
    gridLayerX = 0,
    gridLayerY = 0,
    gridLayerZ = 0,
    cameraFree = false,
    rotateMode = false,
    assets = [] as AssetData[],
    selectedId = null as string | null,
    onSelect = (_id: string | null) => {},
    onMove = (_id: string, _x: number, _y: number, _z: number) => {},
    onRotate = (_id: string, _rx: number, _ry: number, _rz: number) => {},
  }: {
    spaceWidth?: number;
    spaceLength?: number;
    fieldWidth?: number;
    fieldLength?: number;
    gridWidth?: number;
    gridLength?: number;
    gridOpacity?: number;
    gridOffsetY?: number;
    grid3D?: boolean;
    gridVisible?: boolean;
    gridLayerX?: number;
    gridLayerY?: number;
    gridLayerZ?: number;
    cameraFree?: boolean;
    rotateMode?: boolean;
    assets?: AssetData[];
    selectedId?: string | null;
    onSelect?: (id: string | null) => void;
    onMove?: (id: string, x: number, y: number, z: number) => void;
    onRotate?: (id: string, rx: number, ry: number, rz: number) => void;
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
  let originMarker: THREE.Mesh;

  // Asset meshes
  let assetGroup: THREE.Group;
  let assetMeshes = new Map<string, THREE.Mesh>();
  let selectionRing: THREE.Object3D;

  // Gizmo (axis arrows)
  let gizmoGroup: THREE.Group;
  let gizmoArrows: THREE.Mesh[] = [];
  let gizmoAxis: "x" | "y" | "z" | null = null;
  let gizmoStartPos = new THREE.Vector3();
  let gizmoDragId: string | null = null;
  let gizmoStartAngle = 0;
  let dragMoved = false; // tracks if drag changed position (to skip state sync if not)

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
    if (originMarker) {
      scene.remove(originMarker);
      originMarker.geometry.dispose();
      originMarker = undefined as any;
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

    // Grid: flat or 3D cubic
    const halfFw = fw / 2;
    const halfFl = fl / 2;
    const colsX = Math.max(1, Math.round(fw / gw));
    const colsZ = Math.max(1, Math.round(fl / gl));
    const pts: number[] = [];

    if (grid3D) {
      // 3D cubic grid
      const gridH = Math.max(0, gridOffsetY);
      const layersY = Math.max(1, Math.round(gridH / Math.max(gw, gl)));
      const cellW = fw / colsX;
      const cellD = fl / colsZ;
      const cellH = gridH / layersY;

      // Layer filter: 0 = no filter on that axis, >0 = 1-indexed
      const fX = Math.round(gridLayerX);
      const fY = Math.round(gridLayerY);
      const fZ = Math.round(gridLayerZ);
      const hasFilter = fX > 0 || fY > 0 || fZ > 0;

      // Helper: push 12 edges of a cube cell at (ix, iy, iz)
      function pushCubeEdges(ix: number, iy: number, iz: number) {
        const x0 = -halfFw + ix * cellW;
        const x1 = x0 + cellW;
        const y0 = iy * cellH;
        const y1 = y0 + cellH;
        const z0 = -halfFl + iz * cellD;
        const z1 = z0 + cellD;
        // Bottom face
        pts.push(x0, y0, z0, x1, y0, z0);
        pts.push(x1, y0, z0, x1, y0, z1);
        pts.push(x1, y0, z1, x0, y0, z1);
        pts.push(x0, y0, z1, x0, y0, z0);
        // Top face
        pts.push(x0, y1, z0, x1, y1, z0);
        pts.push(x1, y1, z0, x1, y1, z1);
        pts.push(x1, y1, z1, x0, y1, z1);
        pts.push(x0, y1, z1, x0, y1, z0);
        // Vertical edges
        pts.push(x0, y0, z0, x0, y1, z0);
        pts.push(x1, y0, z0, x1, y1, z0);
        pts.push(x1, y0, z1, x1, y1, z1);
        pts.push(x0, y0, z1, x0, y1, z1);
      }

      if (!hasFilter) {
        // Draw full grid (all lines like before)
        for (let ly = 0; ly <= layersY; ly++) {
          const y = (ly * gridH) / layersY;
          for (let i = 0; i <= colsX; i++) {
            const x = -halfFw + (i * fw) / colsX;
            pts.push(x, y, -halfFl, x, y, halfFl);
          }
          for (let j = 0; j <= colsZ; j++) {
            const z = -halfFl + (j * fl) / colsZ;
            pts.push(-halfFw, y, z, halfFw, y, z);
          }
        }
        for (let i = 0; i <= colsX; i++) {
          const x = -halfFw + (i * fw) / colsX;
          for (let j = 0; j <= colsZ; j++) {
            const z = -halfFl + (j * fl) / colsZ;
            pts.push(x, 0, z, x, gridH, z);
          }
        }
      } else {
        // Draw only edges of matching cube cells
        for (let ix = 0; ix < colsX; ix++) {
          if (fX > 0 && ix + 1 !== fX) continue;
          for (let iy = 0; iy < layersY; iy++) {
            if (fY > 0 && iy + 1 !== fY) continue;
            for (let iz = 0; iz < colsZ; iz++) {
              if (fZ > 0 && iz + 1 !== fZ) continue;
              pushCubeEdges(ix, iy, iz);
            }
          }
        }
      }
    } else {
      // Flat grid
      for (let i = 0; i <= colsX; i++) {
        const x = -halfFw + (i * fw) / colsX;
        pts.push(x, 0, -halfFl, x, 0, halfFl);
      }
      for (let j = 0; j <= colsZ; j++) {
        const z = -halfFl + (j * fl) / colsZ;
        pts.push(-halfFw, 0, z, halfFw, 0, z);
      }
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
    gridLines.position.y = 0.02 + (grid3D ? 0 : Math.max(0, gridOffsetY));
    gridLines.visible = gridVisible;
    scene.add(gridLines);

    // Origin corner marker — orange 0.1x0.1 square at top-left corner
    if (originMarker) {
      scene.remove(originMarker);
      originMarker.geometry.dispose();
    }
    if (grid3D && gridVisible) {
      const markerGeo = new THREE.PlaneGeometry(0.05, 0.05);
      const markerMat = new THREE.MeshBasicMaterial({
        color: 0xff8800,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.9,
        depthWrite: false,
        depthTest: false,
      });
      originMarker = new THREE.Mesh(markerGeo, markerMat);
      originMarker.rotation.x = -Math.PI / 2;
      originMarker.position.set(-halfFw + 0.025, 0.012, -halfFl + 0.025);
      originMarker.renderOrder = 999;
      scene.add(originMarker);
    }
  }

  function createCharacter(a: AssetData): THREE.Group {
    const group = new THREE.Group();
    const isFemale = a.gender === "female";
    const body = new THREE.MeshStandardMaterial({
      color: isFemale ? 0xff70b7 : 0x54a9ff,
      roughness: 0.8,
    });

    const makeBox = (
      width: number,
      height: number,
      depth: number,
      material: THREE.Material,
    ) => new THREE.Mesh(new THREE.BoxGeometry(width, height, depth), material);

    const torsoWidth = isFemale ? 0.34 : 0.42;
    const torsoHeight = isFemale ? 0.58 : 0.62;
    const limbWidth = 0.12;
    const limbDepth = 0.12;
    const upperArmLength = 0.34;
    const forearmLength = 0.32;
    const thighLength = 0.48;
    const shinLength = 0.46;
    const hipY = thighLength + shinLength;
    const shoulderY = hipY + 0.12 + torsoHeight - 0.18;
    const headY = hipY + 0.12 + torsoHeight + 0.15;

    const torso = makeBox(torsoWidth, torsoHeight, isFemale ? 0.2 : 0.24, body);
    torso.position.y = hipY + 0.12 + torsoHeight / 2;
    torso.userData.assetId = a.id;
    group.add(torso);

    const pelvis = makeBox(torsoWidth * 0.9, 0.24, 0.22, body);
    pelvis.position.y = hipY;
    pelvis.userData.assetId = a.id;
    group.add(pelvis);

    const head = new THREE.Mesh(
      new THREE.IcosahedronGeometry(isFemale ? 0.145 : 0.16, 0),
      body,
    );
    head.position.y = headY;
    head.rotation.x = THREE.MathUtils.degToRad(a.headRotX ?? 0);
    head.userData.assetId = a.id;
    group.add(head);

    const createArm = (
      side: number,
      upperRotation: number,
      forearmRotation: number,
    ) => {
      const shoulder = new THREE.Group();
      shoulder.position.set(
        side * (torsoWidth / 2 + limbWidth / 2),
        shoulderY,
        0,
      );
      shoulder.rotation.x = THREE.MathUtils.degToRad(upperRotation);

      const upper = makeBox(limbWidth, upperArmLength, limbDepth, body);
      upper.position.y = -upperArmLength / 2;
      upper.userData.assetId = a.id;
      shoulder.add(upper);

      const elbow = new THREE.Group();
      elbow.position.y = -upperArmLength;
      elbow.rotation.x = THREE.MathUtils.degToRad(forearmRotation);
      const forearm = makeBox(
        limbWidth * 0.95,
        forearmLength,
        limbDepth * 0.95,
        body,
      );
      forearm.position.y = -forearmLength / 2;
      forearm.userData.assetId = a.id;
      elbow.add(forearm);
      shoulder.add(elbow);
      group.add(shoulder);
    };

    createArm(-1, a.leftArmRotX ?? 0, a.leftForearmRotX ?? 0);
    createArm(1, a.rightArmRotX ?? 0, a.rightForearmRotX ?? 0);

    const createLeg = (
      side: number,
      thighRotation: number,
      shinRotation: number,
    ) => {
      const hip = new THREE.Group();
      hip.position.set(side * (torsoWidth * 0.25), hipY, 0);
      hip.rotation.x = THREE.MathUtils.degToRad(thighRotation);

      const thigh = makeBox(
        limbWidth * 1.15,
        thighLength,
        limbDepth * 1.1,
        body,
      );
      thigh.position.y = -thighLength / 2;
      thigh.userData.assetId = a.id;
      hip.add(thigh);

      const knee = new THREE.Group();
      knee.position.y = -thighLength;
      knee.rotation.x = THREE.MathUtils.degToRad(shinRotation);
      const shin = makeBox(limbWidth, shinLength, limbDepth, body);
      shin.position.y = -shinLength / 2;
      shin.userData.assetId = a.id;
      knee.add(shin);
      hip.add(knee);
      group.add(hip);
    };

    createLeg(-1, a.leftLegRotX ?? 0, a.leftShinRotX ?? 0);
    createLeg(1, a.rightLegRotX ?? 0, a.rightShinRotX ?? 0);

    group.userData.assetId = a.id;
    group.userData.assetType = "character";
    group.position.set(a.x, a.y + 0.02, a.z);
    group.rotation.set(
      THREE.MathUtils.degToRad(a.rotX),
      THREE.MathUtils.degToRad(a.rotY),
      THREE.MathUtils.degToRad(a.rotZ),
    );
    return group;
  }

  function buildAssets() {
    if (!assetGroup) {
      assetGroup = new THREE.Group();
      scene.add(assetGroup);
    }

    // Remove old asset objects and all character parts
    for (const child of [...assetGroup.children]) {
      assetGroup.remove(child);
      child.traverse((object) => {
        const mesh = object as THREE.Mesh;
        if (mesh.geometry) mesh.geometry.dispose();
        const material = mesh.material as THREE.Material | THREE.Material[];
        if (Array.isArray(material)) {
          material.forEach((item) => item.dispose());
        } else if (material) {
          material.dispose();
        }
      });
    }
    assetMeshes.clear();
    selectionRing = undefined as any;
    gizmoGroup = undefined as any;
    gizmoArrows = [];

    for (const a of assets) {
      if (a.type === "character") {
        const character = createCharacter(a);
        assetGroup.add(character);
        continue;
      }

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
      mesh.rotation.set(
        (a.rotX * Math.PI) / 180,
        (a.rotY * Math.PI) / 180,
        (a.rotZ * Math.PI) / 180,
      );
      mesh.userData.assetId = a.id;
      assetGroup.add(mesh);
      assetMeshes.set(a.id, mesh);
    }

    // Selection square outline
    if (selectionRing) {
      assetGroup.remove(selectionRing);
      (selectionRing as any).geometry?.dispose();
      selectionRing = undefined as any;
    }
    if (selectedId) {
      const a = assets.find((a) => a.id === selectedId);
      if (a) {
        const sw = Math.max(0.01, a.width + 0.06);
        const sh = Math.max(0.01, a.height + 0.06);
        const sd = Math.max(0.01, a.depth + 0.06);
        const boxGeo = new THREE.BoxGeometry(sw, sh, sd);
        const edgesGeo = new THREE.EdgesGeometry(boxGeo);
        selectionRing = new THREE.LineSegments(
          edgesGeo,
          new THREE.LineBasicMaterial({ color: 0xff8800, linewidth: 2 }),
        );
        selectionRing.position.set(a.x, a.y, a.z);
        selectionRing.rotation.set(
          (a.rotX * Math.PI) / 180,
          (a.rotY * Math.PI) / 180,
          (a.rotZ * Math.PI) / 180,
        );
        assetGroup.add(selectionRing);
        boxGeo.dispose();
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

    {
      // Axis arrows are used for both move and rotate modes
      const arrowLen = 0.18;
      const shaftSize = 0.008;
      const headSize = 0.025;
      const headLen = 0.045;

      const axes: {
        dir: THREE.Vector3;
        color: number;
        name: "x" | "y" | "z";
        offset: number;
      }[] = [
        {
          dir: new THREE.Vector3(1, 0, 0),
          color: 0xff3333,
          name: "x",
          offset: a.width / 2,
        },
        {
          dir: new THREE.Vector3(0, 1, 0),
          color: 0x33ff33,
          name: "y",
          offset: a.height / 2,
        },
        {
          dir: new THREE.Vector3(0, 0, 1),
          color: 0x3333ff,
          name: "z",
          offset: a.depth / 2,
        },
      ];

      for (const axis of axes) {
        const shaftGeo = new THREE.BoxGeometry(shaftSize, arrowLen, shaftSize);
        const shaftMat = new THREE.MeshBasicMaterial({ color: axis.color });
        const shaft = new THREE.Mesh(shaftGeo, shaftMat);

        const headGeo = new THREE.BoxGeometry(headSize, headLen, headSize);
        const headMat = new THREE.MeshBasicMaterial({ color: axis.color });
        const head = new THREE.Mesh(headGeo, headMat);

        const arrowGroup = new THREE.Group();
        shaft.position.y = arrowLen / 2;
        head.position.y = arrowLen + headLen / 2;
        arrowGroup.add(shaft);
        arrowGroup.add(head);

        if (axis.name === "x") {
          arrowGroup.rotation.z = -Math.PI / 2;
          arrowGroup.position.x = axis.offset;
        } else if (axis.name === "z") {
          arrowGroup.rotation.x = Math.PI / 2;
          arrowGroup.position.z = axis.offset;
        } else {
          arrowGroup.position.y = axis.offset;
        }

        arrowGroup.userData.axis = axis.name;
        arrowGroup.userData.isGizmo = true;
        gizmoGroup.add(arrowGroup);
        gizmoArrows.push(shaft, head);
      }
    }

    gizmoGroup.position.set(a.x, a.y, a.z);
    gizmoGroup.rotation.set(
      (a.rotX * Math.PI) / 180,
      (a.rotY * Math.PI) / 180,
      (a.rotZ * Math.PI) / 180,
    );
    assetGroup.add(gizmoGroup);
  }

  function getAssetObject(id: string): THREE.Object3D | null {
    for (const child of assetGroup.children) {
      if (child.userData && child.userData.assetId === id) {
        return child;
      }
      let found: THREE.Object3D | null = null;
      child.traverse((obj) => {
        if (!found && obj.userData && obj.userData.assetId === id) {
          found = obj;
        }
      });
      if (found) return child; // return the top-level group/mesh
    }
    return null;
  }

  function updateAssetObjectPos(id: string, x: number, y: number, z: number) {
    const obj = getAssetObject(id);
    if (obj) {
      if (obj.userData && obj.userData.assetType === "character") {
        obj.position.set(x, y + 0.02, z);
      } else {
        obj.position.set(x, y, z);
      }
    }
    if (selectionRing) selectionRing.position.set(x, y, z);
    if (gizmoGroup) gizmoGroup.position.set(x, y, z);
  }

  function updateAssetObjectRot(
    id: string,
    rx: number,
    ry: number,
    rz: number,
  ) {
    const obj = getAssetObject(id);
    if (obj) {
      obj.rotation.set(
        THREE.MathUtils.degToRad(rx),
        THREE.MathUtils.degToRad(ry),
        THREE.MathUtils.degToRad(rz),
      );
    }
    if (selectionRing) {
      selectionRing.rotation.set(
        THREE.MathUtils.degToRad(rx),
        THREE.MathUtils.degToRad(ry),
        THREE.MathUtils.degToRad(rz),
      );
    }
    if (gizmoGroup) {
      gizmoGroup.rotation.set(
        THREE.MathUtils.degToRad(rx),
        THREE.MathUtils.degToRad(ry),
        THREE.MathUtils.degToRad(rz),
      );
    }
  }

  function getPointerCoords(e: MouseEvent) {
    const rect = renderer.domElement.getBoundingClientRect();
    pointer.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    pointer.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  }

  function pickAsset(): THREE.Mesh | null {
    raycaster.setFromCamera(pointer, camera);
    const hits = raycaster.intersectObjects(assetGroup.children, true);
    if (hits.length > 0) {
      let obj: THREE.Object3D | null = hits[0].object;
      while (obj) {
        if (obj.userData && obj.userData.assetId) {
          return obj as THREE.Mesh;
        }
        obj = obj.parent;
      }
    }
    return null;
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
      e.stopPropagation();
      e.preventDefault();
      gizmoAxis = axis;
      gizmoDragId = selectedId;
      isDragging = false;
      const a = assets.find((a) => a.id === selectedId);
      if (a) {
        gizmoStartPos.set(a.x, a.y, a.z);

        if (rotateMode) {
          // For rotate: compute initial angle between pointer and axis center
          raycaster.setFromCamera(pointer, camera);
          // Plane perpendicular to the rotation axis, through object center
          const axisVec = new THREE.Vector3(
            axis === "x" ? 1 : 0,
            axis === "y" ? 1 : 0,
            axis === "z" ? 1 : 0,
          );
          dragPlane.setFromNormalAndCoplanarPoint(axisVec, gizmoStartPos);
          const intersect = new THREE.Vector3();
          if (raycaster.ray.intersectPlane(dragPlane, intersect)) {
            const local = intersect.clone().sub(gizmoStartPos);
            // Angle in the plane perpendicular to axis
            if (axis === "x") {
              gizmoStartAngle = Math.atan2(local.z, local.y);
            } else if (axis === "y") {
              gizmoStartAngle = Math.atan2(local.x, local.z);
            } else {
              gizmoStartAngle = Math.atan2(local.y, local.x);
            }
          }
        } else {
          // For move: build drag plane as before
          const axisVec = new THREE.Vector3(
            axis === "x" ? 1 : 0,
            axis === "y" ? 1 : 0,
            axis === "z" ? 1 : 0,
          );
          const camDir = new THREE.Vector3();
          camera.getWorldDirection(camDir);
          const temp = new THREE.Vector3().crossVectors(camDir, axisVec);
          const planeNormal = new THREE.Vector3()
            .crossVectors(axisVec, temp)
            .normalize();
          dragPlane.setFromNormalAndCoplanarPoint(planeNormal, gizmoStartPos);
          raycaster.setFromCamera(pointer, camera);
          const intersect = new THREE.Vector3();
          raycaster.ray.intersectPlane(dragPlane, intersect);
          dragOffset.copy(gizmoStartPos).sub(intersect);
        }
      }
      controls.enabled = false;
      return;
    }

    // Check asset (skip body drag in rotate mode)
    const hit = pickAsset();
    if (hit) {
      e.stopPropagation();
      e.preventDefault();
      const id = hit.userData.assetId as string;
      onSelect(id);
      if (rotateMode) {
        // In rotate mode, clicking body just selects — no drag
        return;
      }
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
        const a = assets.find((a) => a.id === gizmoDragId);
        if (a) {
          if (rotateMode) {
            const local = intersect.clone().sub(gizmoStartPos);
            let currentAngle: number;
            if (gizmoAxis === "x") {
              currentAngle = Math.atan2(local.z, local.y);
            } else if (gizmoAxis === "y") {
              currentAngle = Math.atan2(local.x, local.z);
            } else {
              currentAngle = Math.atan2(local.y, local.x);
            }
            let deltaRad = currentAngle - gizmoStartAngle;
            let deltaDeg = (deltaRad * 180) / Math.PI;
            if (a) {
              if (gizmoAxis === "x") {
                a.rotX += deltaDeg;
                updateAssetObjectRot(gizmoDragId, a.rotX, a.rotY, a.rotZ);
              } else if (gizmoAxis === "y") {
                a.rotY += deltaDeg;
                updateAssetObjectRot(gizmoDragId, a.rotX, a.rotY, a.rotZ);
              } else if (gizmoAxis === "z") {
                a.rotZ += deltaDeg;
                updateAssetObjectRot(gizmoDragId, a.rotX, a.rotY, a.rotZ);
              }
            }
            gizmoStartAngle = currentAngle;
            dragMoved = true;
          } else {
            const newCoord = intersect.add(dragOffset);
            if (gizmoAxis === "x") {
              a.x = newCoord.x;
              updateAssetObjectPos(gizmoDragId, a.x, a.y, a.z);
            } else if (gizmoAxis === "y") {
              a.y = newCoord.y;
              updateAssetObjectPos(gizmoDragId, a.x, a.y, a.z);
            } else if (gizmoAxis === "z") {
              a.z = newCoord.z;
              updateAssetObjectPos(gizmoDragId, a.x, a.y, a.z);
            }
            dragMoved = true;
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
        a.x = newX;
        a.z = newZ;
        updateAssetObjectPos(dragId, a.x, a.y, a.z);
        dragMoved = true;
      }
    }
  }

  function onSceneMouseUp(e: MouseEvent) {
    if (cameraFree) {
      onFlyMouseUp();
      return;
    }

    if (gizmoAxis) {
      if (dragMoved && gizmoDragId) {
        const a = assets.find((a) => a.id === gizmoDragId);
        if (a) {
          if (rotateMode) {
            onRotate(gizmoDragId, a.rotX, a.rotY, a.rotZ);
          } else {
            onMove(gizmoDragId, a.x, a.y, a.z);
          }
        }
      }
      gizmoAxis = null;
      gizmoDragId = null;
      isDragging = false;
      dragMoved = false;
      controls.enabled = true;
      return;
    }

    if (dragId) {
      if (dragMoved) {
        const a = assets.find((a) => a.id === dragId);
        if (a) {
          onMove(dragId, a.x, a.y, a.z);
        }
      }
      dragId = null;
      isDragging = false;
      dragMoved = false;
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
    // Capture mousedown before OrbitControls gets it
    renderer.domElement.addEventListener("mousedown", onSceneMouseDown, {
      capture: true,
    });
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
      renderer.domElement.removeEventListener("mousedown", onSceneMouseDown, {
        capture: true,
      } as any);
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
    assets;
    selectedId;
    rotateMode;
    if (scene) {
      buildAssets();
    }
  });
</script>

<div bind:this={container} class="w-full h-full"></div>

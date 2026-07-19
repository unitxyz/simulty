<script module lang="ts">
  export interface Asset {
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
</script>

<script lang="ts">
  import Scene from "$lib/Scene.svelte";

  let spaceWidth = $state(20);
  let spaceLength = $state(20);
  let spaceFit = $state(true);

  let fieldWidth = $state(10);
  let fieldLength = $state(10);
  let fieldFit = $state(true);

  let gridWidth = $state(1);
  let gridLength = $state(1);
  let gridFit = $state(true);
  let gridOpacity = $state(0.5);
  let gridOffsetY = $state(0);
  let grid3D = $state(false);
  let gridVisible = $state(true);

  let cameraFree = $state(false);

  let assets = $state<Asset[]>([]);
  let selectedId = $state<string | null>(null);

  let assetCounter = 0;

  function addAsset() {
    assetCounter++;
    const id = `asset-${assetCounter}`;
    const asset: Asset = {
      id,
      type: "box",
      name: `Палка ${assetCounter}`,
      x: 0,
      y: 0.5,
      z: 0,
      width: 0.1,
      height: 1,
      depth: 0.1,
    };
    assets = [...assets, asset];
    selectedId = id;
  }

  function removeAsset(id: string) {
    assets = assets.filter((a) => a.id !== id);
    if (selectedId === id) selectedId = null;
  }

  function selectAsset(id: string | null) {
    selectedId = id;
  }

  function updateAssetPos(id: string, x: number, y: number, z: number) {
    const a = assets.find((a) => a.id === id);
    if (a) {
      a.x = clamp2(x);
      a.y = clamp2(y);
      a.z = clamp2(z);
    }
  }

  let selectedAsset = $derived(assets.find((a) => a.id === selectedId) ?? null);

  function syncSpace(which: "w" | "l") {
    if (!spaceFit) return;
    if (which === "w") spaceLength = spaceWidth;
    else spaceWidth = spaceLength;
  }
  function syncField(which: "w" | "l") {
    if (!fieldFit) return;
    if (which === "w") fieldLength = fieldWidth;
    else fieldWidth = fieldLength;
  }
  function syncGrid(which: "w" | "l") {
    if (!gridFit) return;
    if (which === "w") gridLength = gridWidth;
    else gridWidth = gridLength;
  }

  function clamp2(v: number): number {
    return Math.round(v * 100) / 100;
  }
</script>

<div class="flex h-screen w-screen overflow-hidden">
  <!-- Panel -->
  <div
    class="w-80 shrink-0 bg-gray-900 p-6 flex flex-col gap-5 border-r border-gray-700 overflow-y-auto"
  >
    <h1 class="text-2xl font-bold text-white">Simulty</h1>
    <p class="text-xs text-gray-500">все значения в метрах</p>

    <!-- Space -->
    <div class="flex flex-col gap-2">
      <span class="text-sm font-medium text-blue-400">Пространство</span>
      <div class="flex items-center gap-2">
        <label class="flex flex-col gap-0.5 flex-1">
          <span class="text-xs text-gray-400">ширина</span>
          <input
            type="number"
            step="0.01"
            min="1"
            bind:value={spaceWidth}
            oninput={() => {
              spaceWidth = clamp2(spaceWidth);
              syncSpace("w");
            }}
            class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-blue-500 outline-none"
          />
        </label>
        <label class="flex flex-col gap-0.5 flex-1">
          <span class="text-xs text-gray-400">длина</span>
          <input
            type="number"
            step="0.01"
            min="1"
            bind:value={spaceLength}
            oninput={() => {
              spaceLength = clamp2(spaceLength);
              syncSpace("l");
            }}
            class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-blue-500 outline-none"
          />
        </label>
        <label class="flex flex-col items-center gap-0.5 pt-4">
          <input
            type="checkbox"
            bind:checked={spaceFit}
            class="accent-blue-500"
          />
          <span class="text-xs text-gray-500">fit</span>
        </label>
      </div>
    </div>

    <!-- Field -->
    <div class="flex flex-col gap-2">
      <span class="text-sm font-medium text-green-400">Поле</span>
      <div class="flex items-center gap-2">
        <label class="flex flex-col gap-0.5 flex-1">
          <span class="text-xs text-gray-400">ширина</span>
          <input
            type="number"
            step="0.01"
            min="1"
            bind:value={fieldWidth}
            oninput={() => {
              fieldWidth = clamp2(fieldWidth);
              syncField("w");
            }}
            class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-green-500 outline-none"
          />
        </label>
        <label class="flex flex-col gap-0.5 flex-1">
          <span class="text-xs text-gray-400">длина</span>
          <input
            type="number"
            step="0.01"
            min="1"
            bind:value={fieldLength}
            oninput={() => {
              fieldLength = clamp2(fieldLength);
              syncField("l");
            }}
            class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-green-500 outline-none"
          />
        </label>
        <label class="flex flex-col items-center gap-0.5 pt-4">
          <input
            type="checkbox"
            bind:checked={fieldFit}
            class="accent-green-500"
          />
          <span class="text-xs text-gray-500">fit</span>
        </label>
      </div>
    </div>

    <!-- Grid -->
    <div class="flex flex-col gap-2">
      <span class="text-sm font-medium text-yellow-400">Сетка</span>
      <div class="flex items-center gap-2">
        <label class="flex flex-col gap-0.5 flex-1">
          <span class="text-xs text-gray-400">ширина ячейки</span>
          <input
            type="number"
            step="0.01"
            min="0.1"
            bind:value={gridWidth}
            oninput={() => {
              gridWidth = clamp2(gridWidth);
              syncGrid("w");
            }}
            class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-yellow-500 outline-none"
          />
        </label>
        <label class="flex flex-col gap-0.5 flex-1">
          <span class="text-xs text-gray-400">длина ячейки</span>
          <input
            type="number"
            step="0.01"
            min="0.1"
            bind:value={gridLength}
            oninput={() => {
              gridLength = clamp2(gridLength);
              syncGrid("l");
            }}
            class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-yellow-500 outline-none"
          />
        </label>
        <label class="flex flex-col items-center gap-0.5 pt-4">
          <input
            type="checkbox"
            bind:checked={gridFit}
            class="accent-yellow-500"
          />
          <span class="text-xs text-gray-500">fit</span>
        </label>
      </div>
      <label class="flex flex-col gap-0.5 mt-1">
        <span class="text-xs text-gray-400">прозрачность</span>
        <input
          type="number"
          step="0.05"
          min="0.05"
          max="1"
          bind:value={gridOpacity}
          oninput={() => {
            gridOpacity = clamp2(gridOpacity);
          }}
          class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-yellow-500 outline-none"
        />
      </label>
      <label class="flex flex-col gap-0.5 mt-1">
        <span class="text-xs text-gray-400">высота над полом (м)</span>
        <input
          type="number"
          step="0.01"
          min="0"
          bind:value={gridOffsetY}
          oninput={() => {
            gridOffsetY = clamp2(gridOffsetY);
          }}
          class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-yellow-500 outline-none"
        />
      </label>
      <label class="flex items-center gap-2 mt-1">
        <input
          type="checkbox"
          bind:checked={gridVisible}
          class="accent-yellow-500"
        />
        <span class="text-xs text-gray-400">показать сетку</span>
      </label>
      <label class="flex items-center gap-2 mt-1">
        <input
          type="checkbox"
          bind:checked={grid3D}
          class="accent-yellow-500"
        />
        <span class="text-xs text-gray-400">3D сетка (кубы)</span>
      </label>
    </div>
    <div class="flex flex-col gap-2">
      <span class="text-sm font-medium text-purple-400">Камера</span>
      <label class="flex items-center gap-2">
        <input
          type="checkbox"
          bind:checked={cameraFree}
          class="accent-purple-500"
        />
        <span class="text-sm text-gray-300"
          >Free mode (WASD + Space/Shift, ЛКМ — осмотр)</span
        >
      </label>
    </div>

    <!-- Assets -->
    <div class="flex flex-col gap-2">
      <div class="flex items-center justify-between">
        <span class="text-sm font-medium text-orange-400">Ассеты</span>
        <button
          onclick={addAsset}
          class="text-xs bg-orange-600 hover:bg-orange-500 text-white px-2 py-1 rounded transition-colors"
        >
          + Палка
        </button>
      </div>

      {#if assets.length === 0}
        <p class="text-xs text-gray-600">нет ассетов</p>
      {:else}
        <div class="flex flex-col gap-1">
          {#each assets as a (a.id)}
            <div
              class="flex items-center justify-between gap-2 px-2 py-1.5 rounded cursor-pointer transition-colors {selectedId ===
              a.id
                ? 'bg-orange-900/60 border border-orange-600'
                : 'bg-gray-800 hover:bg-gray-750 border border-transparent'}"
              onclick={() => selectAsset(a.id)}
            >
              <span class="text-sm text-gray-200 flex-1 truncate">{a.name}</span
              >
              <button
                onclick={(e) => {
                  e.stopPropagation();
                  removeAsset(a.id);
                }}
                class="text-xs text-red-400 hover:text-red-300">✕</button
              >
            </div>
          {/each}
        </div>
      {/if}
    </div>

    <!-- Selected asset settings -->
    {#if selectedAsset}
      <div
        class="flex flex-col gap-3 border border-orange-600/40 rounded p-3 bg-gray-800/50"
      >
        <span class="text-sm font-medium text-orange-400"
          >{selectedAsset.name}</span
        >

        <div class="flex flex-col gap-1">
          <span class="text-xs text-gray-400">Позиция (метры)</span>
          <div class="flex gap-2">
            <label class="flex flex-col gap-0.5 flex-1">
              <span class="text-xs text-gray-500">X</span>
              <input
                type="number"
                step="0.01"
                bind:value={selectedAsset.x}
                class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-orange-500 outline-none"
              />
            </label>
            <label class="flex flex-col gap-0.5 flex-1">
              <span class="text-xs text-gray-500">Y</span>
              <input
                type="number"
                step="0.01"
                bind:value={selectedAsset.y}
                class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-orange-500 outline-none"
              />
            </label>
            <label class="flex flex-col gap-0.5 flex-1">
              <span class="text-xs text-gray-500">Z</span>
              <input
                type="number"
                step="0.01"
                bind:value={selectedAsset.z}
                class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-orange-500 outline-none"
              />
            </label>
          </div>
        </div>

        <div class="flex flex-col gap-1">
          <span class="text-xs text-gray-400">Размеры</span>
          <div class="flex gap-2">
            <label class="flex flex-col gap-0.5 flex-1">
              <span class="text-xs text-gray-500">ширина (X)</span>
              <input
                type="number"
                step="0.01"
                min="0.01"
                bind:value={selectedAsset.width}
                class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-orange-500 outline-none"
              />
            </label>
            <label class="flex flex-col gap-0.5 flex-1">
              <span class="text-xs text-gray-500">высота (Y)</span>
              <input
                type="number"
                step="0.01"
                min="0.01"
                bind:value={selectedAsset.height}
                class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-orange-500 outline-none"
              />
            </label>
            <label class="flex flex-col gap-0.5 flex-1">
              <span class="text-xs text-gray-500">глубина (Z)</span>
              <input
                type="number"
                step="0.01"
                min="0.01"
                bind:value={selectedAsset.depth}
                class="w-full bg-gray-800 text-white text-sm rounded px-2 py-1 border border-gray-700 focus:border-orange-500 outline-none"
              />
            </label>
          </div>
        </div>
      </div>
    {/if}
  </div>

  <!-- 3D Scene -->
  <div class="flex-1 relative">
    <Scene
      {spaceWidth}
      {spaceLength}
      {fieldWidth}
      {fieldLength}
      {gridWidth}
      {gridLength}
      {gridOpacity}
      {gridOffsetY}
      {grid3D}
      {gridVisible}
      {cameraFree}
      {assets}
      {selectedId}
      onSelect={selectAsset}
      onMove={updateAssetPos}
    />
  </div>
</div>

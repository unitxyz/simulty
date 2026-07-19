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

  let cameraFree = $state(false);

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
      <label class="flex flex-col gap-1 mt-1">
        <span class="text-xs text-gray-400"
          >прозрачность: {gridOpacity.toFixed(2)}</span
        >
        <input
          type="range"
          min="0"
          max="1"
          step="0.05"
          bind:value={gridOpacity}
          class="accent-yellow-500"
        />
      </label>
    </div>

    <!-- Camera mode -->
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
      {cameraFree}
    />
  </div>
</div>

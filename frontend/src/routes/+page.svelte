<script lang="ts">
  import Scene from "$lib/Scene.svelte";

  let spaceSize = $state(20);
  let fieldSize = $state(18);
  let gridSize = $state(1);
</script>

<div class="flex h-screen w-screen overflow-hidden">
  <!-- Panel -->
  <div
    class="w-72 shrink-0 bg-gray-900 p-6 flex flex-col gap-6 border-r border-gray-700"
  >
    <h1 class="text-2xl font-bold text-white">Simulty</h1>

    <div class="flex flex-col gap-5">
      <label class="flex flex-col gap-1">
        <span class="text-sm text-gray-300">Размер пространства</span>
        <input
          type="range"
          min="5"
          max="50"
          bind:value={spaceSize}
          class="accent-blue-500"
        />
        <span class="text-xs text-gray-400">{spaceSize}</span>
      </label>

      <label class="flex flex-col gap-1">
        <span class="text-sm text-gray-300">Размер поля</span>
        <input
          type="range"
          min="3"
          max="49"
          bind:value={fieldSize}
          class="accent-green-500"
        />
        <span class="text-xs text-gray-400">{fieldSize}</span>
      </label>

      <label class="flex flex-col gap-1">
        <span class="text-sm text-gray-300">Размер ячейки сетки</span>
        <input
          type="range"
          min="0.5"
          max="5"
          step="0.5"
          bind:value={gridSize}
          class="accent-yellow-500"
        />
        <span class="text-xs text-gray-400">{gridSize}</span>
      </label>
    </div>
  </div>

  <!-- 3D Scene -->
  <div class="flex-1 relative">
    <Scene {spaceSize} {fieldSize} {gridSize} />
  </div>
</div>

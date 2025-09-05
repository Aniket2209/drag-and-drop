<template>
  <div class="flex relative gap-4">
    <div class="type-panel flex gap-2 mb-2">
      <button v-for="type in ['list','edit','detail']" 
              :key="type" 
              :class="['border px-4 py-1 rounded', selectedType===type?'bg-blue-500 text-white':'bg-gray-100']"
              @click="selectedType=type">{{type.toUpperCase()}}
      </button>
    </div>
    <div class="mb-4">
      <label class="font-bold mr-2">Select Platform:</label>
      <select v-model="selectedPlatform" class="border rounded px-2 py-1">
        <option value="web">Web</option>
        <option value="android">Android</option>
      </select>
    </div>
  </div>
  <div class="flex relative gap-4">
    <div class="flex-1 p-4 border border-blue-300 rounded">
      <h2 class="font-bold mb-2">Available Fields</h2>
      <div v-for="(item, idx) in [...items, ...dynamicFields.filter(f => !f.used)]" :key = "idx" 
        :class="[
          'p-2 mb-2 rounded',
          item.groupType === 'container' ? 'bg-green-300' :
          item.groupType === 'row' ? 'bg-teal-200' :
          item.groupType === 'field' ? 'bg-yellow-200' :
          ''
        ]"
        draggable = "true" @dragstart="(e) => onDragStart(item, idx, null, null, e)"
        >
        {{ item.name }}
      </div>
    </div>
    <div class="flex-1 p-4 border border-green-400 rounded"
    @dragover.prevent @drop="onDropRoot($event)"
    :class = "{ 'shake': rootShake }">
      <h2 class="font-bold mb-2">Selected Fields</h2>
        <div v-for = "(container, cIdx) in droppedContainers" :key = "cIdx" class="p-2 mb-2 bg-green-300 rounded"
        draggable = "true"
        @dragstart = "(e) => { e.stopPropagation(); onDragStart(container, cIdx, null, 0); }"
        @dragover.stop.prevent="onDragOverContainer(cIdx, $event)"
        @dragleave.stop="onDragLeaveContainer(cIdx, $event)"
        @drop.stop="onDropToContainerOrSwap(cIdx)"
        :class="{'border-4 border-blue-400': highlightedContainerDropIndex === cIdx}"
        >
          <div v-if="editingContainerId === cIdx">
            <input
              v-model="container.name"
              @blur="editingContainerId = null"
              class="border border-gray-400 rounded px-2 py-1 text-sm w-full"
              autofocus
            />
          </div>
          <div v-else @dblclick="editingContainerId = cIdx">
            {{ container.name }}
          </div>
            <div v-for ="(child, rIdx) in container.items" :key="rIdx" class="bg-teal-200 p-2 rounded min-h-[40px] mb-2"
            :class="{
              'border-4 border-blue-400': highlightedContainerIndex === cIdx && highlightedRowIndex === rIdx
            }"
            :draggable="dragType !== 'field'"
            @dragstart="(e) => { e.stopPropagation(); onDragStart(child, rIdx, null, cIdx); }"
            @dragover.stop.prevent="onDragOverRow(cIdx, rIdx, $event)"
            @drop.stop="onDropToRowOrSwap(cIdx, rIdx)"
            @dragleave="() => onDragLeaveRow(cIdx, rIdx)"
            @click="onRowClick(child)"
            >
              {{  }}
              <div class="flex flex-wrap gap-1">
                <div v-for = "(field, fIdx) in child.items" :key= "fIdx" :style = "{ width: `calc(${100 / child.items.length}% - 0.5rem)` }" class="p-2 rounded"
                :class="field.groupType === 'empty' ? 'bg-white border border-dashed text-gray-400 italic' : 'bg-yellow-200'"
                draggable = "true"
                @dragstart="(e) => { e.stopPropagation(); onDragStart(field, fIdx, rIdx, cIdx); }"
                @dragover.prevent
                @click.stop="onFieldClick(field)"
                >
                <span
                  class="block truncate"
                  :title="field.label?.text"
                  >
                  {{ field.groupType === 'empty' ? '| |' : field.label?.text }}
                </span>
                </div>
              </div>
            </div>
        </div>
      </div>
    <div class="flex-1 flex flex-col gap-4">
      <div
        class="w-full h-48 border border-red-400 bg-gray-200 flex items-center justify-center text-red-700 font-bold text-lg rounded self-start"
        @dragover.prevent
        @drop="onDropToDelete()"
      >
        🗑️ Drag Items here to delete Them
      </div>
      <div
        v-if="selectedField"
        class="p-4 border border-gray-400 rounded bg-white shadow-md max-w-sm"
      >
        <template v-if="selectedPlatform === 'web'">
          <h3 class="font-semibold mb-2">Field Properties</h3>
          <div class="mb-2">
            <label class="block text-sm font-medium text-gray-700">Label:</label>
            <input
              type="text"
              class="mt-1 block w-full border border-gray-300 rounded px-2 py-1"
              v-model="selectedField.label.text"
            />
          </div>
          <div class="mb-2">
            <label class="block text-sm font-medium text-gray-700">Editor Type:</label>
            <input
              type="text"
              class="mt-1 block w-full border border-gray-300 rounded px-2 py-1"
              v-model="selectedField.editorType"
            />
          </div>
          <div class="mb-2">
            <label class="block text-sm font-medium text-gray-700">Width:</label>
            <input
              type="text"
              placeholder="Enter width"
              class="mt-1 block w-full border border-gray-300 rounded px-2 py-1"
              v-model="fieldProperties.width"
            />
          </div>
          <div class="flex items-center space-x-4 mt-3">
            <label class="inline-flex items-center">
              <input
                type="checkbox"
                class="form-checkbox"
                v-model="fieldProperties.required"
              />
              <span class="ml-2 text-sm text-gray-700">Required</span>
            </label>
            <label class="inline-flex items-center">
              <input
                type="checkbox"
                class="form-checkbox"
                v-model="fieldProperties.readonly"
              />
              <span class="ml-2 text-sm text-gray-700">Read Only</span>
            </label>
          </div>
        </template>
        <template v-if="selectedPlatform === 'android'">
          <h3 class="font-semibold mb-2">Field Properties</h3>
          <div class="mb-2">
            <label class="block text-sm font-medium">Label:</label>
            <input
              type="text"
              class="mt-1 block w-full border rounded px-2 py-1"
              v-model="fieldProperties.label.text"
            />
          </div>
          <div class="mb-2">
            <label class="block text-sm font-medium">Editor Type:</label>
            <input
              type="text"
              class="mt-1 block w-full border rounded px-2 py-1"
              v-model="fieldProperties.editorType"
            />
          </div>
          <div class="mb-2">
            <label class="block text-sm font-medium">Size:</label>
            <input v-model="fieldProperties.size" type="number" class="mt-1 block w-full border rounded px-2 py-1" />
          </div>
          <div class="mb-2">
            <label class="block text-sm font-medium">Font Size:</label>
            <input v-model="fieldProperties.font_size" type="number" class="mt-1 block w-full border rounded px-2 py-1" />
          </div>
          <div class="mb-2">
            <label class="block text-sm font-medium">Font Weight:</label>
            <input v-model="fieldProperties.font_weight" type="text" placeholder="normal, bold, semi-bold" class="mt-1 block w-full border rounded px-2 py-1" />
          </div>
          <h4 class="font-semibold mt-4 mb-2">Padding</h4>
          <div class="flex gap-2 mb-2">
            <div>
              <label class="block text-xs">Top</label>
              <input v-model="fieldProperties.padding_top" type="number" class="w-16 border rounded px-2 py-1" />
            </div>
            <div>
              <label class="block text-xs">Right</label>
              <input v-model="fieldProperties.padding_right" type="number" class="w-16 border rounded px-2 py-1" />
            </div>
            <div>
              <label class="block text-xs">Bottom</label>
              <input v-model="fieldProperties.padding_bottom" type="number" class="w-16 border rounded px-2 py-1" />
            </div>
            <div>
              <label class="block text-xs">Left</label>
              <input v-model="fieldProperties.padding_left" type="number" class="w-16 border rounded px-2 py-1" />
            </div>
          </div>
        </template>
        <button
          @click="applyFieldProperties"
          class="mt-4 bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded"
        >
          Apply
        </button>
      </div>
      <div v-if="selectedRow && selectedPlatform === 'android'" class="p-4 border rounded-lg shadow bg-white w-full">
        <h3 class="text-lg font-semibold mb-4">Row Properties</h3>
        <div class="space-y-3">
          <!-- Common -->
          <div>
            <label class="block text-sm font-medium">Horizontal Align</label>
            <select class="mt-1 block w-full border rounded px-2 py-1" v-model="rowProperties.horizontal_alignment">
              <option value="start">Start</option>
              <option value="center">Center</option>
              <option value="end">End</option>
            </select>
          </div>

          <div>
            <fieldset class="block text-sm font-medium">
              <legend>Padding</legend>
              <div class="flex gap-2 mb-2">
                <input v-model.number="rowProperties.padding_top" class="mt-1 block w-full border rounded px-2 py-1" type="number" placeholder="Top" />
                <input v-model.number="rowProperties.padding_right" class="mt-1 block w-full border rounded px-2 py-1" type="number" placeholder="Right" />
                <input v-model.number="rowProperties.padding_bottom" class="mt-1 block w-full border rounded px-2 py-1" type="number" placeholder="Bottom" />
                <input v-model.number="rowProperties.padding_left" class="mt-1 block w-full border rounded px-2 py-1" type="number" placeholder="Left" />
              </div>
            </fieldset>
          </div>

          <!-- Specific: Non-template -->
          <template v-if="rowProperties.editorType === 'displayTextInitials'">
            <div>
              <label class="block text-sm font-medium">Vertical Align</label>
              <select class="mt-1 block w-full border rounded px-2 py-1" v-model="rowProperties.vertical_alignment">
                <option value="top">Top</option>
                <option value="center">Center</option>
                <option value="bottom">Bottom</option>
              </select>
            </div>

            <div class="flex gap-2 mb-2">
              <div class="flex-1">
                <label class="block text-sm font-medium">Size</label>
                <input v-model.number="rowProperties.size" class="mt-1 block w-full border rounded px-2 py-1" type="number" />
              </div>

              <div class="flex-1">
                <label class="block text-sm font-medium">Background Color</label>
                <input v-model="rowProperties.bgColor" class="mt-1 block w-full border rounded px-2 py-1" type="color" />
              </div>

              <div class="flex-1">
                <label class="block text-sm font-medium">Text Color</label>
                <input v-model="rowProperties.textColor" class="mt-1 block w-full border rounded px-2 py-1" type="color" />
              </div>
            </div>

            <div>
              <label class="block text-sm font-medium">Icon</label>
              <input v-model="rowProperties.icon" class="mt-1 block w-full border rounded px-2 py-1" type="text" />
            </div>

            <div>
              <label class="block text-sm font-medium">Action Type</label>
              <input v-model="rowProperties.actionType" class="mt-1 block w-full border rounded px-2 py-1" type="text" />
            </div>

            <div>
              <label class="block text-sm font-medium">Action Destination</label>
              <input v-model="rowProperties.actionDestination" class="mt-1 block w-full border rounded px-2 py-1" type="text" />
            </div>
          </template>

          <!-- Specific: Template -->
          <template v-else-if="rowProperties.editorType === 'group'">
            <div>
              <label class="block text-sm font-medium">Font Size</label>
              <input v-model.number="rowProperties.font_size" class="mt-1 block w-full border rounded px-2 py-1" type="number" />
            </div>

            <div>
              <label class="block text-sm font-medium">Font Weight</label>
              <select class="mt-1 block w-full border rounded px-2 py-1" v-model="rowProperties.font_weight">
                <option value="normal">Normal</option>
                <option value="semi-bold">Semi-Bold</option>
                <option value="bold">Bold</option>
              </select>
            </div>

            <div>
              <label class="block text-sm font-medium">Text Align</label>
              <select class="mt-1 block w-full border rounded px-2 py-1" v-model="rowProperties.text_align">
                <option value="start">Start</option>
                <option value="center">Center</option>
                <option value="end">End</option>
              </select>
            </div>
          </template>
        </div>
        <button class="mt-4 px-4 py-2 rounded bg-blue-500 text-white" @click="applyRowProperties">Apply</button>
      </div>
    </div>
  </div>
  <div class="flex gap-4 mt-4">
    <button
      @click="saveAndDeploy"
      class="bg-green-500 hover:bg-green-600 text-white px-4 py-1 rounded text-sm"
    >
      💾 Save and Deploy
    </button>
  </div>
  <div class="flex relative gap-4">
    <pre class="w-1/2 text-xs overflow-auto mt-4 bg-white p-2 border max-h-64">
      {{ JSON.stringify(droppedContainers, null, 2) }}
    </pre>
  </div>
</template>

<script setup>
  import { ref, onMounted, watch } from 'vue';
  import customFieldJSON from './assets/customFields.json';
  import axios from 'axios';

  const items = ref([
    { name: 'Row' , groupType: "row"},
    { name: 'Container' , groupType: "container"},
    { name: '| | Empty Slot' , groupType: "empty"}
  ]);
  const droppedContainers = ref([]);

  let draggedContainerIndex = null;
  let draggedRowIndex = null;
  let draggedFieldIndex = null;
  let dragType = null;
  let originalContainerIndex = null;
  let originalRowIndex = null;
  let draggedItem = null;
  let isFromPalette = false;
  let dragCursorEl = null;

  const containerCount = ref(0)
  const rowCount = ref(1)
  const fieldCount = ref(1)
  const selectedType = ref('list');
  const selectedPlatform = ref('web');
  const saving = ref(false);
  const rootShake = ref(false);
  const highlightedContainerIndex = ref(null);
  const highlightedRowIndex = ref(null);
  const highlightedContainerDropIndex = ref(null);

  function updateColCounts() {
    for (const container of droppedContainers.value) {
      if (!((selectedType.value === 'detail' && selectedPlatform.value === 'web') || (selectedPlatform.value === 'android' && selectedType.value === 'list'))) {
        container.colCount = container.items?.length || 0;
      } else {
        delete container.colCount; // ensure removed
      }

      for (const row of container.items || []) {
        if (!((selectedType.value === 'detail' && selectedPlatform.value === 'web') || (selectedPlatform.value === 'android' && selectedType.value === 'list'))) {
          row.colCount = row.items?.length || 0;
        } else {
          delete row.colCount; // ensure removed
        }
      }
    }
  }

  function onDragStart(item, itemIndex, parentRowIndex = null, parentContainerIndex = null, event) {
    dragType = item.groupType;

    isFromPalette = (parentRowIndex === null && parentContainerIndex === null);
    // We decide it's from the palette if parent indexes are null
    if (isFromPalette) {
      draggedItem = { ...item }; // shallow clone is enough for palette
    } else {
      draggedItem = item;
    }

    // Reset indexes
    draggedContainerIndex = null;
    draggedRowIndex = null;
    draggedFieldIndex = null;
    originalContainerIndex = null;
    originalRowIndex = null;

    switch (dragType) {
      case 'container':
        draggedContainerIndex = itemIndex;
        originalContainerIndex = itemIndex;
        break;
      case 'row':
        draggedRowIndex = itemIndex;
        draggedContainerIndex = parentContainerIndex;
        originalContainerIndex = parentContainerIndex;
        originalRowIndex = itemIndex;
        break;
      case 'field':
      case 'empty':
        draggedFieldIndex = itemIndex;
        draggedRowIndex = parentRowIndex;
        draggedContainerIndex = parentContainerIndex;
        originalContainerIndex = parentContainerIndex;
        originalRowIndex = parentRowIndex;
        break;
      default:
        console.warn("Unknown drag type:", dragType);
        break;
    }
    console.log(`Drag Start - Type: ${dragType}, ContainerIdx: ${draggedContainerIndex}, RowIdx: ${draggedRowIndex}, FieldIdx: ${draggedFieldIndex}`);
    event.dataTransfer.setDragImage(new Image(), 0, 0);

    dragCursorEl = document.createElement("div");
    dragCursorEl.className = "drag-cursor";
    dragCursorEl.innerText = item.name || item.groupType;
    dragCursorEl.setAttribute("style", getCursorStyle(item.groupType));
    document.body.appendChild(dragCursorEl);

    document.addEventListener("dragover", moveCustomCursor);
  }

  function onDropRoot(e) {
    if (selectedType.value === 'list' && selectedPlatform.value === 'android')
    {
      rootShake.value = true;
      setTimeout(() => { rootShake.value = false;}, 300);
      return;
    }
    else
    {
      // Only allow dropping new containers (created from available items panel)
      console.log(
        'onDropRoot:',
        'dragType:', dragType,
        'draggedItem._fresh:', draggedItem?._fresh,
        'isFromPalette:', isFromPalette
      );
      if (e.target !== e.currentTarget) return;

      if (!(dragType === 'container' && isFromPalette)) {
        rootShake.value = true;
        setTimeout(() => {
          rootShake.value = false;
        }, 300);
        return;
      }
      containerCount.value++;
      droppedContainers.value.push(createContainer());
      updateColCounts();
      // Clear drag state
      draggedItem = null;
      dragType = null;
      // Clear stored source indexes instead of IDs
      draggedContainerIndex = null;
      draggedRowIndex = null;
      draggedFieldIndex = null;
      originalContainerIndex = null;
      originalRowIndex = null;
      isFromPalette = false;
      clearDragState();
    }
  }

  function onDropToContainerOrSwap(targetContainerIdx) {
    if (!draggedItem || !dragType || !draggedItem.groupType) return;

    if (dragType === 'row' && draggedItem.groupType === 'row') {
      // Move or add the dragged row into the target container
      onDropToContainer(targetContainerIdx);
    } else if (dragType === 'container' && draggedItem.groupType === 'container') {
      // Swap two containers based on their index
      onDropToSwapContainer(targetContainerIdx);
    } else {
      rootShake.value = true;
      setTimeout(() => {
        rootShake.value = false;
      }, 300);
      console.warn(`Dropped ${dragType} with mismatched type`);
    }
  }

  function onDropToRowOrSwap(targetContainerIdx, targetRowIdx) {
    if (!draggedItem || !dragType || !draggedItem.groupType) return;

    if ((dragType === 'field' || dragType === 'empty') && (draggedItem.groupType === 'field' || draggedItem.groupType === 'empty')) {
      onDropToRow(targetContainerIdx, targetRowIdx);
    }
    else if (dragType === 'row' && draggedItem.groupType === 'row') {
      onDropToSwapRow(targetContainerIdx, targetRowIdx);
    }
    else {
      rootShake.value = true;
      setTimeout(() => {
        rootShake.value = false;
      }, 300);
      console.warn(`Dropped ${dragType} with mismatched type`);
    }
    updateColCounts();
  }

  function onDropToContainer(targetContainerIndex) {
    if (!draggedItem || dragType !== 'row') return;

    const targetContainer = droppedContainers.value[targetContainerIndex];
    if (!targetContainer) return;

    if (draggedItem && isFromPalette) {
      rowCount.value++;
      if(targetContainer.template === "expandable")
      {
        targetContainer.items.push(createRow(true));
      }
      else{
        targetContainer.items.push(createRow(false));
      }
    }
    else {
      if (
        typeof originalContainerIndex === 'number' &&
        typeof originalRowIndex === 'number' &&
        originalContainerIndex >= 0 &&
        originalRowIndex >= 0
      ) {
        const originalContainer = droppedContainers.value[originalContainerIndex];
        if (!originalContainer) return;

        const [movedRow] = originalContainer.items.splice(originalRowIndex, 1);
        targetContainer.items.push(movedRow);
      }
    }
    updateColCounts();
    draggedItem = null;
    dragType = null;
    originalContainerIndex = null;
    originalRowIndex = null;
    isFromPalette = false;
    highlightedContainerDropIndex.value = null;
    clearDragState();
  }

  function onDropToSwapContainer(targetContainerIndex) {
    if (dragType !== 'container') return;

    // draggedContainerIndex is set during drag start
    if (draggedContainerIndex === null) return;
    if (targetContainerIndex === null) return;
    if (draggedContainerIndex === targetContainerIndex) return;

    const arr = droppedContainers.value;

    // Swap containers at draggedContainerIndex and targetContainerIndex
    const temp = arr[draggedContainerIndex];
    arr[draggedContainerIndex] = arr[targetContainerIndex];
    arr[targetContainerIndex] = temp;
    updateColCounts();
    // Cleanup drag state
    draggedContainerIndex = null;
    draggedRowIndex = null;
    draggedFieldIndex = null;
    dragType = null;
    originalContainerIndex = null;
    originalRowIndex = null;
    draggedItem = null;
    isFromPalette = false;
    clearDragState();
  }

  function onDropToRow(targetContainerIndex, targetRowIndex) {
    if (dragType !== 'field' && dragType !== 'empty') return;

    const targetContainer = droppedContainers.value[targetContainerIndex];
    if (!targetContainer) return;

    const targetRow = targetContainer.items[targetRowIndex];
    if (!targetRow) return;

    if (!Array.isArray(targetRow.items)) targetRow.items = [];

    if (draggedItem && isFromPalette) {
      // Dragging new dynamic field
      targetRow.items.push(createField());

      // Mark dynamic field as used
      const identifier = getFieldIdentifier(draggedItem);
      const matchingDynamic = dynamicFields.value.find(f => getFieldIdentifier(f) === identifier);
      if (matchingDynamic) matchingDynamic.used = true;
    }
    else if (
      typeof originalContainerIndex === 'number' &&
      typeof originalRowIndex === 'number' &&
      typeof draggedFieldIndex === 'number'
    ) {
      // Moving existing field to new row
      const originalContainer = droppedContainers.value[originalContainerIndex];
      if (!originalContainer) return;

      const originalRow = originalContainer.items[originalRowIndex];
      if (!originalRow) return;

      // Remove field from original row
      const [movedField] = originalRow.items.splice(draggedFieldIndex, 1);

      // Add it to target row
      targetRow.items.push(movedField);
    } 
    else {
      // Add new default field or empty slot
      fieldCount.value++;
      targetRow.items.push(createField());
    }
    updateColCounts();
    draggedItem = null;
    dragType = null;
    draggedFieldIndex = null;
    draggedRowIndex = null;
    draggedContainerIndex = null;
    originalContainerIndex = null;
    originalRowIndex = null;
    isFromPalette = false;
    highlightedContainerIndex.value = null;
    highlightedRowIndex.value = null;
    clearDragState();
  }

  function onDropToSwapRow(targetContainerIdx, targetRowIdx) {
    if (dragType !== 'row') return;
    if (draggedContainerIndex === null || draggedRowIndex === null) return;
    if (targetContainerIdx === null || targetRowIdx === null) return;

    const sourceContainer = droppedContainers.value[draggedContainerIndex];
    if (!sourceContainer) return;

    // Take row from source
    const [movingRow] = sourceContainer.items.splice(draggedRowIndex, 1);

    if (draggedContainerIndex === targetContainerIdx) {
      // 🔹 Case 1: Same container → reorder by inserting at new index
      const targetRows = droppedContainers.value[targetContainerIdx].items;

      // Adjust index if dragging forward (because splice removed earlier element)
      const adjustedIndex = draggedRowIndex < targetRowIdx ? targetRowIdx - 1 : targetRowIdx;

      targetRows.splice(adjustedIndex, 0, movingRow);
    } else {
      // 🔹 Case 2: Different container → insert into target
      const targetContainer = droppedContainers.value[targetContainerIdx];
      if (!targetContainer) return;

      targetContainer.items.splice(targetRowIdx, 0, movingRow);
    }
    updateColCounts();
    // ✅ Cleanup
    draggedRowIndex = null;
    draggedContainerIndex = null;
    draggedItem = null;
    dragType = null;
    originalContainerIndex = null;
    originalRowIndex = null;
    isFromPalette = false;
    clearDragState();
  }

  function onDropToDelete() {
    if (!draggedItem || !dragType) return;

    if (dragType === 'row') {
      if (typeof originalContainerIndex === 'number' && typeof originalRowIndex === 'number') {
        const container = droppedContainers.value[originalContainerIndex];
        if (container) {
          container.items.splice(originalRowIndex, 1);
        }
      }
    } 
    else if (dragType === 'field' || dragType === 'empty') {
      if (
        typeof originalContainerIndex === 'number' &&
        typeof originalRowIndex === 'number' &&
        typeof draggedFieldIndex === 'number'
      ) {
        const container = droppedContainers.value[originalContainerIndex];
        if (container) {
          const row = container.items[originalRowIndex];
          if (row && Array.isArray(row.items)) {
            row.items.splice(draggedFieldIndex, 1);
          }
        }
      }
    } 
    else if (dragType === 'container') {
      if (typeof draggedContainerIndex === 'number') {
        droppedContainers.value.splice(draggedContainerIndex, 1);
      }
    }
    updateColCounts();
    // Clear drag state
    draggedItem = null;
    dragType = null;
    draggedContainerIndex = null;
    draggedRowIndex = null;
    draggedFieldIndex = null;
    originalContainerIndex = null;
    originalRowIndex = null;
  }

  const editingContainerId = ref(null);
  const dynamicFields = ref([]);

  onMounted(() => {
    const fieldsFromFile = extractFieldsFromJSON(customFieldJSON);
      dynamicFields.value.push(...fieldsFromFile);
  });

  function getFieldIdentifier(field) {
    return field?.label?.text || field?.name || field?.dataField || null;
  }

  function extractFieldsFromJSON(groups) {
    const items = [];

    groups.forEach(group => {
      if (Array.isArray(group.items)) {
        group.items.forEach(item => {
          if (item.itemType === 'empty') return;
          const id = getFieldIdentifier(item) || "Unnamed Field";
          items.push({
            name: id,
            groupType: 'field',
            dataField: item.dataField,
            editorType: item.editorType,
            label: { text:id },
            used: false
          });
        });
      }
    });
    return items;
  }

  function extractUsedFieldNames(structure) {
    const used = new Set();
    structure.forEach(container => {
      (container.items || []).forEach(row => {
        (row.items || []).forEach(field => {
          const id = getFieldIdentifier(field);
          if (id) used.add(id);
        })
      })
    })
    return used;
  }

  function syncDynamicFieldsWithLayout() {
    const fieldsFromFile = extractFieldsFromJSON(customFieldJSON);
    const usedFields = extractUsedFieldNames(droppedContainers.value);
    dynamicFields.value = fieldsFromFile
      .map(f => ({ ...f, used: false }))
      .filter(field => !usedFields.has(getFieldIdentifier(field)));
  }

  const selectedField = ref(null);
  const fieldProperties = ref({
    width: '',
    required: false,
    readonly: false,
    size: '',
    font_size: '',
    font_weight: '',
    padding_top: 0,
    padding_right: 0,
    padding_bottom: 0,
    padding_left: 0,
  });

  watch([selectedType, selectedPlatform], () => {
    selectedField.value = null;
    selectedRow.value = null;
  });

  function onFieldClick(field)
  {
    selectedField.value = field;
    selectedRow.value = null;

    if (!field.label) {
      field.label = { text:""};
    }
    
    if (!field.editorType) {
      field.editorType = "";
    }

    fieldProperties.value = {
      width: field.width || '',
      required: !!field.required,
      readonly: !!field.readonly,
      label: { text: field.label.text || "" },
      editorType: field.editorType || "",
      size: field.size || '',
      font_size: field.font_size || '',
      padding_top: field.padding?.top ?? 0,
      padding_right: field.padding?.right ?? 0,
      padding_bottom: field.padding?.bottom ?? 0,
      padding_left: field.padding?.left ?? 0,
    };
  }

  function applyFieldProperties()
  {
    if (!selectedField.value) return;

    if (selectedPlatform.value === "web") {
      selectedField.value.width = fieldProperties.value.width || undefined;
      selectedField.value.required = !!fieldProperties.value.required;
      selectedField.value.readonly = !!fieldProperties.value.readonly;
    }
    else {
      delete selectedField.value.width;
      delete selectedField.value.required;
      delete selectedField.value.readonly;
    }
    if (selectedPlatform.value === "android") {
      selectedField.value.label = { text: fieldProperties.value.label.text || "" };
      selectedField.value.editorType = fieldProperties.value.editorType || "";
      if (selectedPlatform.value.size !== '') selectedField.value.size = Number(fieldProperties.value.size);
      else delete selectedField.value.size;

      if (fieldProperties.value.font_size !== '') selectedField.value.font_size = Number(fieldProperties.value.font_size);
      else delete selectedField.value.font_size;

      if (fieldProperties.value.font_weight !== '') selectedField.value.font_weight = Number(fieldProperties.value.font_weight);
      else delete selectedField.value.font_weight;

      const paddings = ['padding_top', 'padding_right', 'padding_bottom', 'padding_left'].map(d => Number(fieldProperties.value[d]) || 0);
      if (paddings.some(val => val !== 0)) {
        selectedField.value.padding = {
          top: paddings[0],
          right: paddings[1],
          bottom: paddings[2],
          left: paddings[3],
        };
      } 
      else {
        delete selectedField.value.padding;
      }
    } 
    else {
      // Remove Android-specific fields for web or other platforms
      delete selectedField.value.size;
      delete selectedField.value.font_size;
      delete selectedField.value.font_weight;
      delete selectedField.value.padding;
    }

    alert(`Field "${selectedField.value.label?.text}" updated successfully.`);
  }

  const selectedRow = ref(null)
  const rowProperties = ref(null)
  function onRowClick(row) {
    selectedRow.value = row;
    selectedField.value = null;

    if (!row.label) row.label = { text: '' };

    const base = {
      editorType: row.editorType || "group",
      label: row.label?.text || "",
      horizontal_alignment: row.horizontal_alignment || "start",
      padding_top: row.padding?.top ?? 0,
      padding_right: row.padding?.right ?? 0,
      padding_bottom: row.padding?.bottom ?? 0,
      padding_left: row.padding?.left ?? 0,
    };

    if (row.editorType === "displayTextInitials") {
      rowProperties.value = {
        ...base,
        vertical_alignment: row.vertical_alignment || "top",
        size: row.size || 30,
        bgColor: row.bgColor || "#FFFFFFFF",
        textColor: row.textColor || "#FF000000",
        icon: row.icon || null,
        actionType: row.action?.type || null,
        actionDestination: row.action?.destination || null,
      };
    } else if (row.editorType === "group") {
      rowProperties.value = {
        ...base,
        font_size: row.font_size || 14,
        font_weight: row.font_weight || "normal",
        text_align: row.text_align || "start",
      };
    } else {
      rowProperties.value = base; // fallback
    }
  }

  function applyRowProperties() {
    if (!selectedRow.value) return;

    // Common
    selectedRow.value.editorType = rowProperties.value.editorType;
    selectedRow.value.label = { text: rowProperties.value.label };
    selectedRow.value.horizontal_alignment = rowProperties.value.horizontal_alignment;
    selectedRow.value.padding = {
      top: rowProperties.value.padding_top,
      right: rowProperties.value.padding_right,
      bottom: rowProperties.value.padding_bottom,
      left: rowProperties.value.padding_left,
    };

    // Type-specific
    if (rowProperties.value.editorType === "displayTextInitials") {
      selectedRow.value.vertical_alignment = rowProperties.value.vertical_alignment;
      selectedRow.value.size = rowProperties.value.size;
      selectedRow.value.bgColor = rowProperties.value.bgColor;
      selectedRow.value.textColor = rowProperties.value.textColor;
      selectedRow.value.icon = rowProperties.value.icon;
      selectedRow.value.action = {
        type: rowProperties.value.actionType,
        destination: rowProperties.value.actionDestination,
      };
    } else if (rowProperties.value.editorType === "group") {
      selectedRow.value.font_size = rowProperties.value.font_size;
      selectedRow.value.font_weight = rowProperties.value.font_weight;
      selectedRow.value.text_align = rowProperties.value.text_align;
    }

    alert("Row properties updated!");
  }

  axios.defaults.baseURL = 'http://127.0.0.1:8000';

  async function loadDropzoneStructure() {
    try {
      const res = await axios.get(`http://127.0.0.1:8000/api/layouts/${selectedType.value}/${selectedPlatform.value}`);
      let data = res.data;
      if(selectedType.value === 'list' && selectedPlatform.value === 'android')
      {
        const valid = Array.isArray(data) && data.length === 2 && data.every(c => c.groupType === 'container');
        if (!valid) {
          initAndroidListLayout();
          return;
        }
      }
      if (!Array.isArray(data)) {
        console.warn("Expected array but got:", data);
        data = [];
      }
      droppedContainers.value = data;
      syncDynamicFieldsWithLayout();
    }
    catch (e) {
      console.error("API load failed:", e);
      if (selectedType.value === 'list' && selectedPlatform.value === 'android') {
        initAndroidListLayout();
      }
      else {
        droppedContainers.value = [];
      }
    }
  }

  onMounted(loadDropzoneStructure);
  watch([selectedType, selectedPlatform], loadDropzoneStructure);
  watch(droppedContainers, syncDynamicFieldsWithLayout, { deep: true });

  async function saveAndDeploy() {
    try {
      saving.value = true;
      const type = selectedType.value;
      const platform = selectedPlatform.value;
      await axios.put(`http://127.0.0.1:8000/api/layouts/${type}/${platform}`, droppedContainers.value);
      alert("Layout saved and deployed!");
      await loadDropzoneStructure();  // Reload after save
    } catch (error) {
      alert("Failed to save layout.");
    } finally {
      saving.value = false;
    }
  }

  function createContainer() 
  {
    if(selectedPlatform.value === 'web')
    {
      if(selectedType.value === 'list' || selectedType.value === 'edit')
      {
        return {
          name: `Container${containerCount.value}`,
          itemType: 'group',
          groupType: 'container',
          colCount: 0,
          items: []
        };
      }
      else if(selectedType.value === 'detail')
      {
        return {
          name: `Container${containerCount.value}`,
          itemType: 'group',
          groupType: 'container',
          items: []
        };
      }
    }
    else if(selectedPlatform.value === 'android')
    {
      if(selectedType.value === 'edit')
      {
        console.log("Sorry Not Supported Yet!");
      }
      else if(selectedType.value === 'detail')
      {
        console.log("Sorry Not Supported Yet!");
      }
    }
  }

  function createRow(hasTemplate)
  {
    if(selectedPlatform.value === 'web')
    {
      if(selectedType.value === 'list' || selectedType.value === 'edit')
      {
        return {
          itemType: 'group',
          groupType: 'row',
          colCount: 0,
          items: []
        };
      }
      else if(selectedType.value === 'detail')
      {
        return {
          itemType: 'group',
          groupType: 'row',
          items: []
        };
      }
    }
    else if(selectedPlatform.value === 'android')
    {
      if(selectedType.value === 'list')
      {
        if(hasTemplate)
        {
          return {
            editorType: "group",
            font_size: 18,
            font_weight: "normal",
            groupType: 'row',
            horizontal_align: "center",
            font_weight: "normal",
            text_align: "center",
            padding: 0,
            label: { text: "Email" },
            items: []
          };
        }
        else
        {
          return {
            editorType: "displayTextInitials",
            groupType: "row",
            vertical_alignment: "top",
            horizontal_alignment: "start",
            icon: "location",
            size: 30,
            action: {
              type: "navigate",
              destination: "",
              actionType: ""
            },
            padding: {
              top: 10,
              bottom: 10
            },
            bgColor: "#FF5599FF",
            textColor: "#FFFFFFFF",
            label: {
              text: "Email"
            },
            items: []
          };
        }
      }
      else if(selectedType.value === 'edit')
      {
        console.log("Sorry Not Supported Yet!");
      }
      else if(selectedType.value === 'detail')
      {
        console.log("Sorry Not Supported Yet!");
      }
    }
  }

  function createField()
  {
    if(selectedPlatform.value === 'web')
    {
      if(selectedType.value === 'list')
      {
        return {
          groupType: draggedItem.groupType || 'field',
          dataField: draggedItem.dataField,
          editorType: draggedItem.editorType,
          label: { text: draggedItem.name },   // migrate name → label.text
          visible: true
        };
      }
      else if(selectedType.value === 'edit' || selectedType.value === 'detail')
      {
        return {
          groupType: draggedItem.groupType || 'field',
          dataField: draggedItem.dataField,
          editorType: draggedItem.editorType,
          label: { text: draggedItem.name },   // migrate name → label.text
        };
      }
    }
    else if(selectedPlatform.value === 'android')
    {
      if(selectedType.value === 'list')
      {
        return {
          editorType: draggedItem.editorType,
          groupType: draggedItem.groupType || 'field',
          icon: "call_color",
          size: 28,
          font_size: 18,
          font_weight: "semi-bold",
          action: {
            type: ""
          },
          label: {
            text: draggedItem.name
          },
          padding: {
            right: 7
          }
        };
      }
      else if(selectedType.value === 'edit')
      {
        console.log("Sorry Not Supported Yet!");
      }
      else if(selectedType.value === 'detail')
      {
        console.log("Sorry Not Supported Yet!");
      }
    }
  }
  function moveCustomCursor(e) {
    if (dragCursorEl) {
      dragCursorEl.style.left = e.pageX + "px";
      dragCursorEl.style.top = e.pageY + "px";
    }
  }

  function clearDragState() {
    // call this in onDropRoot, onDropToContainer, onDropToRow etc.
    document.removeEventListener("dragover", moveCustomCursor);
    if (dragCursorEl) {
      dragCursorEl.remove();
      dragCursorEl = null;
    }
    document.body.classList.remove("dragging");
  }

  function getCursorStyle(groupType) {
    switch (groupType) {
      case "container":
        return "background: #86efac; color: black;"; // Tailwind green-300
      case "row":
        return "background: #7dd3fc; color: black;"; // Tailwind sky-300
      case "field":
        return "background: #fde68a; color: black;"; // Tailwind yellow-300
      case "empty":
        return "background: #FFFFFF; color: black;";
      default:
        return "background: rgba(59, 130, 246, 0.9); color: white;"; // fallback blue
    }
  }

  function onDragOverRow(containerIdx, rowIdx, event) {
    if (dragType !== 'field' && dragType !== 'empty') {
      highlightedContainerIndex.value = null;
      highlightedRowIndex.value = null;
      return;
    }
    highlightedContainerIndex.value = containerIdx;
    highlightedRowIndex.value = rowIdx;
  }

  function onDragLeaveRow(containerIdx, rowIdx) {
    if (
      highlightedContainerIndex.value === containerIdx &&
      highlightedRowIndex.value === rowIdx
    ) {
      highlightedContainerIndex.value = null;
      highlightedRowIndex.value = null;
    }
  }

  function onDragOverContainer(cIdx, event) {
    // Only highlight when dragging a row
    if (dragType !== 'row') {
      highlightedContainerDropIndex.value = null;
      return;
    }
    highlightedContainerDropIndex.value = cIdx;
  }

  function onDragLeaveContainer(cIdx, event) {
    if (highlightedContainerDropIndex.value === cIdx) {
      highlightedContainerDropIndex.value = null;
    }
  }

  onMounted(() => {
    document.addEventListener("drop", clearDragState);
    document.addEventListener("dragend", clearDragState);
  });

  function initAndroidListLayout() {
    droppedContainers.value = JSON.parse(JSON.stringify([
      {
        name: "Default Container",
        editorType: "group",
        groupType: "container",
        horizontal_align: "center",
        vertical_align: "center",
        items: []
      },
      {
        name: "Expendable Container",
        editorType: "group",
        groupType: "container",
        template: "expandable",
        horizontal_alignment: "start",
        items: []
      }
    ]));
  }
</script>

<style>
  .drag-cursor {
    position: absolute;
    pointer-events: none;
    z-index: 9999;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 12px;
    transform: translate(-50%, -50%);
  }

  @keyframes shake {
    0%   { transform: translateX(0); }
    20%  { transform: translateX(-5px); }
    40%  { transform: translateX(5px); }
    60%  { transform: translateX(-5px); }
    80%  { transform: translateX(5px); }
    100% { transform: translateX(0); }
  }

  .shake {
    animation: shake 0.3s ease-in-out;
  }

  [draggable="true"] {
    cursor: grab !important;
  }
</style>
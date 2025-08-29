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
          'p-2 mb-2 cursor-move rounded',
          item.groupType === 'container' ? 'bg-green-300' :
          item.groupType === 'row' ? 'bg-teal-200' :
          item.groupType === 'field' ? 'bg-yellow-200' :
          ''
        ]"
        draggable = "true" @dragstart="onDragStart(item, idx)"
        >
        {{ item.name }}
      </div>
    </div>
    <div class="flex-1 p-4 border border-green-400 rounded"
    @dragover.prevent @drop="onDropRoot($event)">
      <h2 class="font-bold mb-2">Selected Fields</h2>
        <div v-for = "(container, cIdx) in droppedContainers" :key = "cIdx" class="p-2 mb-2 bg-green-300 rounded"
        draggable = "true"
        @dragstart = "(e) => { e.stopPropagation(); onDragStart(container, cIdx, null, 0); }"
        @dragover.stop.prevent
        @drop.stop="onDropToContainerOrSwap(cIdx)"
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
            <div v-for ="(child, rIdx) in container.items" :key="rIdx" class="bg-teal-200 p-2 rounded cursor-move min-h-[40px]"
            :draggable="dragType !== 'field'"
            @dragstart="(e) => { e.stopPropagation(); onDragStart(child, rIdx, null, cIdx); }"
            @dragover.stop.prevent
            @drop.stop="onDropToRowOrSwap(cIdx, rIdx)"
            >
              {{  }}
              <div class="flex flex-wrap gap-1">
                <div v-for = "(field, fIdx) in child.items" :key= "fIdx" :style = "{ width: `calc(${100 / child.items.length}% - 0.5rem)` }" class="p-2 rounded cursor-move"
                :class="field.groupType === 'empty' ? 'bg-white border border-dashed text-gray-400 italic' : 'bg-yellow-200'"
                draggable = "true"
                @dragstart="(e) => { e.stopPropagation(); onDragStart(field, fIdx, rIdx, cIdx); }"
                @dragover.prevent
                @click.prevent="onFieldClick(field)"
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
        <h3 class="font-semibold mb-2">Field Properties</h3>
        <div class="mb-2">
          <label class="block text-sm font-medium text-gray-700">Label:</label>
          <input
            type="text"
            class="mt-1 block w-full border border-gray-300 rounded px-2 py-1"
            v-model="selectedField.label.text"
            disabled
          />
        </div>
        <div class="mb-2">
          <label class="block text-sm font-medium text-gray-700">Editor Type:</label>
          <input
            type="text"
            class="mt-1 block w-full border border-gray-300 rounded px-2 py-1"
            v-model="selectedField.editorType"
            disabled
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
        <button
          @click="applyFieldProperties"
          class="mt-4 bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded"
        >
          Apply
        </button>
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
    { name: 'Field' , groupType: "field"},
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

  const containerCount = ref(0)
  const rowCount = ref(1)
  const fieldCount = ref(1)
  const selectedType = ref('list');
  const selectedPlatform = ref('web');
  const saving = ref(false);

  function updateColCounts() {
    for (const container of droppedContainers.value) {
      if (!(selectedType.value === 'detail' && selectedPlatform.value === 'web')) {
        container.colCount = container.items?.length || 0;
      } else {
        delete container.colCount; // ensure removed
      }

      for (const row of container.items || []) {
        if (!(selectedType.value === 'detail' && selectedPlatform.value === 'web')) {
          row.colCount = row.items?.length || 0;
        } else {
          delete row.colCount; // ensure removed
        }
      }
    }
  }

  function onDragStart(item, itemIndex, parentRowIndex = null, parentContainerIndex = null) {
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
  }

  function onDropRoot(e) {
    // Only allow dropping new containers (created from available items panel)
    console.log(
      'onDropRoot:',
      'dragType:', dragType,
      'draggedItem._fresh:', draggedItem?._fresh,
      'isFromPalette:', isFromPalette
    );
    if (e.target !== e.currentTarget) return;

    if (!(dragType === 'container' && isFromPalette)) {
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
      targetContainer.items.push(createRow());
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
    updateColCounts()
    draggedItem = null;
    dragType = null;
    originalContainerIndex = null;
    originalRowIndex = null;
    isFromPalette = false;
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
      const matchingDynamic = dynamicFields.value.find(f => f.name === draggedItem.name);
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

  function extractFieldsFromJSON(groups) {
    const items = [];

    groups.forEach(group => {
      if (Array.isArray(group.items)) {
        group.items.forEach(item => {
          if (item.itemType === 'empty') return;

          items.push({
            name: item.label?.text || item.dataField || "Unnamed Field",
            groupType: 'field',
            dataField: item.dataField,
            editorType: item.editorType,
            used: false
          });
        });
      }
    });
    return items;
  }

  const selectedField = ref(null);
  const fieldProperties = ref({
    width: '',
    required: false,
    readonly: false,
  });

  function onFieldClick(field)
  {
    selectedField.value = field;

    fieldProperties.value = {
      width: field.width || '',
      required: !!field.required,
      readonly: !!field.readonly,
    };
  }

  function applyFieldProperties()
  {
    if (!selectedField.value) return;

    selectedField.value.width = fieldProperties.value.width;
    selectedField.value.required = fieldProperties.value.required;
    selectedField.value.readonly = fieldProperties.value.readonly;

    alert(`Field "${selectedField.value.label?.text}" updated successfully.`);
  }

  axios.defaults.baseURL = 'http://127.0.0.1:8000';

  async function loadDropzoneStructure() {
    const type = selectedType.value;     // e.g., 'list'
    const platform = selectedPlatform.value;  // e.g., 'web'
    try {
      const res = await axios.get(`http://127.0.0.1:8000/api/layouts/${type}/${platform}`);
      let data = res.data;
      if (!Array.isArray(data)) {
        console.warn("Expected array but got:", data);
        data = [];
      }
      droppedContainers.value = data;
    }
    catch (e) {
      console.error("API load failed:", e);
      droppedContainers.value = [];
    }
  }

  onMounted(loadDropzoneStructure);
  watch([selectedType, selectedPlatform], loadDropzoneStructure);

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
      if(selectedType.value === 'list')
      {
        console.log("Sorry Not Supported Yet!");
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

  function createRow()
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
        console.log("Sorry Not Supported Yet!");
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
        console.log("Sorry Not Supported Yet!");
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
</script>
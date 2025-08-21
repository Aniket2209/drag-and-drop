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
      <h2 class="font-bold mb-2">Draggable Items</h2>
      <div v-for="item in [...items, ...dynamicFields.filter(f => !f.used)]" :key = "item.id" 
        :class="[
          'p-2 mb-2 cursor-move rounded',
          item.type === 'container' ? 'bg-green-300' :
          item.type === 'row' ? 'bg-teal-200' :
          item.type === 'field' ? 'bg-yellow-200' :
          ''
        ]"
        draggable = "true" @dragstart="onDragStart(item)"
        >
        {{ item.name }}
      </div>
    </div>
    <div class="flex-1 p-4 border border-green-400 rounded"
    @dragover.prevent @drop="onDropRoot">
      <h2 class="font-bold mb-2">Drop Zone</h2>
        <div v-for = "container in droppedContainers" :key = "container.dropId" class="p-2 mb-2 bg-green-300 rounded"
        draggable = "true"
        @dragstart = "() => onDragStart(container)"
        @dragover.prevent
        @drop="event => onDropToContainerOrSwap(container.dropId, event)"
        >
          <div v-if="editingContainerId === container.dropId">
            <input
              v-model="container.name"
              @blur="editingContainerId = null"
              class="border border-gray-400 rounded px-2 py-1 text-sm w-full"
              autofocus
            />
          </div>
          <div v-else @dblclick="editingContainerId = container.dropId">
            {{ container.name }}
          </div>
            <div v-for ="child in container.children" :key="child.dropId" class="bg-teal-200 p-2 rounded cursor-move min-h-[40px]"
            :draggable="dragType !== 'field'"
            @dragstart="(e) => { e.stopPropagation(); onDragStart(child, container.dropId); }"
            @dragover.prevent
            @drop="event => onDropToRow(child.dropId)"
            >
              {{  }}
              <div class="flex flex-wrap gap-1">
                <div v-for = "field in child.fields" :key= "field.dropId" :style = "{ width: `calc(${100 / child.fields.length}% - 0.5rem)` }" class="p-2 rounded cursor-move"
                :class="field.type === 'empty' ? 'bg-white border border-dashed text-gray-400 italic' : 'bg-yellow-200'"
                draggable = "true"
                @dragstart="(e) => { e.stopPropagation(); onDragStart(field, child.dropId); }"
                @dragover.prevent
                @click.prevent="onFieldClick(field)"
                >
                <span
                  class="block truncate"
                  :title="field.name"
                  >
                  {{ field.type === 'empty' ? '| |' : field.name }}
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
            v-model="selectedField.label"
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
      @click="saveStructure"
      class="bg-green-500 hover:bg-green-600 text-white px-4 py-1 rounded text-sm"
    >
      💾 Save Layout
    </button>
    <input type="file" accept=".json" 
       ref="fileInput" 
       style="display: none;" 
       @change="loadStructure" />
    <button
      @click="$refs.fileInput.click()"
      class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-1 rounded text-sm"
    >
      📂 Load Layout
    </button>
  </div>
  <div class="flex relative gap-4">
    <pre class="w-1/2 text-xs overflow-auto mt-4 bg-white p-2 border max-h-64">
      {{ JSON.stringify(platformBasedJSON, null, 2) }}
    </pre>
  </div>
</template>

<script setup>
  import { ref, computed, onMounted } from 'vue';
  import customFieldJSON from './assets/customFields.json'

  const items = ref([
    { id: 1, name: 'Row' , type: "row"},
    { id: 2, name: 'Container' , type: "container"},
    { id: 3, name: 'Field' , type: "field"},
    { id: 4, name: '| | Empty Slot' , type: "empty"}
  ]);
  const droppedContainers = ref([]);

  let draggedItem = null;
  let dragType = null;
  let originalParentId = null;
  let originalRowId = null;

  const containerCount = ref(0)
  const rowCount = ref(1)
  const fieldCount = ref(1)
  const selectedPlatform = ref('web');

  function onDragStart(item, parentId = null) {
  // Explicitly assign drag type
    dragType = item.type;

    const isFreshItem = item.hasOwnProperty('id');
    draggedItem = { ...item };

    // Ensure dropId exists once — don’t overwrite
    if (!draggedItem.dropId) {
      draggedItem.dropId = Date.now() + Math.random();
    }

    draggedItem._fresh = isFreshItem;

    // Reset both parent IDs to avoid leakage
    originalParentId = null;
    originalRowId = null;

    // Track source origin
    if (dragType === 'field' || dragType === 'empty') {
      originalRowId = parentId;
    } else if (dragType === 'row') {
      originalParentId = parentId;
    }

    console.log("Drag Start:", draggedItem, "Type:", dragType);
  }

  function onDropRoot() {
    if(dragType === 'container' && draggedItem && draggedItem._fresh && !droppedContainers.value.some(c => c.dropId === draggedItem.dropId)) {
      containerCount.value++;
      draggedItem.name = `Container${containerCount.value}`;
      draggedItem.children = [];
      droppedContainers.value.push(draggedItem);
    }
    else {
      console.warn("Ignored container drop in blank space not fresh");
    }
    draggedItem = null;
    dragType = null;
    originalParentId = null;
    originalRowId= null;
  }

  function onDropToContainerOrSwap(containerId, event)
  {
    if (!draggedItem || !dragType || !draggedItem.type) return;

    if (dragType === 'row' && draggedItem.type === 'row') {
      onDropToContainer(containerId);
    } 
    else if (dragType === 'container' && draggedItem.type === 'container') {
      onDropToSwapContainer(containerId);
    } 
    else {
      console.warn(`Dropped ${dragType} with mismatched type:`, draggedItem);
    }
    event.preventDefault();
  }


  function onDropToContainer(containerId) {
    const targetContainer = droppedContainers.value.find(c => c.dropId === containerId);
    if (!targetContainer || dragType !== "row") return;

    const isFresh = draggedItem?.type === 'row' && draggedItem?.id !== undefined;

    if (isFresh) {
      // Fresh row — assign identity
      draggedItem = {
        name: `Row${rowCount.value}`,
        dropId: Date.now() + Math.random(),
        type: 'row',
        fields: []
      };
      rowCount.value++;
      // Feel free to add naming logic back if needed
    } else if (originalParentId !== null) {
      const originalContainer = droppedContainers.value.find(c => c.dropId === originalParentId);
      if (originalContainer) {
        originalContainer.children = originalContainer.children.filter(
          row => row.dropId !== draggedItem.dropId
        );
      }
    }

    targetContainer.children.push(draggedItem);

    // Cleanup
    draggedItem = null;
    dragType = null;
    originalParentId = null;
    originalRowId = null;
  }

  function onDropToSwapContainer(targetContainerId)
  {
    if (dragType !== 'container' || !draggedItem?.dropId) return;

    const draggedIndex = droppedContainers.value.findIndex(c => c.dropId === draggedItem.dropId);
    const targetIndex = droppedContainers.value.findIndex(c => c.dropId === targetContainerId);

    if (draggedIndex === -1 || targetIndex === -1 || draggedIndex === targetIndex) return;

    // Swap containers
    const temp = droppedContainers.value[draggedIndex];
    droppedContainers.value[draggedIndex] = droppedContainers.value[targetIndex];
    droppedContainers.value[targetIndex] = temp;

    // Cleanup
    draggedItem = null;
    dragType = null;
    originalParentId = null;
    originalRowId = null;
  }

  function onDropToRow(targetRowId) {
    if (dragType !== 'field' && dragType !== 'empty') return;

    const targetContainer = droppedContainers.value.find(container =>
      container.children.some(row => row.dropId === targetRowId)
    );

    if (!targetContainer) return;

    const targetRow = targetContainer.children.find(row => row.dropId === targetRowId);
    if (!targetRow.fields) targetRow.fields = [];

    if(draggedItem.dropId && originalRowId !== null) {
      const originalContainer = droppedContainers.value.find(container =>
        container.children.some(row => row.dropId === originalRowId)
      );
      if (originalContainer) {
        const originalRow = originalContainer.children.find(row => row.dropId === originalRowId);
        if (originalRow) {
          originalRow.fields = originalRow.fields.filter(field => field.dropId !== draggedItem.dropId);
        }
      }
      targetRow.fields.push(draggedItem)
    }
    else {
      if (draggedItem.dropId && originalRowId !== null) {
        // Move existing field from one row to another
        const originalContainer = droppedContainers.value.find(container =>
          container.children.some(row => row.dropId === originalRowId)
        );
        if (originalContainer) {
          const originalRow = originalContainer.children.find(row => row.dropId === originalRowId);
          if (originalRow) {
            originalRow.fields = originalRow.fields.filter(field => field.dropId !== draggedItem.dropId);
          }
        }
        targetRow.fields.push(draggedItem);
      }
      else if (draggedItem?.id) {
        // Dragging from dynamic fields (preserve field)
        const newField = {
          ...draggedItem,
          dropId: Date.now() + Math.random(),  // Give it a unique ID
          used: true
        };
        targetRow.fields.push(newField);

        const matchingDynamic = dynamicFields.value.find(f => f.id === draggedItem.id);
        if (matchingDynamic) matchingDynamic.used = true;
      } 
      else {
        // Regular default fields
        const newField = {
          name: dragType === 'empty' ? "| |" : `Field${fieldCount.value}`,
          dropId: Date.now() + Math.random(),
          type: dragType
        };
        fieldCount.value++;
        
        targetRow.fields.push(newField);
      }
    }

    const matchingDynamic = dynamicFields.value.find(f => f.id === draggedItem.id);
    if (matchingDynamic) matchingDynamic.used = true;
    
    console.log(draggedItem);
    console.log(originalRowId);
    console.log(targetRow.fields);

    draggedItem = null;
    dragType = null;
    originalRowId = null;
    originalParentId = null;
  }

  function onDropToDelete()
  {
    if (!draggedItem || !dragType) return;

    if (dragType === 'row' && originalParentId !== null) {
      const container = droppedContainers.value.find(c => c.dropId === originalParentId);
      if (container) {
        container.children = container.children.filter(row => row.dropId !== draggedItem.dropId);
      }
    }

    if ((dragType === 'field' || dragType === 'empty') && originalRowId !== null) {
      const container = droppedContainers.value.find(c =>
        c.children.some(row => row.dropId === originalRowId)
      );
      if (container) {
        const row = container.children.find(r => r.dropId === originalRowId);
        if (row) {
          row.fields = row.fields.filter(field => field.dropId !== draggedItem.dropId);
        }
      }
    }

    if (dragType === 'container' && draggedItem?.type === 'container') {
      droppedContainers.value = droppedContainers.value.filter(
        c => c.dropId !== draggedItem.dropId
      );
    }

    // Cleanup
    draggedItem = null;
    dragType = null;
    originalParentId = null;
    originalRowId = null;
  }

  const formattedJSON = computed(() =>
    droppedContainers.value.map(container => ({
    name: container.name || "Untitled Group",
    itemType: "group",
    colCount: container.children.length || 1,
    items: container.children.map(row => ({
      itemType: "group",
      colCount: row.fields.length || 1,
      items: row.fields.map(field => {
        if (field.type === "empty") {
          return {
            itemType: "empty",
            label: {
              text: "| |"
            }
          };
        }
        return {
          dataField: field.name || "unknown_field",
          editorType: field.editorType || inferEditorType(field.name),
          label: {
            text: formatLabel(field.name)
          },
          width: field.width || null,
          required: field.required || false,
          readonly: field.readonly || false,
        };
      })
    }))
  }))
  );

  function formatLabel(name) {
    // Convert camelCase or snake_case to Title Case
    return name
      .replace(/[_\-]/g, ' ')
      .replace(/([a-z])([A-Z])/g, '$1 $2')
      .replace(/\b\w/g, l => l.toUpperCase());
  }

  function inferEditorType(fieldName) {
    if (!fieldName) return "dxTextBox";

    const lower = fieldName.toLowerCase();
    if (lower.includes("date")) return "dxDateBox";
    if (lower.includes("time")) return "Datetime";
    if (lower.includes("status") || lower.includes("type")) return "dxSelectBox";
    if (lower.startsWith("rlt_") || lower.includes("employee")) return "Relate";

    return "dxTextBox";
  }

  const editingContainerId = ref(null);
  const dynamicFields = ref([]);

  onMounted(() => {
    const fieldsFromFile = extractFieldsFromJSON(customFieldJSON);
      dynamicFields.value.push(...fieldsFromFile);
  });

  function extractFieldsFromJSON(groups) {
    const fields = [];

    groups.forEach(group => {
      if (Array.isArray(group.items)) {
        group.items.forEach(item => {
          if (item.itemType === 'empty') return;

          fields.push({
            id: Date.now() + Math.random(),
            name: item.label?.text || item.dataField || "Unnamed Field",
            type: 'field',
            dataField: item.dataField,
            editorType: item.editorType,
            used: false
          });
        });
      }
    });
    return fields;
  }

  function saveStructure()
  {
    const json = generateJSON();
    const blob = new Blob([JSON.stringify(json, null, 2)], {type: "application/json"});

    const filename = `${selectedType.value}_${selectedPlatform.value}.json`;
    if (window.navigator.msSaveOrOpenBlob) {
      window.navigator.msSaveBlob(blob, filename);
    } 
    else {
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = filename;
      link.click();
      setTimeout(() => URL.revokeObjectURL(link.href), 1000);
    }
  }

  function loadStructure(event)
  {
    const file = event.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = function(e) {
      try {
        console.log("File contents:", e.target.result);
        const json = JSON.parse(e.target.result);
        console.log("Parsed JSON:", json, selectedType.value, selectedPlatform.value);
        loadFromJson(json);
        alert("Layout loaded successfully!");
      } catch (err) {
        alert("Invalid JSON file.");
      }
    };
    reader.readAsText(file);
  }

  const selectedField = ref(null);
  const fieldProperties = ref({
    width: '',
    required: false,
    readonly: false,
  });

  function onFieldClick(field)
  {
    selectedField.value = {
      ...field,
      label: formatLabel(field.name),
      editorType: inferEditorType(field.name)
    };

    fieldProperties.value = {
      width: field.width || '',
      required: !!field.required,
      readonly: !!field.readonly,
    };
  }

  function applyFieldProperties()
  {
    if (!selectedField.value) return;
    for (const container of droppedContainers.value) {
      for (const row of container.children) {
        const fieldIndex = row.fields.findIndex(f => f.dropId === selectedField.value.dropId);
        if (fieldIndex !== -1) {
          const field = row.fields[fieldIndex];

          field.width = fieldProperties.value.width;
          field.required = fieldProperties.value.required;
          field.readonly = fieldProperties.value.readonly;

          selectedField.value.width = fieldProperties.value.width;
          selectedField.value.required = fieldProperties.value.required;
          selectedField.value.readonly = fieldProperties.value.readonly;

          alert(`Field "${field.name}" updated successfully.`);

          return;
        }
      }
    }

    alert("Failed to update the selected field. It was not found.");
  }

  const platformBasedJSON = computed(() => {
    if (selectedPlatform.value === 'web') {
      // Your current web JSON logic, as in formattedJSON
      return formattedJSON.value;
    } 
    else if (selectedPlatform.value === 'android')
    {
      return droppedContainers.value.map(container => ({
        groupLabel: container.name || "Group",
        rows: (container.children || []).map(row => ({
          fields: (row.fields || []).map(field => ({
            name: field.name,
            type: mapEditorTypeToAndroid(field.editorType || inferEditorType(field.name)),
            label: formatLabel(field.name),
            required: !!field.required,
            readonly: !!field.readonly,
            width: field.width || null
          }))
        }))
      }));
    }
    return [];
  });

  const selectedType = ref('list');

  function mapEditorTypeToAndroid(editorType) {
    switch (editorType) {
      case 'dxTextBox':
        return 'text';
      case 'dxDateBox':
        return 'date';
      case 'dxSelectBox':
        return 'select';
      case 'dxCheckBox':
        return 'checkbox';
      // Add more as needed for your mobile UI
      default:
        return 'text';
    }
  }

  function getAllFields(containers) {
    return containers.flatMap(container =>
      container.children.flatMap(row =>
        row.fields
      )
    );
  }

  function generateListWeb(containers) {
    // Each container becomes a 'group' with nested rows mapped as separate sub-groups
    return containers.map((container, ci) => ({
      name: container.name || `Container${ci + 1}`,  // preserve container name
      itemType: "group",
      colCount: container.children.length,
      items: container.children.map((row, ri) => ({
        itemType: "group",
        colCount: row.fields.length,
        items: row.fields.map(field => {
          if (field.type === "empty") {
            return {
              itemType: "empty",
              label: { text: "| |" }
            };
          }
          return {
            dataField: field.dataField || field.name,
            editorType: field.editorType || "dxTextBox",
            label: { text: field.name },
            visible: true,
            required: field.required || false,
            readonly: field.readonly || false,
            width: field.width || null,
          };
        })
      }))
    }));
  }
  function generateListMobile(containers) {
    return {
      version: "1.0.0",
      Screen: {
        ScreenId: "02_listView",
        ScreenName: "List View",
        layout: {
          type: "ItemLayout",
          dataKey: "userData",
          children: containers.map(container => ({
            type: "verticalContainer",
            name: container.name || "Unnamed Container",
            corner_radius: 15,
            bgColor: "#FFEFEFEF",
            padding: { left: 10, right: 10, top: 5, bottom: 5 },
            children: container.children.map(row => ({
              type: "singleRowLayout",
              name: row.name || "Unnamed Row",
              children: row.fields.map(field => ({
                type:
                  field.editorType === "dxSelectBox"
                    ? "dropdown"
                    : field.editorType === "dxTextBox"
                    ? "inputText"
                    : field.editorType === "dxTextArea"
                    ? "inputText"
                    : "inputText",
                id: field.dataField || field.name,
                keyboardType: field.editorType === "Phone" ? "number" : "text",
                font_size: 15,
                font_weight: "normal",
                padding: { left: 10, right: 10 },
                text: field.name
              }))
            }))
          }))
        }
      }
    };
  }
  function generateEditWeb(containers) {
    return containers.map((container, ci) => ({
      name: container.name || `Container${ci + 1}`,  // Preserve container name
      itemType: "group",
      colCount: container.children.length,
      items: container.children.map((row, ri) => ({
        itemType: "group",
        colCount: row.fields.length,
        items: row.fields.map(field => {
          if (field.type === "empty") {
            return {
              itemType: "empty",
              label: { text: "| |" }
            };
          }
          return {
            dataField: field.dataField || field.name,
            editorType: field.editorType || "dxTextBox",
            label: { text: field.name },
            required: field.required || false,
            readonly: field.readonly || false,
            width: field.width || null,
          };
        })
      }))
    }));
  }
  function generateEditMobile(containers) {
    return {
      version: "1.0.0",
      Screen: {
        ScreenId: "03_createView",
        ScreenName: "Create View",
        layout: {
          type: "singleColumnLayout",
          padding: { top: 10 },
          children: containers.map(container => ({
            name: container.name || "Unnamed Container",
            type: "verticalContainer",
            corner_radius: 15,
            bgColor: "#FFC5DCFF",
            padding: { left: 10, right: 10, top: 5, bottom: 5 },
            children: container.children.map(row => ({
              type: "singleRowLayout",
              children: row.fields.map(field => ({
                type: field.editorType === 'dxSelectBox' ? "dropdown" :
                      field.editorType === 'dxTextBox' ? "inputText" :
                      field.editorType === 'dxTextArea' ? "inputText" : "inputText",
                id: field.dataField || field.name,
                keyboardType: (field.editorType === "Phone" ? "number" : "text"),
                font_size: 15,
                font_weight: "normal",
                padding: { left: 10, right: 10 },
                text: field.name
              }))
            }))
          }))
        }
      }
    };
  }
  function generateDetailWeb(containers) {
    return containers.map((container, ci) => ({
      name: container.name || `Container${ci + 1}`,
      itemType: "group",
      children: container.children.map(row => ({
        name: row.name || "Row",
        itemType: "group",
        items: row.fields.map(field => {
          if (field.type === "empty") {
            return {
              itemType: "empty",
              label: { text: "| |" }
            };
          }
          return {
            dataField: field.dataField || field.name,
            editorType: field.editorType || "dxTextBox",
            label: { text: field.name },
            required: field.required || false,
            readonly: field.readonly || false,
            width: field.width || null
          };
        })
      }))
    }));
  }

  function generateJSON()
  {
    if(selectedType.value==='list' && selectedPlatform.value==='web') {
      return generateListWeb(droppedContainers.value);
    }
    if(selectedType.value==='list' && selectedPlatform.value==='android') {
      return generateListMobile(droppedContainers.value);
    }
    if(selectedType.value==='edit' && selectedPlatform.value==='web') {
      return generateEditWeb(droppedContainers.value);
    }
    if(selectedType.value==='edit' && selectedPlatform.value==='android') {
      return generateEditMobile(droppedContainers.value);
    }
    if(selectedType.value==='detail' && selectedPlatform.value==='web') {
      return generateDetailWeb(droppedContainers.value);
    }
    if(selectedType.value==='detail' && selectedPlatform.value==='android') {
      return {};
    }
  }

  function loadFromJson(json) {
    const key = `${selectedType.value}_${selectedPlatform.value}`;
    let parsed;
    switch (key) {
      case "list_web":
        parsed = parseListWeb(json);
        break;
      case "list_android":
        parsed = parseListMobile(json);
        break;
      case "edit_web":
        parsed = parseEditWeb(json);
        break;
      case "edit_android":
        parsed = parseEditMobile(json);
        break;
      case "detail_web":
        parsed = parseDetailWeb(json);
        break;
      case "detail_android":
        // Implement when ready, or clear:
        parsed = [];
        break;
      default:
        alert("Unsupported layout or file format.");
        parsed = [];
    }
    droppedContainers.value = parsed;
  }

  function generateId() {
    return Date.now() + Math.random();
  }

  function parseListWeb(json) {
    if (!Array.isArray(json)) {
      alert("File format error: Expected array for List Web layout.");
      return [];
    }

    return json.map((group, gi) => ({
      id: generateId(),
      dropId: generateId(),
      name: group.name || `Container${gi + 1}`,
      type: "container",
      children: Array.isArray(group.items) && group.items.length > 0
        ? group.items.map((row, ri) => ({
            id: generateId(),
            dropId: generateId(),
            name: `Row${ri + 1}`,
            type: "row",
            fields: Array.isArray(row.items)
              ? row.items.map(field => {
                  if (field.itemType === "empty") {
                    return {
                      id: generateId(),
                      dropId: generateId(),
                      name: "| |",
                      type: "empty",
                      used: true,
                    };
                  }
                  return {
                    id: generateId(),
                    dropId: generateId(),
                    name: field.dataField || "unknown_field",
                    type: "field",
                    dataField: field.dataField,
                    editorType: field.editorType,
                    used: true,
                    required: field.required || false,
                    readonly: field.readonly || false,
                    width: field.width || null,
                  };
                })
              : []
          }))
        : [{
            id: generateId(),
            dropId: generateId(),
            name: "Row1",
            type: "row",
            fields: []
          }]
    }));
  }

  function parseListMobile(json) {
    try {
      const layoutChildren = json?.Screen?.layout?.children;
      if (!Array.isArray(layoutChildren)) {
        alert("File format error: Expected array of containers in mobile list layout.");
        return [];
      }

      return layoutChildren.map(container => {
        const rows = Array.isArray(container.children) ? container.children : [];

        return {
          id: generateId(),
          dropId: generateId(),
          name: container.name || "Unnamed Container",
          type: "container",
          children: rows.map((row, idx) => {
            if (row.type !== "singleRowLayout" || !Array.isArray(row.children)) {
              return null;
            }
            return {
              id: generateId(),
              dropId: generateId(),
              name: row.name || `Row${idx + 1}`,
              type: "row",
              fields: row.children.map(field => ({
                id: generateId(),
                dropId: generateId(),
                name: field.id || field.text || "unknown_field",
                type: "field",
                dataField: field.id || field.text || "unknown_field",
                editorType: mapMobileTypeToEditorType(field.type),
                used: true,
                required: false,
                readonly: false,
                width: null
              }))
            };
          }).filter(r => r !== null)
        };
      });
    } catch (err) {
      console.error("Error parsing list mobile layout:", err);
      alert("Invalid JSON structure for Mobile List layout.");
      return [];
    }
  }

  function parseEditWeb(json) {
    if (!Array.isArray(json)) {
      alert("File format error: Expected array for Edit Web layout.");
      return [];
    }

    return json.map((group, gi) => ({
      id: generateId(),
      dropId: generateId(),
      name: group.name || `Container${gi + 1}`,
      type: "container",
      children: Array.isArray(group.items) && group.items.length > 0
        ? group.items.map((row, ri) => ({
            id: generateId(),
            dropId: generateId(),
            name: `Row${ri + 1}`,
            type: "row",
            fields: Array.isArray(row.items)
              ? row.items.map(field => {
                  if (field.itemType === "empty") {
                    return {
                      id: generateId(),
                      dropId: generateId(),
                      name: "| |",
                      type: "empty",
                      used: true
                    };
                  }
                  return {
                    id: generateId(),
                    dropId: generateId(),
                    name: field.dataField || "unknown_field",
                    type: "field",
                    dataField: field.dataField,
                    editorType: field.editorType,
                    used: true,
                    required: field.required || false,
                    readonly: field.readonly || false,
                    width: field.width || null
                  };
                })
              : []
          }))
        : [{
            id: generateId(),
            dropId: generateId(),
            name: "Row1",
            type: "row",
            fields: []
          }]
    }));
  }

  function parseEditMobile(json) {
    try {
      // Defensive: Check structure exists
      if (!json || !json.Screen || !json.Screen.layout || !Array.isArray(json.Screen.layout.children)) {
        alert("Invalid mobile edit structure.");
        return [];
      }

      // Each 'verticalContainer' in children array is mapped to a container
      const containers = json.Screen.layout.children.map(vc => {
        // Defensive: children could be undefined
        const childrenArr = Array.isArray(vc.children) ? vc.children : [];

        return {
          id: generateId(),
          dropId: generateId(),
          name: (typeof vc.name === "string") ? vc.name : "",
          type: "container",
          children: childrenArr.map((row, idx) => {
            // Each singleRowLayout in verticalContainer.children is a row
            if (row.type !== "singleRowLayout" || !Array.isArray(row.children)) {
              return null; // skip unexpected node
            }
            return {
              id: generateId(),
              dropId: generateId(),
              name: "Row" + (idx + 1),
              type: "row",
              fields: row.children
                .filter(field => field && (field.type === "inputText" || field.type === "dropdown"))
                .map(field => ({
                  id: generateId(),
                  dropId: generateId(),
                  name: field.id || field.text || "unknown_field",
                  type: "field",
                  dataField: field.id || field.text,
                  editorType: mapMobileTypeToEditorType(field.type),
                  used: true,
                  required: field.isRequired || false,
                  readonly: false,
                  width: null,
                })),
            };
          }).filter(r => !!r) // Remove nulls
        };
      });

      return containers;
    } catch (err) {
      console.error("Error in parseEditMobile:", err);
      alert("Invalid JSON structure for Android Edit layout.");
      return [];
    }
  }

  // Helper to map mobile field types to your editorType strings
  function mapMobileTypeToEditorType(type) {
    switch (type) {
      case "inputText":
        return "dxTextBox";
      case "dropdown":
        return "dxSelectBox";
      case "inputTextArea":
        return "dxTextArea";
      default:
        return "dxTextBox";
    }
  }

  function parseDetailWeb(json) {
    if (!Array.isArray(json)) {
      alert("File format error: Expected array for Detail Web layout.");
      return [];
    }

    return json.map((container, ci) => {
      const rows = Array.isArray(container.children) ? container.children : [];

      return {
        id: generateId(),
        dropId: generateId(),
        name: container.name || `Container${ci + 1}`,
        type: "container",
        children: rows.length > 0
          ? rows.map((row, ri) => ({
              id: generateId(),
              dropId: generateId(),
              name: row.name || `Row${ri + 1}`,
              type: "row",
              fields: Array.isArray(row.items)
                ? row.items.map(field => {
                    if (field.itemType === "empty") {
                      return {
                        id: generateId(),
                        dropId: generateId(),
                        name: "| |",
                        type: "empty",
                        used: true
                      };
                    }
                    return {
                      id: generateId(),
                      dropId: generateId(),
                      name: (field.label && field.label.text) || field.dataField || "unknown_field",
                      type: "field",
                      dataField: field.dataField,
                      editorType: field.editorType || "dxTextBox",
                      used: true,
                      required: !!field.required,
                      readonly: !!field.readonly,
                      width: field.width || null
                    };
                  })
                : []
            }))
          : [{
              id: generateId(),
              dropId: generateId(),
              name: "Row1",
              type: "row",
              fields: []
            }]
      };
    });
  }

function splitIntoRows(items, size) {
  let result = [];
  for(let i=0; i < items.length; i += size) {
    result.push(items.slice(i, i+size));
  }
  return result;
}
</script>
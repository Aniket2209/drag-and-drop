<template>
  <div class="flex relative gap-8">
    <div class="w-1/2 p-4 border border-blue-300 p-4 rounded">
      <h2 class="font-bold mb-2">Draggable Items</h2>
      <div v-for="item in [...items, ...dynamicFields.filter(f => !f.used)]" :key = "item.id" class="p-2 mb-2 bg-blue-300 cursor-move rounded"
        draggable = "true" @dragstart="onDragStart(item)"
        >
        {{ item.name }}
      </div>
    </div>
    <div class="w-1/2 p-4 border border-green-400 p-4 rounded"
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
  </div>
  <!-- Delete Zone -->
  <div
    class="mt-8 w-1/4 h-24 border border-red-400 bg-gray-200 flex items-center justify-center text-red-700 font-bold text-lg rounded"
    @dragover.prevent
    @drop="onDropToDelete()"
  >
    🗑️ Drag Items here to delete Them
  </div>
  <pre class="text-xs overflow-auto mt-4 bg-white p-2 border max-h-64">
    {{ JSON.stringify(formattedJSON, null, 2) }}
  </pre>
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
    if (dragType === 'field') {
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

    if (dragType === 'field' && originalRowId !== null) {
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
  {
    return droppedContainers.value.map(container =>
      ({
        name: container.name || "Untitled Group",
        itemType: "group",
        colCount: 2,
        items: container.children.flatMap(row =>
          row.fields.map(field => {
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
              editorType: inferEditorType(field.name),
              label: {
                text: formatLabel(field.name)
              }
            };
          })
        )
      }));
    });

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
</script>
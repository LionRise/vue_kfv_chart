<template>
    <div class="flex flex-col">
        <label :for="id" class="mb-1 text-sm font-medium text-gray-700">{{ label }}:</label>
        <select :id="id" :value="modelValue"
            @change="$emit('update:modelValue', ($event.target as HTMLSelectElement).value)"
            class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-blue-400 text-sm">
            <option value="">{{ allLabel }}</option>
            <option v-for="option in options" :key="option" :value="option">
                {{ displayMap?.[option] || option }}
            </option>
        </select>
    </div>
</template>

<script setup lang="ts">
withDefaults(defineProps<{
    id: string;
    label: string;
    options: string[];
    modelValue: string;
    displayMap?: Record<string, string>;
    allLabel?: string;
}>(), {
    allLabel: "Alle"
});

defineEmits(["update:modelValue"]);
</script>
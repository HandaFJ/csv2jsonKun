<script setup lang="ts">
defineProps<{
  modelValue: string;
}>();

defineEmits<{
  (e: "update:modelValue", value: string): void;
}>();
</script>

<template>
  <section class="pane">
    <div class="pane-header">
      <span>CSV Input</span>
      <button @click="$emit('update:modelValue', '')" class="btn-secondary">
        クリア
      </button>
    </div>
    <textarea
      :value="modelValue"
      @input="
        $emit('update:modelValue', ($event.target as HTMLTextAreaElement).value)
      "
      class="csv-textarea"
      placeholder="ここにCSVを入力またはペーストしてください..."
      spellcheck="false"
    ></textarea>
  </section>
</template>

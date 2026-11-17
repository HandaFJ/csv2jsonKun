<script setup lang="ts">
import { ref } from "vue";

defineProps<{
  jsonString: string;
  error: string | null;
  highlightedJson: string;
}>();

const copySuccess = ref(false);

async function copyToClipboard(text: string) {
  if (!text) return;
  await navigator.clipboard.writeText(text);
  copySuccess.value = true;
  setTimeout(() => {
    copySuccess.value = false;
  }, 2000);
}
</script>

<template>
  <section class="pane">
    <div class="pane-header">
      <span>JSON Output (Realtime)</span>
      <button @click="copyToClipboard(jsonString)" class="btn-primary">
        {{ copySuccess ? "コピー完了!" : "JSONをコピー" }}
      </button>
    </div>
    <div v-if="error" class="error">
      {{ error }}
    </div>
    <pre
      v-else
      class="json-output"
    ><code class="json-code" v-html="highlightedJson"></code></pre>
  </section>
</template>

<script setup lang="ts">
import { computed, ref } from "vue";
import { useLockHtmlScroll } from "./use-lock-html-scroll";

// click the button then we found html will not be scroll locked.
const open = ref(false);

useLockHtmlScroll(computed(() => open.value));
</script>

<template>
  <div style="height: 5000px">
    <button @click="open = !open">trigger</button>
  </div>
  <div class="modal" v-show="open"></div>
</template>

<style>
.modal {
  position: fixed;
  inset: 0;
  background-color: red;
  width: 500px;
  height: 500px;
}
</style>

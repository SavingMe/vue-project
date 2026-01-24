<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
function add() {
    count.value++
}
function sub() {
    count.value--

}
</script>
<template>
    <input type="button" value="+" @click="add" />
    {{ count }}
      <input type="button" value="+" @click="add" />
    <input type="button" value="-" @click="sub" />
</template>

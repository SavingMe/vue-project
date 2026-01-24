<script setup lang="ts">
import { ref, reactive } from 'vue'
const msg = ref('Hello World')
const user = reactive({ name: '张三' })
</script>
<template>
    <h2 class="ref">{{ msg }}</h2>
    <h2 class="reactive">{{ user.name }}</h2>
</template>

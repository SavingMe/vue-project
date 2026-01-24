<script setup lang="ts">
import { ref, reactive } from 'vue'
const img = ref('./src/assets/vue.svg')
</script>
<template>
    <img :src="img" alt="vue logo">
</template>

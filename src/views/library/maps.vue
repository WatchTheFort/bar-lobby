<route lang="json">
{ "meta": { "title": "Maps", "order": 1, "transition": { "name": "slide-left" }, "offine": true } }
</route>

<template>
    <div>
        <h1>{{ route.meta.title }}</h1>
        <div>TEst</div>
    </div>
</template>

<script lang="ts" setup>
/**
 * TODO:
 * - Similar to online map browser
 * - Indicator whether map is installed or not
 * - Easy one click install button
 * - Demo map button that launches a simple offline game on the map
 * - Searchable
 * - Sortable
 * - Filterable
 * - Paginated
 */

import { useRoute } from "vue-router";

const route = useRoute();
</script>

<style lang="scss" scoped></style>

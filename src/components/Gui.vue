<template>
    <div class="GUI_container">
        <h3>GEOMETRY SELECT</h3>
        <div class="geometry_container">
            <select name="select" v-model="geometry">
                <option value="chair">Chair</option>
                <option value="cube" selected>Cube</option>
                <option value="helmet">Helmet</option>
                <option value="suzanne">Suzanne</option>
            </select>
            <button @click="() => { $emit('geometryChange', geometry) }">Create</button>
        </div>

        <h3>POSITION</h3>
        <div>
            <input v-model="position.x" @change="$emit('positionChange', position)" type="number">
            <input v-model="position.y" @change="$emit('positionChange', position)" type="number">
            <input v-model="position.z" @change="$emit('positionChange', position)" type="number">
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, watch  } from 'vue';
const props = defineProps({
    position: Object,
})

watch(
  () => props.position,
  () => {
    position.value.x = props.position?.x
    position.value.y = props.position?.y
    position.value.z = props.position?.z
  }
)

const geometry = ref(null)
const position = ref({ x: 0, y: 0, z: 0 })

</script>

<style scoped>
.GUI_container {
    position: absolute;
    z-index: 100;
    left: 0;
    top: 0;
}

.geometry_container {
    display: flex;
    gap: 8px
}
</style>

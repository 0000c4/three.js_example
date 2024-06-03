<template>
    <div class="GUI_container">
        <h3>GEOMETRY SELECT</h3>
        <div class="geometry_container">
            <select name="geometry" v-model="geometry">
                <option value="chair">Chair</option>
                <option value="cube" selected>Cube</option>
                <option value="helmet">Helmet</option>
                <option value="suzanne">Suzanne</option>
            </select>
            <button @click="() => { $emit('geometryChange', geometry) }">Create</button>
        </div>

        <h3>POSITION</h3>
        <div>
            <input v-model="position.x" @change="$emit('positionChange', position)" type="number" step="0.1">
            <input v-model="position.y" @change="$emit('positionChange', position)" type="number" step="0.1">
            <input v-model="position.z" @change="$emit('positionChange', position)" type="number" step="0.1">
        </div>
        <h3>MATERIAL</h3>
        <div class="material_container">
            <div class="material_row">
                <label for="albedo">albedo</label>
                <select name="albedo" v-model="material.map" @click="() => { $emit('materialChange', material) }">
                    <option value="textures/albedo/albedo-leather.ktx2">albedo-leather</option>
                    <option value="textures/albedo/albedo-metal.png" selected>albedo-metal</option>
                    <option value="textures/albedo/albedo-velours.png">albedo-velours</option>
                    <option value="textures/albedo/albedo-wood.png">albedo-wood</option>
                </select>
            </div>
            <div class="material_row">
                <label for="normal">normal</label>
                <select name="normal" v-model="material.normalMap" @click="() => { $emit('materialChange', material) }">
                    <option value="textures/normal/normal-leather.ktx2">normal-leather</option>
                    <option value="textures/normal/normal-metal.png" selected>normal-metal</option>
                    <option value="textures/normal/normal-velours.png">normal-velours</option>
                    <option value="textures/normal/normal-wood.png">normal-wood</option>
                </select>
            </div>
            <div class="material_row">
                <label for="metalness">metalness</label>
                <select name="metalness" v-model="material.metalnessMap"
                    @click="() => { $emit('materialChange', material) }">
                    <option value="textures/metalness/metalness-leather.ktx2">metalness-leather</option>
                    <option value="textures/metalness/metalness-metal.png" selected>metalness-metal</option>
                    <option value="textures/metalness/metalness-velours.png">metalness-velours</option>
                    <option value="textures/metalness/metalness-wood.png">metalness-wood</option>
                </select>
            </div>
            <div class="material_row">
                <label for="roughness">roughness</label>
                <select name="roughness" v-model="material.roughnessMap"
                    @click="() => { $emit('materialChange', material) }">
                    <option value="textures/roughness/roughness-leather.ktx2">roughness-leather</option>
                    <option value="textures/roughness/roughness-metal.png" selected>roughness-metal</option>
                    <option value="textures/roughness/roughness-velours.png">roughness-velours</option>
                    <option value="textures/roughness/roughness-wood.png">roughness-wood</option>
                </select>
            </div>
            <div class=buttton_container>
                <button @click="() => $emit('delete')">Delete object</button>
                <button @click="() => $emit('save')">Save Scene</button>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';

const props = defineProps({
    position: Object,
    resetData: Boolean
})

watch(
    () => props.position,
    () => {
        position.value.x = props.position?.x
        position.value.y = props.position?.y
        position.value.z = props.position?.z
    },
)
watch(
    () => props.resetData, //reset form data
    () => {
        material.value = {
            map: null,
            normalMap: null,
            metalnessMap: null,
            roughnessMap: null
        }
    },
)

const geometry = ref(null)
const position = ref({ x: 0, y: 0, z: 0 })
const material = ref({
    map: null,
    normalMap: null,
    metalnessMap: null,
    roughnessMap: null
})

</script>

<style scoped>
h3 {
    margin-top: 8px;
    margin-bottom: 2px;
    font-size: 14px;
}

.GUI_container {
    padding: 8px;
    border-radius: 8px;
    position: absolute;
    z-index: 100;
    right: 2px;
    top: 4px;
    color: white;
    background-color: rgb(88, 88, 88);
}

.geometry_container {
    display: flex;
    gap: 8px
}

.material_container {
    display: flex;
    flex-direction: column
}

.material_row {
    display: flex;
    gap: 8px;
}

.buttton_container {
    margin-top: 16px;
    display: flex;
    gap: 8px;
}
</style>

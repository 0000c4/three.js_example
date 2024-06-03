<template>
	<Gui :position="position" @geometryChange="loadGeometry" @positionChange="setIntersectedPosition"
		@materialChange="setMaterial" @delete="deleteMesh" @save="saveScene" :resetData="resetGuiData"/>
</template>
<script setup lang="ts">
import * as THREE from 'three';
import Stats from 'three/addons/libs/stats.module.js';
import { ref } from 'vue';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { TransformControls } from 'three/addons/controls/TransformControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { get, set } from 'idb-keyval';
import Gui from './Gui.vue';

const camera = new THREE.PerspectiveCamera(35, window.innerWidth / window.innerHeight, 1, 500);
const renderer = new THREE.WebGLRenderer({ antialias: true });
const control = new TransformControls(camera, renderer.domElement);
const raycaster = new THREE.Raycaster();
const stats = new Stats();

const loader = new GLTFLoader();
const ktx2Loader = new KTX2Loader().setTranscoderPath( 'basis/' ).detectSupport( renderer );

const position = ref({ x: 0, y: 0, z: 0 })
const resetGuiData = ref(false);

let scene = new THREE.Scene();

let INTERSECTED: any;
const pointer = new THREE.Vector2();

interface ITextureMap {
	map: THREE.Texture | null,
	normalMap: THREE.Texture | null,
	metalnessMap: THREE.Texture | null,
	roughnessMap: THREE.Texture | null
}

let TextureMap: ITextureMap = {
	map: null,
	normalMap: null,
	metalnessMap: null,
	roughnessMap: null
}


init();

async function init() {
	try {
		scene = await loadScene()
	} catch (error) {
		console.log('saved scene not found')
	}

	document.body.appendChild(stats.dom);

	scene.background = new THREE.Color(0x999999);

	const light = new THREE.DirectionalLight(0xffffff, 3);
	light.position.set(0.5, 1.0, 0.5).normalize();

	control.addEventListener('change', render);
	control.addEventListener('objectChange', () => {
		position.value = { x: INTERSECTED.position.x, y: INTERSECTED.position.y, z: INTERSECTED.position.z }
	})

	control.addEventListener('dragging-changed', function (event) {

		orbit.enabled = !event.value;

	});


	const grid = new THREE.GridHelper(50, 50, 0xffffff, 0x7b7b7b);
	grid.position.set(0, -1, 0)

	camera.position.y = 5;
	camera.position.z = 10;

	scene.add(camera);
	scene.add(light);
	scene.add(control);
	scene.add(grid);

	renderer.setPixelRatio(window.devicePixelRatio);
	renderer.setSize(window.innerWidth, window.innerHeight);
	document.body.appendChild(renderer.domElement);

	const orbit = new OrbitControls(camera, renderer.domElement);
	orbit.addEventListener('change', render);
	orbit.update();

	document.addEventListener('mousedown', onPointerMove);
	window.addEventListener('resize', onWindowResize);

	render()

}

function setIntersectedPosition(event: { x: number, y: number, z: number }) {
	INTERSECTED.position.set(event.x, event.y, event.z)
	render()
}
function onWindowResize() {

	camera.aspect = window.innerWidth / window.innerHeight;
	camera.updateProjectionMatrix();

	renderer.setSize(window.innerWidth, window.innerHeight);

	render();

}
function onPointerMove(event: { clientX: number; clientY: number; }) {
	pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
	pointer.y = - (event.clientY / window.innerHeight) * 2 + 1;

}

async function TextureToMaterial(URL: string, map: keyof ITextureMap) {
	const texture = URL.split('.')[1] === 'ktx2' ? await ktx2Loader.loadAsync(URL) : new THREE.TextureLoader().load(URL, render)
	texture.wrapS = THREE.RepeatWrapping
	texture.wrapT = THREE.RepeatWrapping
	texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
	if (map !== 'normalMap') texture.colorSpace = THREE.SRGBColorSpace;

	TextureMap[map] = texture
}
interface IMaterial {
	map: null | string,
	normalMap: null | string,
	metalnessMap: null | string,
	roughnessMap: null | string
}
async function setMaterial(material: IMaterial) {
	material.map !== null ? await TextureToMaterial(material.map, 'map') : null
	material.normalMap !== null ? await TextureToMaterial(material.normalMap, 'normalMap') : null
	material.metalnessMap !== null ? await TextureToMaterial(material.metalnessMap, 'metalnessMap') : null
	material.roughnessMap !== null ? await TextureToMaterial(material.roughnessMap, 'roughnessMap') : null
	for (const children of INTERSECTED.children) {
		children.material = new THREE.MeshStandardMaterial({
			map: TextureMap.map,
			normalMap: TextureMap.normalMap,
			metalnessMap: TextureMap.metalnessMap,
			roughnessMap: TextureMap.roughnessMap
		})
		for (const subChildren of children.children) {
			subChildren.material = new THREE.MeshStandardMaterial({
				map: TextureMap.map,
				normalMap: TextureMap.normalMap,
				metalnessMap: TextureMap.metalnessMap,
				roughnessMap: TextureMap.roughnessMap
			})
		}
	}
	INTERSECTED.material = new THREE.MeshStandardMaterial({
		map: TextureMap.map,
		normalMap: TextureMap.normalMap,
		metalnessMap: TextureMap.metalnessMap,
		roughnessMap: TextureMap.roughnessMap
	})
}

function loadGeometry(geometry: string): void {
	
	loader.load(`geometries/${geometry}.glb`, function (gltf) {
		scene.add(gltf.scene.children[0]);
		render();
	}, undefined, function (error) {

		console.error(error);

	});
}

function deleteMesh() {
	control.detach()
	scene.remove(INTERSECTED)
	position.value = { x: 0, y: 0, z: 0 }
	resetGuiData.value = !resetGuiData.value
	render()
}
function saveScene() {
	control.detach();
	scene.remove(control);
	const jsonScene = scene.toJSON();
	set('scene', JSON.stringify(jsonScene));
}
function loadScene() {
	return new Promise<any>(function (resolve, reject) {
		get('scene').then((val) => {
			try {
				resolve(new THREE.ObjectLoader().parse(JSON.parse(val)))
			} catch (error) {
				reject(error)
			}

		});
	});
}
function rayCast(): void {
	raycaster.setFromCamera(pointer, camera);
	const intersects = raycaster.intersectObjects(scene.children, false);
	if (intersects.length > 0) {

		if (INTERSECTED != intersects[0].object) {

			if (intersects[0].object.type !== "GridHelper") {
				resetGuiData.value = !resetGuiData.value
				INTERSECTED = intersects[0].object;
				control.attach(INTERSECTED);
				position.value = { x: INTERSECTED.position.x, y: INTERSECTED.position.y, z: INTERSECTED.position.z }
				
			}

		}

	}
}
function render(): void {
	rayCast();
	stats.update();
	renderer.render(scene, camera);
}
</script>

<style scoped></style>

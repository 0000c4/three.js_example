<template>
	<Gui :position="position" @geometryChange="loadGeometry" @positionChange="setIntersectedPosition" />
</template>
<script setup lang="ts">
import * as THREE from 'three';
import Stats from 'three/addons/libs/stats.module.js';
import { ref} from 'vue';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { TransformControls } from 'three/addons/controls/TransformControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
import Gui from './Gui.vue';

const camera = new THREE.PerspectiveCamera(35, window.innerWidth / window.innerHeight, 1, 500);
const renderer = new THREE.WebGLRenderer({ antialias: true });
const control = new TransformControls(camera, renderer.domElement);
const raycaster = new THREE.Raycaster();
const stats = new Stats();

const loader = new GLTFLoader();
const gui = new GUI();

const position = ref({x: 0, y: 0, z: 0})

let scene = localStorage.getItem('scene') !== null ? loadScene() : new THREE.Scene();



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


interface IGuiControls {
	x: number,
	y: number,
	z: number,
	geometryText: string,
	albedoText: string,
	metalnessText: string,
	roughnessText: string,
	normalText: string,
	geometry: string,
	metalness: 0,
	roughness: 0,
	deleteMesh: () => void
	newGeometry: () => void
	saveScene: () => void
}

let guiControls: IGuiControls = {
	x: 0,
	y: 0,
	z: 0,
	geometryText: 'select geometry',
	albedoText: 'select albedo',
	metalnessText: 'select metalness',
	roughnessText: 'select roughness',
	normalText: 'select normal',
	geometry: '',
	metalness: 0,
	roughness: 0,
	deleteMesh: () => {
		control.detach()
		scene.remove(INTERSECTED)
		resetGuiControl()
		render()
	},
	newGeometry: () => {
		loadGeometry(guiControls.geometry)
	},
	saveScene: () => {
		control.detach();
		scene.remove(control);
		const jsonScene = scene.toJSON();
		localStorage.clear()
		localStorage.setItem('scene', JSON.stringify(jsonScene));
	},

}

function resetGuiControl(): void {
	guiControls.geometryText = 'select geometry'
	guiControls.albedoText = 'select albedo'
	guiControls.metalnessText = 'select metalness'
	guiControls.roughnessText = 'select roughness'
	guiControls.normalText = 'select normal'
	guiControls.metalness = 0
	guiControls.roughness = 0
	guiControls.x = 0
	guiControls.y = 0
	guiControls.z = 0
}

init();
initGui();

function init(): void {

	document.body.appendChild(stats.dom);

	scene.background = new THREE.Color(0x999999);

	const light = new THREE.DirectionalLight(0xffffff, 3);
	light.position.set(0.5, 1.0, 0.5).normalize();

	control.addEventListener('change', render);
	control.addEventListener('objectChange', () => {
		guiControls.x = INTERSECTED.position.x
		guiControls.y = INTERSECTED.position.y
		guiControls.z = INTERSECTED.position.z
		position.value = {x: INTERSECTED.position.x, y: INTERSECTED.position.y, z: INTERSECTED.position.z}
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
function initGui(): void {

	const settings = gui.addFolder('Object Settings');

	settings.add(guiControls, 'albedoText',
		[
			'albedo-metal',
			'albedo-velours',
			'albedo-wood',
		]
	).onChange((value) => {
		const textureURL = `textures/albedo/${value}.png`
		TextureToMaterial(textureURL, 'map')
		render()
	}).listen()

	settings.add(guiControls, 'metalnessText',
		[
			'metalness-metal',
			'metalness-velours',
			'metalness-wood',
		]
	).onChange((value) => {
		const textureURL = `textures/metalness/${value}.png`
		TextureToMaterial(textureURL, 'metalnessMap')

		render()
	}).listen()

	function TextureToMaterial(URL: string, map: keyof ITextureMap) {
		const texture = new THREE.TextureLoader().load(URL, render);
		texture.wrapS = THREE.RepeatWrapping
		texture.wrapT = THREE.RepeatWrapping
		texture.colorSpace = THREE.SRGBColorSpace;
		texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
		TextureMap[map] = texture

		INTERSECTED.material = new THREE.MeshStandardMaterial({
			map: TextureMap.map,
			normalMap: TextureMap.normalMap,
			metalnessMap: TextureMap.metalnessMap,
			roughnessMap: TextureMap.roughnessMap
		})
	}

	settings.add(guiControls, 'roughnessText',
		[
			'roughness-metal',
			'roughness-velours',
			'roughness-wood',
		]
	).onChange((value) => {
		const texture = new THREE.TextureLoader().load(`textures/roughness/${value}.png`, render);
		texture.wrapS = THREE.RepeatWrapping
		texture.wrapT = THREE.RepeatWrapping
		texture.colorSpace = THREE.SRGBColorSpace;
		texture.anisotropy = renderer.capabilities.getMaxAnisotropy();

		TextureMap.roughnessMap = texture

		INTERSECTED.material = new THREE.MeshStandardMaterial({
			map: TextureMap.map,
			normalMap: TextureMap.normalMap,
			metalnessMap: TextureMap.metalnessMap,
			roughnessMap: TextureMap.roughnessMap
		})
		render()
	}).listen()

	settings.add(guiControls, 'normalText',
		[
			'normal-metal',
			'normal-velours',
			'normal-wood',
		]
	).onChange((value) => {

		const texture = new THREE.TextureLoader().load(`textures/normal/${value}.png`, render);
		texture.wrapS = THREE.RepeatWrapping
		texture.wrapT = THREE.RepeatWrapping
		//texture.colorSpace = THREE.SRGBColorSpace;
		texture.anisotropy = renderer.capabilities.getMaxAnisotropy();

		TextureMap.normalMap = texture

		INTERSECTED.material = new THREE.MeshStandardMaterial({
			map: TextureMap.map,
			normalMap: TextureMap.normalMap,
			metalnessMap: TextureMap.metalnessMap,
			roughnessMap: TextureMap.roughnessMap
		})
		render()
	}).listen()

	settings.add(guiControls, 'metalness', 0, 1, 0.001).onChange((value) => {

		INTERSECTED.material.metalness = value

		render()
	}).listen()

	settings.add(guiControls, 'roughness', 0, 1, 0.001).onChange((value) => {

		INTERSECTED.material.roughness = value

		render()
	}).listen()

	settings.add(guiControls, 'deleteMesh')
}
function setIntersectedPosition(event: { x: number, y: number, z: number }) {
	INTERSECTED.position.set(event.x/100, event.y /100, event.z / 100)
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

function loadGeometry(geometry: string): void {
	console.log(geometry)
	loader.load(`geometries/${geometry}.glb`, function (gltf) {
		scene.add(gltf.scene.children[0]);
		render();
	}, undefined, function (error) {

		console.error(error);

	});
}
function loadScene() {
	const jsonScene = localStorage.getItem('scene');
	if (jsonScene === null) {
		return 'saved scene not found'
	}
	return new THREE.ObjectLoader().parse(JSON.parse(jsonScene));
}
function rayCast(): void {
	raycaster.setFromCamera(pointer, camera);
	const intersects = raycaster.intersectObjects(scene.children, false);
	if (intersects.length > 0) {

		if (INTERSECTED != intersects[0].object) {

			if (intersects[0].object.type !== "GridHelper") {
				INTERSECTED = intersects[0].object;
				resetGuiControl()
				position.value = {x: INTERSECTED.position.x, y: INTERSECTED.position.y, z: INTERSECTED.position.z}
				guiControls.x = INTERSECTED.position.x
				guiControls.y = INTERSECTED.position.y
				guiControls.z = INTERSECTED.position.z
				control.attach(INTERSECTED);
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

<style scoped>
.read-the-docs {
	color: #888;
}
</style>

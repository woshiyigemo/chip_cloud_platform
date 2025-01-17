<template>
	<div class="container">
		<div ref="top" class="top"></div>
		<div ref="left" class="left"></div>
		<div ref="main" class="main"></div>
	</div>
</template>

<script setup lang="ts">
import { onMounted, ref, toRefs, watch } from "vue";
import * as PIXI from "pixi.js";
import { throttle } from "lodash-es";
// import useCanvasView from "../../../../hooks/CanvasView";
import { getClosestVal, getStepByZoom } from "../../../../utils/CalculateRuler";
import { Positon, offset } from "../../../../types";
import bus from "../../../../utils/EventBus";

// const { getShapeGraphicsArr } = useCanvasView();
const top = ref<HTMLElement>();
const main = ref<HTMLElement>();
const left = ref<HTMLElement>();
// props
const props = defineProps<{
	checkedOptions: string[];
	zoom: number;
	flag: boolean;
}>();
// zoom 是缩放比， zoom 越大 step越小
const { checkedOptions, zoom, flag } = toRefs(props);
// 记录画布总偏移量
let totalOffset = { x: 0, y: 0 };
// 监听画布右侧的选项变化
watch(checkedOptions, (newValue) => {
	const res = matchLayerContainers(newValue);
	reRender(res);
});
// 监听(点击放大缩小按钮)、(鼠标滚轮)时zoom的变化来重新刻画标尺
watch(zoom, (newZoom) => {
	updateRuler(newZoom, totalOffset);
});
watch(flag, () => {
	scaleCenter(zoom.value);
});
// 初始化main、left、top & App
const mainApp = new PIXI.Application({
	background: "#000000",
	autoDensity: true,
	resolution: window.devicePixelRatio,
	antialias: true,
});
const ParentContainer = new PIXI.Container();

const setupMainApp = () => {
	const width = main.value!.clientWidth;
	const height = main.value!.clientHeight;
	mainAppRectBound.width = width;
	mainAppRectBound.height = height;
	// zIndex 生效
	ParentContainer.sortableChildren = true;
	// 设置舞台能够交互
	mainApp.stage.eventMode = "dynamic";
	// 确保舞台覆盖整个场景
	mainApp.stage.hitArea = mainApp.screen;
	// 各种监听
	mainApp.stage.on("wheel", onMouseWheel);
	mainApp.stage.on("pointerdown", onDragStart);
	mainApp.stage.on("pointermove", throttleOnDrag);
	mainApp.stage.on("pointerup", onDragEnd);
	mainApp.stage.on("pointerupoutside", onDragEnd);
	mainApp.renderer.resize(width, height);
	// 初次渲染所有的图形
	// renderShapes(DataJson.data);
	mainApp.stage.addChild(ParentContainer);
	main.value?.appendChild(mainApp.view as any);
};
const topApp = new PIXI.Application({
	background: "#ffffff",
	autoDensity: true,
	resolution: window.devicePixelRatio,
	antialias: true,
});
const setupTopApp = () => {
	const width = top.value!.clientWidth;
	const height = top.value!.clientHeight;
	topApp.renderer.resize(width, height);
	topApp.stage.addChild(rulerContainerX);
	top.value?.appendChild(topApp.view as any);
};
const leftApp = new PIXI.Application({
	background: "#ffffff",
	autoDensity: true,
	resolution: window.devicePixelRatio,
	antialias: true,
});
const setupLeftApp = () => {
	const width = left.value!.clientWidth;
	const height = left.value!.clientHeight;
	leftApp.renderer.resize(width, height);
	leftApp.stage.addChild(rulerContainerY);
	left.value?.appendChild(leftApp.view as any);
};
/**
 * zoomRef是为了修改zoom而增的变量
 * 根据数据的单向流通性，父组件传递的 zoom 不能直接在子组件中修改
 * 通过emit事件通知父组件修改
 */
const zoomRef = ref(zoom.value);
const emits = defineEmits(["onMouseWheel"]);

const calculateOffset = (offset: offset, currScale: number, prevScale: number) => {
	// 获取鼠标、画布中心的位置
	const { gx, gy, cx, cy } = offset;
	// 计算缩放比例
	const scaleDiff = currScale / prevScale;
	// 计算偏移值，根据变换矩阵来计算
	const dx = (gx - cx) * (1 - scaleDiff);
	const dy = (gy - cy) * (1 - scaleDiff);
	// container做相对应的移动、缩放
	ParentContainer.scale.set(currScale);
	ParentContainer.x += dx;
	ParentContainer.y += dy;
	totalOffset.x -= dx;
	totalOffset.y -= dy;
};
// 按照中心点来缩放
const scaleCenter = (newZoom: number) => {
	const { width: gx, height: gy } = mainApp.screen;
	const { x: cx, y: cy } = ParentContainer;
	const prevScale = zoomRef.value;
	calculateOffset({ gx, gy, cx, cy }, newZoom, prevScale);
};

// 按照鼠标位置滚轮缩放
const onMouseWheel = (e: PIXI.FederatedWheelEvent) => {
	// 获取鼠标的位置
	const { x: gx, y: gy } = e.global;
	// 获取container的位置
	const { x: cx, y: cy } = ParentContainer;
	// 最后一次的zoom值
	const prevScale = zoom.value;
	// 鼠标滚轮的方向
	const deltaY = e.deltaY;
	// ↑为放大， ↓为缩小
	zoomRef.value += deltaY < 0 ? 0.1 : -0.1;
	// 最小的zoom值
	if (zoomRef.value < 0.2) {
		zoomRef.value = 0.2;
	}
	// 更新zoom的值
	emits("onMouseWheel", zoomRef.value);
	calculateOffset({ gx, gy, cx, cy }, zoomRef.value, prevScale);
};

// 拖拽画布的实现
let isDragging = false;
let lastPosition = new PIXI.Point();
const onDragStart = (e: PIXI.FederatedPointerEvent) => {
	e.preventDefault();
	isDragging = true;
	// 设置起初坐标
	const { x, y } = e.global;
	lastPosition.set(x, y);
};
const onDrag = (e: PIXI.FederatedPointerEvent) => {
	if (isDragging) {
		const { x: globalX, y: globalY } = e.global;
		// 滑动的距离
		const dx = (globalX - lastPosition.x) / zoom.value;
		const dy = (globalY - lastPosition.y) / zoom.value;
		totalOffset.x += dx;
		totalOffset.y += dy;
		ParentContainer.x -= dx;
		ParentContainer.y -= dy;
		updateRuler(zoom.value, totalOffset);
		lastPosition.set(globalX, globalY);
	}
};
const throttleOnDrag = throttle(onDrag, 50);

const onDragEnd = () => {
	isDragging = false;
};

// 标尺容器
const rulerContainerX = new PIXI.Container();
const rulerContainerY = new PIXI.Container();

// 定义标尺的样式
const setting = {
	rulerMarkSize: 10,
	rulerMarkStroke: 0x000000,
};
// 刻度绘制的位置
const rulerX = 20;
const rulerY = 20;

const mainAppRectBound = {
	width: 0,
	height: 0,
};
// 视口位置
const viewport = {
	x: 0,
	y: 0,
	width: 0,
	height: 0,
};
// 场景坐标
const getSceneCoords = (position: Positon = { x: 0, y: 0 }) => {
	const { width, height } = mainAppRectBound;
	viewport.x = position.x / zoom.value;
	viewport.y = position.y / zoom.value;
	viewport.width = width!;
	viewport.height = height!;
};
/**
 * 标尺更新
 * @param zoom 缩放比
 * @param position 画布所在的位置
 */
const updateRuler = (zoom: number, position: Positon = { x: 0, y: 0 }) => {
	getSceneCoords(position);
	const tempContainerX = new PIXI.Container();
	const tempGraphicsX = new PIXI.Graphics();
	// X 坐标的标尺
	// 清空之前的文本元素
	const newStep = getStepByZoom(zoom);
	const startMarkX = getClosestVal(viewport.x, newStep);
	const endMarkX = getClosestVal(Math.ceil(viewport.x + viewport.width / zoom), newStep);

	tempGraphicsX.lineStyle(1, setting.rulerMarkStroke);
	for (let x = startMarkX; x <= endMarkX; x += newStep) {
		// 刻度线,刻度线不能直接在场景中绘制，因为缩放变换会导致线的粗细变化
		const posX = (x - viewport.x) * zoom;
		tempGraphicsX.moveTo(posX, rulerY + setting.rulerMarkSize);
		tempGraphicsX.lineTo(posX, rulerY);
		tempGraphicsX.endFill();
		// 刻度值
		const text = new PIXI.Text(String(x), { fontSize: 12, fill: setting.rulerMarkStroke });
		text.x = posX;
		text.y = 0;
		text.anchor.set(0.5, 0);
		tempContainerX.addChild(text);
	}
	rulerContainerX.removeChildren();
	rulerContainerX.addChild(tempGraphicsX);
	rulerContainerX.addChild(tempContainerX);

	// Y 坐标的标尺
	const tempContainerY = new PIXI.Container();
	const tempGraphicsY = new PIXI.Graphics();
	tempContainerY.removeChildren(); // 清空之前的文本元素
	const startMarkY = getClosestVal(viewport.y, newStep);
	const endMarkY = getClosestVal(Math.ceil(viewport.y + viewport.height / zoom), newStep);
	tempGraphicsY.lineStyle(1, setting.rulerMarkStroke);
	for (let y = startMarkY; y <= endMarkY; y += newStep) {
		const posY = (y - viewport.y) * zoom;
		tempGraphicsY.moveTo(rulerX, posY);
		tempGraphicsY.lineTo(rulerX + setting.rulerMarkSize, posY);
		tempGraphicsY.endFill();
		const text = new PIXI.Text(String(y), { fontSize: 12, fill: setting.rulerMarkStroke });
		text.x = rulerX - 20;
		text.y = posY;
		text.anchor.set(0.5, 0);
		text.rotation = -(Math.PI / 2);
		tempContainerY.addChild(text);
	}
	rulerContainerY.removeChildren();
	rulerContainerY.addChild(tempGraphicsY);
	rulerContainerY.addChild(tempContainerY);
};

interface Group {
	type: "group";
	structName: string;
	children?: Item[];
}
interface Path {
	type: "path";
	datatype: number;
	pathtype: number;
	width: number;
	layer: number;
	path: number[][];
}
interface Box {
	type: "box";
	id: string;
	layer: number;
	path: number[][];
}
type Item = Group | Path | Box;
interface GroupedData {
	[layer: number]: Item[];
}

// 分组的数据
let groupedData: GroupedData = {};

// layerInfo 的映射
const layerInfoMap: Record<string, number> = {};
// layerContainer 的映射
const layerContainerMap: Map<number, PIXI.Container> = new Map();

// 每一层创建一个container
const createChildContainer = (value: Item[]) => {
	const childContainer = new PIXI.Container();
	const Graphics = new PIXI.Graphics();
	for (const item of value) {
		const object = createObjectFromElement(item, Graphics);
		childContainer.addChild(object);
	}
	return childContainer;
};

// setTimeout(() => {
// 	fetch("/js/final_design.json")
// 		.then((res) => res.json())
// 		.then((data) => {
// 			for (const ele of data.layerInfo) {
// 				layerInfoMap[ele.layername] = ele.id;
// 			}
// 			groupedData = groupDataByLayer(data.data);
// 			const entries: [string, Item[]][] = Object.entries(groupedData);
// 			for (const [key, value] of entries) {
// 				const childContainer = createChildContainer(value);
// 				layerContainerMap.set(Number(key), childContainer);
// 				ParentContainer.addChild(childContainer);
// 			}
// 			mainApp.renderer.render(ParentContainer);
// 		});
// }, 100);
bus.on("jsonLoaded", (data: any) => {
	console.log("bus on", data);
	data = JSON.parse(data);
	for (const ele of data.layerInfo) {
		layerInfoMap[ele.layername] = ele.id;
	}

	groupedData = groupDataByLayer(data.data);
	const entries: [string, Item[]][] = Object.entries(groupedData);
	const logId = `rerender-${Math.random().toString(16).slice(2)}`;
	console.time(logId);
	console.log("开始清除");
	console.timeLog(logId);
	for (const [key, value] of entries) {
		const childContainer = createChildContainer(value);
		layerContainerMap.set(Number(key), childContainer);
		ParentContainer.addChild(childContainer);
	}
	mainApp.renderer.render(ParentContainer);
	console.log("渲染完毕");
	console.timeLog(logId);
});

// 筛选不同的层匹配对应的Container
const matchLayerContainers = (layers: string[]) => {
	const layerContainers: PIXI.Container[] = [];
	if (layers.length === 0) {
		// 没有勾选任何层，说明是渲染全部内容
		[...layerContainerMap.entries()].forEach((container) => {
			layerContainers.push(container[1]);
		});
	} else {
		for (const layer of layers) {
			const layerId = layerInfoMap[layer];
			const layerContainer = layerContainerMap.get(layerId);
			if (layerContainer !== undefined) layerContainers.push(layerContainer);
		}
	}
	return layerContainers;
};
// 根据匹配的Container重新渲染
const reRender = (containers: PIXI.Container[]) => {
	ParentContainer.removeChildren();
	containers.forEach((container) => {
		ParentContainer.addChild(container);
	});
	mainApp.renderer.render(ParentContainer);
};

// 路劲path转化
function parse(num: number) {
	return parseFloat((num / 100).toFixed(3));
}

// 根据 layer 将数据分组
const groupDataByLayer = (data: Item[]) => {
	const groupedData: GroupedData = {};
	for (const item of data) {
		if (item.type === "group") {
			if (item.children) {
				const childrenData = groupDataByLayer(item.children);
				mergeGroupedData(groupedData, childrenData);
			}
		} else if (item.type === "box" || item.type === "path") {
			if (item.layer !== undefined) {
				groupedData[item.layer] = groupedData[item.layer] || [];
				groupedData[item.layer].push(item);
			}
		}
	}
	return groupedData;
};

// 合并groupedData的数据
const mergeGroupedData = (target: GroupedData, source: GroupedData) => {
	for (const layer in source) {
		target[layer] = target[layer] || [];
		target[layer].push(...source[layer]);
	}
};

type Carrier = PIXI.Graphics | PIXI.Container;
const createObjectFromElement = (element: Item, carrier: Carrier = new PIXI.Graphics()): Carrier => {
	let object;
	switch (element.type) {
		case "path":
			object = carrier as PIXI.Graphics;
			object.lineStyle(element.width!, 0xffffff);
			object.moveTo(element.path![0][0], element.path![0][1]);
			for (let i = 1; i < element.path!.length; i++) {
				const x = element.path![i][0];
				const y = element.path![i][1];
				object.lineTo(x, y);
			}
			break;
		case "box": {
			object = carrier as PIXI.Graphics;
			object.lineStyle(0.2, 0xffffff);
			object.moveTo(parse(element.path![0][0]), parse(element.path![0][1]));
			for (let i = 1; i < element.path!.length; i++) {
				object.lineTo(parse(element.path![i][0]), parse(element.path![i][1]));
			}
			object.closePath();
			break;
		}
		case "group": {
			object = carrier;
			for (const childElement of element.children!) {
				const childGraphics = createObjectFromElement(childElement, new PIXI.Container());
				object.addChild(childGraphics!);
			}
			break;
		}
		default:
			// 未知的元素类型
			object = new PIXI.Graphics();
			break;
	}
	return object;
};

// 自适应
function resizeRender() {
	const mainClientRects = main.value!.getClientRects();
	const { width, height } = mainClientRects[0];
	// 更新
	mainAppRectBound.width = width;
	mainAppRectBound.height = height;
	mainApp.renderer.resize(width, height);
	topApp.renderer.resize(width, 22);
	leftApp.renderer.resize(22, height);
	updateRuler(zoom.value, totalOffset);
}
const throttleResizeRender = throttle(resizeRender, 100);
window.addEventListener("resize", throttleResizeRender);

onMounted(() => {
	setupTopApp();
	setupMainApp();
	setupLeftApp();
	updateRuler(zoom.value);
});
</script>

<style scoped>
.container {
	height: 60vh;
	width: 100%;
	position: relative;
	overflow: hidden;
}
.top {
	position: absolute;
	left: 22px;
	width: calc(100% - 23px);
	height: 22px;
	overflow: hidden;
}
.left {
	position: absolute;
	top: 22px;
	width: 22px;
	height: calc(100% - 23px);
	overflow: hidden;
}
.main {
	position: absolute;
	top: 22px;
	left: 22px;
	z-index: 99;
	width: calc(100% - 23px);
	height: calc(100% - 23px);
	overflow: hidden;
	display: flex;
	align-items: center;
	justify-content: center;
	.container {
		display: flex;
		align-items: center;
		justify-content: center;
		height: 100%;
		width: 100%;
		overflow: hidden;
	}
}
</style>

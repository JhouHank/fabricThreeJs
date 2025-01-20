<template>
  <!-- 你可以直接使用 <canvas> 或 <div> 做容器，視需求調整 -->
  <canvas
    ref="canvasRef"
    :width="canvasWidth"
    :height="canvasHeight"
    style="border: 1px solid #ccc"
  ></canvas>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { fabric } from 'fabric';
import sofaUrl from '../assets/sofa.glb?url';
import model3 from '../assets/model3.glb?url';
import model4 from '../assets/model4.glb?url';

// Canvas 尺寸
const canvasWidth = 1500;
const canvasHeight = 800;

// 取用 <canvas ref="canvasRef">
const canvasRef = ref(null);

// 在元件掛載後初始化
onMounted(() => {
  // 一、先建立 Fabric.js 的 canvas
  const fabricCanvas = new fabric.Canvas(canvasRef.value, {
    backgroundColor: '#f0f0f0',
  });

  // 二、自訂類別繼承 fabric.Object
  const ThreeDBox = fabric.util.createClass(fabric.Object, {
    type: 'threeDBox',

    initialize(options) {
      this.callSuper('initialize', options);
      // Fabric 物件尺寸
      this.width = options.width || 300;
      this.height = options.height || 300;

      // 3D 旋轉角度 (繞X, Y, Z)
      this.rotate3dX = 0;
      this.rotate3dY = 0;
      this.rotate3dZ = 0;

      // 建立離線 Canvas + Three.js Renderer
      this.webglCanvas = document.createElement('canvas');
      this.webglCanvas.width = this.width;
      this.webglCanvas.height = this.height;
      this.renderer = new THREE.WebGLRenderer({
        canvas: this.webglCanvas,
        alpha: true,
      });
      this.renderer.setSize(this.width, this.height);

      // 建立 Three.js 場景、相機
      this.scene = new THREE.Scene();
      this.camera = new THREE.PerspectiveCamera(
        30,
        this.width / this.height,
        0.1,
        2000
      );
      this.camera.position.set(0, 0, 5);

      // 簡單光源
      const light = new THREE.AmbientLight(0xffffff, 10);
      this.scene.add(light);

      // 載入 GLTF 模型 (若你有 sofa.glb 就放在 src/assets/sofa.glb)
      this.loader = new GLTFLoader();
      this.model = null;

      this.loader.load(
        model3,
        // model4,
        // sofaUrl,
        (gltf) => {
          this.model = gltf.scene;

          // 1) 計算模型的包圍盒 Bounding Box
          const box3 = new THREE.Box3().setFromObject(this.model);
          const center = new THREE.Vector3();
          box3.getCenter(center); // 取得幾何中心 (世界座標)

          // 2) 將整個模型 shift，使得幾何中心位於 (0,0,0)
          this.model.position.sub(center);

          // （可選）若想控制縮放，讓它更好地置於鏡頭
          const size = box3.getSize(new THREE.Vector3()).length();
          const desiredSize = 2; // 你希望它最大外徑大約是 2 (或其他值)
          const scaleFactor = desiredSize / size;
          this.model.scale.set(scaleFactor, scaleFactor, scaleFactor);

          // 載入完成後：
          this.model.updateMatrixWorld(true);

          const overallBox = new THREE.Box3();

          // 遍歷所有子物件
          this.model.traverse((child) => {
            if (child.isMesh) {
              // 先確保 geometry boundingBox 更新
              child.geometry.computeBoundingBox();
              // 取得該 Mesh 在世界座標的包圍盒
              const tempBox = new THREE.Box3().setFromObject(child);
              overallBox.union(tempBox);
            }
          });

          const center2 = overallBox.getCenter(new THREE.Vector3());
          this.model.position.sub(center2);

          const dirLight = new THREE.DirectionalLight(0xffffff, 5);
          dirLight.position.set(1, 1, 1); // 光源位置
          this.scene.add(dirLight);

          // 最後再加到場景
          this.scene.add(this.model);
        },
        undefined,
        (error) => {
          console.error('GLTF load error:', error);
        }
      );
    },

    // override _render
    _render(ctx) {
      // 如果有載入 glTF 模型
      if (this.model) {
        this.model.rotation.x = this.rotate3dX;
        this.model.rotation.y = this.rotate3dY;
        this.model.rotation.z = this.rotate3dZ;
      }

      // 若有加其他 shape，如 this.cube，也可一起做 3D 旋轉
      // if (this.cube) {
      //   this.cube.rotation.x = this.rotate3dX
      //   this.cube.rotation.y = this.rotate3dY
      //   this.cube.rotation.z = this.rotate3dZ
      // }

      this.renderer.render(this.scene, this.camera);
      // 將結果繪製到 Fabric 2D context
      const w = this.width;
      const h = this.height;
      ctx.drawImage(this.webglCanvas, -w / 2, -h / 2, w, h);
    },
  });

  // 建立 3DBox 物件，加入 Fabric Canvas
  const box = new ThreeDBox({
    left: 300,
    top: 300,
    width: 300,
    height: 300,
    originX: 'center',
    originY: 'center',
  });
  fabricCanvas.add(box);

  // 預設保留 Fabric 所有控制點 (可移動、縮放、旋轉...)
  // 若需要自訂 3D 旋轉控制柄
  const rotate3DControl = new fabric.Control({
    x: 0,
    y: -0.5,
    offsetY: -40,
    cursorStyle: 'move',
    actionName: 'rotate3D',
    render(ctx, left, top) {
      ctx.save();
      ctx.beginPath();
      ctx.arc(left, top, 10, 0, 2 * Math.PI);
      ctx.fillStyle = 'orange';
      ctx.fill();
      ctx.strokeStyle = 'black';
      ctx.stroke();
      ctx.restore();
    },
    actionHandler(eventData, transform, x, y) {
      const target = transform.target;
      const c = target.canvas;
      const pointer = c.getPointer(eventData.e);
      if (!transform.prevPointer) {
        transform.prevPointer = pointer;
        return true;
      }
      const dx = pointer.x - transform.prevPointer.x;
      const dy = pointer.y - transform.prevPointer.y;
      // 更新 3D 旋轉
      target.rotate3dY += dx * 0.01;
      target.rotate3dX -= dy * 0.01;

      target.set('dirty', true);
      c.requestRenderAll();
      transform.prevPointer = pointer;
      return true;
    },
    actionPerformed(eventData, transform) {
      transform.prevPointer = null;
    },
  });

  // 合併自訂 3D 控制柄
  const defaultControls = fabric.Object.prototype.controls;
  box.controls = {
    ...defaultControls,
    rotate3D: rotate3DControl,
  };

  fabricCanvas.on('object:scaling', function (e) {
    const obj = e.target;
    if (obj.type === 'threeDBox') {
      // 計算新的寬高 (Fabric 物件縮放後的實際尺寸)
      const newWidth = obj.width * obj.scaleX;
      const newHeight = obj.height * obj.scaleY;

      // 重新設定 offscreen Canvas + Three.js Renderer 大小
      obj.webglCanvas.width = newWidth;
      obj.webglCanvas.height = newHeight;
      obj.renderer.setSize(newWidth, newHeight);

      // 如果你在 Fabric 物件中，也把 width/height 更新回來
      // (可視需求: 可能先把 scaleX/scaleY 設回 1，再直接套 newWidth/newHeight)
      obj.set({
        width: newWidth,
        height: newHeight,
        scaleX: 1,
        scaleY: 1,
      });

      obj.dirty = true;
      fabricCanvas.renderAll();
    }
  });
});
</script>

<style scoped>
/* 如有需要在此添加元件專屬的 CSS */
</style>

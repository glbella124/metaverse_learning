<template>
  <div id="container"></div>
</template>

<script setup>
/**
 * 1. 元宇宙物理环境搭建
 *
 * 2. 元宇宙碰撞检测实现
 *
 * 3. 控制物体运动阻尼
 *
 * 4. 运用叉积控制左右水平运动
 */
import * as THREE from "three";
import Stats from "three/examples/jsm/libs/stats.module.js";

import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

// 八叉树空间 -- Capsule, Octree,八叉树空间的一切碰撞检测和重力都靠这两个库实现
// 整体来说，整个八叉树空间就是Capsule 和 Octree运算
import { Octree } from "three/examples/jsm/math/Octree.js";
// 辅助器查看如何划分空间
import { OctreeHelper } from "three/examples/jsm/helpers/OctreeHelper.js";

import { Capsule } from "three/examples/jsm/math/Capsule.js";

import { GUI } from "three/examples/jsm/libs/lil-gui.module.min.js";

import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { LightningStorm } from "three/examples/jsm/objects/LightningStorm.js";

import { reactive, onMounted, ref } from "vue";

onMounted(() => {
  const clock = new THREE.Clock();
  // 初始化场景
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x88ccee);
  scene.fog = new THREE.Fog(0x88ccee, 0, 50);

  const camera = new THREE.PerspectiveCamera(
    70,
    window.innerWidth / window.innerHeight,
    0.001,
    1000
  );
  camera.position.set(0, 5, 10);

  // 后置摄像头
  const backCamera = new THREE.PerspectiveCamera(
    70,
    window.innerWidth / window.innerHeight,
    0.001,
    1000
  );

  camera.position.set(0, 5, -10);

  const container = document.getElementById("container");

  // 渲染器渲染
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.VSMShadowMap;
  renderer.outputEncoding = THREE.sRGBEncoding;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  container.appendChild(renderer.domElement);

  const stats = new Stats();
  stats.domElement.style.position = "absolute";
  stats.domElement.style.top = "0px";
  container.appendChild(stats.domElement);

  // 控制器
  // const controls = new OrbitControls(camera, renderer.domElement);
  // controls.target.set(0, 0, 0);

  // console.log(renderer.info);

  // 设置激活相机，以便切换视角
  let activeCamera = camera;

  function animate() {
    let delta = clock.getDelta();
    controlPlayer(delta);
    updatePlayer(delta);
    resetPlayer();
    stats.update();
    // controls.update();
    renderer.render(scene, activeCamera);
    requestAnimationFrame(animate);
  }

  // 创建一个平面
  const planeGeometry = new THREE.PlaneGeometry(20, 20, 1, 1);
  const planeMaterial = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    side: THREE.DoubleSide,
  });

  const plane = new THREE.Mesh(planeGeometry, planeMaterial);
  plane.receiveShadow = true;
  plane.rotation.x = -Math.PI / 2;

  // 循环创建叠楼梯效果
  for (let i = 0; i < 8; i++) {
    const boxGeometry = new THREE.BoxGeometry(1, 1, 0.15);
    const boxMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
    const box = new THREE.Mesh(boxGeometry, boxMaterial);
    box.position.y = 0.15 + i * 0.15;
    box.position.z = i * 0.4;
    plane.add(box);
  }

  // 创建一个octree -- 世界空间
  // 创建一个地面组
  const group = new THREE.Group();
  // 将平面添加到组里面
  group.add(plane);
  scene.add(group);
  const worldOctree = new Octree();
  // 将八叉树划分
  worldOctree.fromGraphNode(group);

  // 创建一个octreeHelper
  // const octreeHelper = new OctreeHelper(worldOctree);
  // scene.add(octreeHelper);

  // 创建一个人的碰撞体
  const playerCollider = new Capsule(
    // 人底部，0.35半径 -- 起点
    new THREE.Vector3(0, 0.35, 0),
    // 身高 -- 终点
    new THREE.Vector3(0, 1.35, 0),
    0.35
  );

  // console.log(playerCollider);
  // console.log(worldOctree);

  // 创建一个平面
  const capsuleBodyGeometry = new THREE.PlaneGeometry(1, 0.5, 1, 1);
  const capsuleBodyMaterial = new THREE.MeshBasicMaterial({
    color: 0x0000ff,
    side: THREE.DoubleSide,
  });
  const capsuleBody = new THREE.Mesh(capsuleBodyGeometry, capsuleBodyMaterial);
  capsuleBody.position.set(0, 0.5, 0);

  // 创建一个胶囊物体
  const capsuleGeometry = new THREE.CapsuleGeometry(0.35, 1, 32);
  const capsuleMaterial = new THREE.MeshBasicMaterial({
    color: 0xffff00,
    side: THREE.DoubleSide,
  });

  const capsule = new THREE.Mesh(capsuleGeometry, capsuleMaterial);
  capsule.position.set(0, 0.85, 0);

  // 将相机作为胶囊的子元素，就可以实现跟随
  camera.position.set(0, 2, -5);
  camera.lookAt(capsule.position);

  // 为切换视角，相机需要初始化
  backCamera.position.set(0, 2, 5);
  backCamera.lookAt(capsule.position);
  // controls.target = capsule.position;
  // 控制旋转上下的空3d对象
  // 注：胶囊 - 空对象 - 相机，上下旋转只控制空对象，不会影响物体
  const capsuleBodyControl = new THREE.Object3D();
  capsuleBodyControl.add(camera);
  capsuleBodyControl.add(backCamera);
  capsule.add(capsuleBodyControl);
  capsule.add(capsuleBody);

  scene.add(capsule);

  // 设置重力加速度 -- 竖直方向
  const gravity = -9.8;
  // 玩家的速度
  const playerVelocity = new THREE.Vector3(0, 0, 0);
  // 方向向量
  const playerDirection = new THREE.Vector3(0, 0, 0);

  // 键盘按下事件
  const keyStates = {
    KeyW: false,
    KeyA: false,
    KeyS: false,
    keyD: false,
    Space: false,
    isDown: false,
  };
  // 用于判断玩家是否在地面上
  let playerIsOnFloor = false;

  // 每一帧都需要更新玩家的状态 -- 延迟时间
  // 玩家在地面上时还需要给物体增加摩擦力
  function updatePlayer(deltaTime) {
    let damping = -0.05;
    if (playerIsOnFloor) {
      // 在地面上的话，不设置重力加速度
      playerVelocity.y = 0;
      // 添加反方向的力 -- 阻尼 -- 向量计算
      // addScaledVector -- 将所传入的v 与s相乘所得的乘积和这个向量相加
      // 反方向的摩擦力 -- 键盘未按下时执行
      keyStates.isDown ||
        playerVelocity.addScaledVector(playerVelocity, damping);
    } else {
      playerVelocity.y += gravity * deltaTime;
    }

    // console.log(playerVelocity);
    // 计算玩家移动的距离
    const playerMoveDistance = playerVelocity.clone().multiplyScalar(deltaTime);
    // 碰撞体进行移动
    playerCollider.translate(playerMoveDistance);
    // 设置胶囊位置 -- 传入一个三维向量
    playerCollider.getCenter(capsule.position);

    // 进行碰撞检测
    playerCollisions();
  }

  // 玩家碰撞检测
  function playerCollisions() {
    const result = worldOctree.capsuleIntersect(playerCollider);
    playerIsOnFloor = false;
    if (result) {
      // 由于速度太快，尽管有抬升，依然在持续下降
      // 法向向量向上的时候，设置为true
      playerIsOnFloor = result.normal.y > 0;
      // 方向 * 陷进去的深度
      playerCollider.translate(result.normal.multiplyScalar(result.depth));
    }
    // 碰撞到的话可以获取一个向量
    // console.log(result);
  }

  // 重置 -- 从天上往下掉
  function resetPlayer() {
    if (capsule.position.y < -20) {
      playerCollider.start.set(0, 2.35, 0);
      playerCollider.end.set(0, 3.35, 0);
      playerCollider.radius = 0.35;
      // 速度和方向需要恢复
      playerVelocity.set(0, 0, 0);
      playerDirection.set(0, 0, 0);
    }
  }

  // 监听键盘按下的事件
  document.addEventListener(
    "keydown",
    (event) => {
      console.log(event.code);
      keyStates[event.code] = true;
      keyStates.isDown = true;
    },
    false
  );

  // 监听键盘抬起的事件
  document.addEventListener(
    "keyup",
    (event) => {
      keyStates[event.code] = false;
      keyStates.isDown = false;
      if (event.code === "KeyV") {
        activeCamera = activeCamera === camera ? backCamera : camera;
      }
    },
    false
  );

  document.addEventListener(
    "mousedown",
    (event) => {
      // 锁定鼠标指针
      document.body.requestPointerLock();
    },
    false
  );

  // 根据键盘状态更新玩家的速度
  function controlPlayer(deltaTime) {
    // 前进
    if (keyStates["KeyW"]) {
      playerDirection.z = 1;
      // 获取胶囊的正前面方向
      const capsuleFront = new THREE.Vector3(0, 0, 1);
      capsule.getWorldDirection(capsuleFront);
      // console.log(capsuleFront, "获取胶囊的正前方方向");
      // 计算玩家的速度
      playerVelocity.add(capsuleFront.multiplyScalar(deltaTime));
    }

    // 后退
    if (keyStates["KeyS"]) {
      playerDirection.z = 1;
      const capsuleFront = new THREE.Vector3(0, 0, 0);
      capsule.getWorldDirection(capsuleFront);
      playerVelocity.add(capsuleFront.multiplyScalar(-deltaTime));
    }

    // 左右侧向
    if (keyStates["KeyA"]) {
      playerDirection.x = 1;
      // 获取胶囊的正前面方向
      const capsuleFront = new THREE.Vector3(0, 0, 0);
      capsule.getWorldDirection(capsuleFront);
      // 侧方的方向，正前面的方向和胶囊的正上方求叉积，求出侧方的方向
      capsuleFront.cross(capsule.up);
      // 计算玩家的速度
      playerVelocity.add(capsuleFront.multiplyScalar(-deltaTime));
    }

    if (keyStates["KeyD"]) {
      playerDirection.x = 1;
      // 获取胶囊的正前面方向
      const capsuleFront = new THREE.Vector3(0, 0, 0);
      capsule.getWorldDirection(capsuleFront);
      // 侧方的方向，正前面的方向和胶囊的正上方求叉积，求出侧方的方向
      capsuleFront.cross(capsule.up);
      // 计算玩家的速度
      playerVelocity.add(capsuleFront.multiplyScalar(deltaTime));
    }

    // 按空格键进行跳跃
    if (keyStates["Space"]) {
      playerVelocity.y = 10;
    }
  }

  // 根据鼠标在屏幕上的移动，来旋转胶囊

  // window.addEventListener("mousemove", (event) => {
  //   const mouseX = event.clientX;
  //   const mouseY = event.clientY;
  //   const mouseDeltaX = mouseX - window.innerWidth / 2;
  //   const mouseDeltaY = mouseY - window.innerHeight / 2;
  //   // capsule.rotation.y -= mouseDeltaX * 0.00001;
  //   // 左右旋转
  //   capsule.rotation.y -= mouseDeltaX * 0.00001;
  // });

  // movementX -- 根据movement旋转胶囊，移动距离越大，旋转角度越大
  window.addEventListener(
    "mousemove",
    (event) => {
      capsule.rotation.y -= event.movementX * 0.005;
      capsuleBodyControl.rotation.x += event.movementY * 0.003;

      // 设置旋转的阈值
      if (capsuleBodyControl.rotation.x > Math.PI / 8) {
        capsuleBodyControl.rotation.x = Math.PI / 8;
      } else if (capsuleBodyControl.rotation.x < -Math.PI / 8) {
        capsuleBodyControl.rotation.x = -Math.PI / 8;
      }
    },
    false
  );

  // 多层次细节展示 LOD
  // 提升了渲染性能，相机距离的远近决定了物体展示的精细程度
  let lod = new THREE.LOD();
  const lodMaterial = new THREE.MeshBasicMaterial({
    color: 0xff0000,
    wireframe: true,
  });

  for (let i = 0; i < 5; i++) {
    const geometry = new THREE.SphereBufferGeometry(i, 22 - i * 5, 22 - i * 5);
    const mesh = new THREE.Mesh(geometry, lodMaterial);
    lod.addLevel(mesh, i * 5);
  }

  lod.position.set(10, 0, 10);
  scene.add(lod);

  animate();
});
</script>

<style>
* {
  margin: 0;
  padding: 0;
}
#container {
  width: 100vw;
  height: 100vh;
}
</style>

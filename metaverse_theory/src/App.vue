<template>
  <div id="container"></div>
</template>

<script setup>
/**
 * 元宇宙物理环境搭建
 *
 * 元宇宙碰撞检测实现
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
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.target.set(0, 0, 0);

  // console.log(renderer.info);

  function animate() {
    let delta = clock.getDelta();
    controlPlayer(delta);
    updatePlayer(delta);
    resetPlayer();
    stats.update();
    controls.update();
    renderer.render(scene, camera);
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
  scene.add(plane);

  // 创建一个octree -- 世界空间
  // 创建一个地面组
  const group = new THREE.Group();
  // 将平面添加到组里面
  group.add(plane);
  scene.add(group);
  const worldOctree = new Octree();
  // 将八叉树划分
  worldOctree.fromGraphNode(group);

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

  // 创建一个胶囊物体
  const capsuleGeometry = new THREE.CapsuleGeometry(0.35, 1, 32);
  const capsuleMaterial = new THREE.MeshBasicMaterial({
    color: 0xffff00,
    side: THREE.DoubleSide,
  });

  const capsule = new THREE.Mesh(capsuleGeometry, capsuleMaterial);
  capsule.position.set(0, 0.85, 0);
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
    },
    false
  );

  // 根据键盘状态更新玩家的速度
  function controlPlayer(deltaTime) {
    if (keyStates["KeyW"]) {
      playerDirection.z = 1;
      // 获取胶囊的正前面方向
      const capsuleFront = new THREE.Vector3(0, 0, 1);
      capsule.getWorldDirection(capsuleFront);
      // console.log(capsuleFront, "获取胶囊的正前方方向");
      // 计算玩家的速度
      playerVelocity.add(capsuleFront.multiplyScalar(deltaTime));
    }
  }

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

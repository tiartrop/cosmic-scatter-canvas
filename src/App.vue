<script setup lang="ts">
import { computed, onMounted, useTemplateRef } from 'vue';

const WIDTH = document.documentElement.clientWidth
const HEIGHT = document.documentElement.clientHeight
const RADIUS = 40
const initPoint = { x: WIDTH / 2, y: HEIGHT / 2 }

interface Point {
  x: number
  y: number
}

interface Branch {
  start: Point
  length: number
  angle: number
}

const el = useTemplateRef<HTMLCanvasElement>('el');
let ctx: CanvasRenderingContext2D

// 存储活动弧线的角度范围
let closeAngles: number[][] = []

// 动画状态管理
let currentBranches: Branch[] = [];
let currentProbability = 50;
let currentLength = RADIUS;

// 任务队列管理
// eslint-disable-next-line @typescript-eslint/no-unsafe-function-type
const pendingTasks: Function[] = []

// 动画帧计数
let framesCount = 0

// ========== 初始化函数 ==========
function init() {
  // 初始化canvas上下文
  ctx = computed(() => el.value!.getContext('2d')!).value;
  ctx.imageSmoothingEnabled = true;
  ctx.imageSmoothingQuality = 'high';
  ctx.strokeStyle = "#000"
  ctx.lineWidth = 0.5

  // 生成初始角度分布
  const minAngleDifference = 4;
  const angleCount = Math.floor(Math.random() * 7) + 30;
  const angles = generateAngles(angleCount, minAngleDifference);

  // 创建初始分支点并绘制从中心到边缘的射线
  const circlePoints: Branch[] = [];
  for (const angle of angles) {
    const endPoint = computeEndpoint(initPoint.x, initPoint.y, RADIUS, angle)
    circlePoints.push({ start: endPoint, length: RADIUS, angle })
    line(initPoint, endPoint)
  }

  // 按角度排序以便后续处理
  circlePoints.sort((a, b) => a.angle - b.angle);

  // 初始化活动弧线范围
  closeAngles = arcFromPoint(initPoint.x, initPoint.y, circlePoints, 25, 0)

  // 设置动画初始状态
  currentBranches = circlePoints;
  currentProbability = 50;
  currentLength = RADIUS;

  step();
}

// ========== 动画控制函数 ==========
function step() {
  // 终止条件检查
  if (currentBranches.length === 0 || currentLength > 1000)  return;

  if (currentProbability >= 100) currentProbability = 90

  const newBranch: Branch[] = [];
  const angles: number[] = [];

  // 为每个当前分支创建新的生长任务
  for (const { angle } of currentBranches) {
    const isInArcRange = closeAngles.find(arr => arr[0] >= angle && arr[1] <= angle)
    pendingTasks.push(() =>
      radialLine(
        0.01 * currentProbability * (isInArcRange ? 1 : 0.8),
        0.1 * (100 - currentProbability),
        angle,
        currentLength,
        angles,
        newBranch
      )
    )
  }

  // 添加状态更新任务
  pendingTasks.push(() => {
    // 标准化角度并排序
    newBranch.forEach(n => (n.angle + 360) % 360);
    newBranch.sort((a, b) => a.angle - b.angle);

    // 更新活动弧线范围
    if (currentProbability < 90) {
      closeAngles = arcFromPoint(initPoint.x, initPoint.y, newBranch, currentProbability, currentProbability);
    }

    // 更新下一步的状态参数
    currentBranches = newBranch;
    currentProbability += 8;
    currentLength = Math.min(currentLength + RADIUS, 1000);
  });
}

function frame() {
  // 执行所有待处理的任务
  const tasks = [...pendingTasks]
  pendingTasks.length = 0
  tasks.forEach((fn) => fn())

  step();
}

function startFrame() {
  requestAnimationFrame(() => {
    framesCount += 1
    if (framesCount % 3 === 0) {
      frame()
    }
    startFrame()
  })
}

// ========== 分支生成函数 ==========
function radialLine(random: number, count: number, angle: number, length: number, angles: number[], endPoints: Branch[]) {
  // 根据随机概率决定是否生成新分支
  if (Math.random() > random) {
    const copyArr = copyRandom(Math.floor(Math.random() * 2) + Math.floor(count))
    const dir = Math.floor(Math.random() * (copyArr.length + 1))
    if (dir > 1) copyArr.forEach(c => c - dir)

    // 为每个随机偏移创建新分支
    copyArr.forEach((d) => {
      const _angle = angle - 3 * d
      const _start = computeEndpoint(initPoint.x, initPoint.y, length, _angle)
      const _end = computeEndpoint(initPoint.x, initPoint.y, length + RADIUS, _angle)

      // 检查角度是否有效（避免重叠）
      if (angles.every(existingAngle => isAngleValid(_angle, existingAngle))) {
        line(_start, _end)
        angles.push(_angle)
        endPoints.push({ start: _end, length: length + RADIUS, angle: _angle })
      }
    })
  }
}

// ========== 角度生成函数 ==========
function generateAngles(count: number, minDifference: number): number[] {
  const angles: number[] = [];
  let attempts = 0;
  const maxAttempts = 1000;

  // 确保每个象限至少有一个角度，保证分布均匀
  const quadrants = [
    [0, 90],    // 第一象限
    [90, 180],  // 第二象限
    [180, 270], // 第三象限
    [270, 360]  // 第四象限
  ];

  // 在每个象限中生成至少一个角度
  for (const [min, max] of quadrants) {
    while (true) {
      const newAngle = Math.floor(Math.random() * (max - min) + min);
      const isValid = angles.every(existingAngle => {
        const diff = Math.abs(newAngle - existingAngle);
        return Math.min(diff, 360 - diff) >= minDifference;
      });

      if (isValid) {
        angles.push(newAngle);
        break;
      }
      attempts++;
      if (attempts >= maxAttempts) return angles;
    }
  }

  // 生成剩余的角度
  while (angles.length < count && attempts < maxAttempts) {
    const newAngle = Math.floor(Math.random() * 360);
    const isValid = angles.every(existingAngle => {
      const diff = Math.abs(newAngle - existingAngle);
      return Math.min(diff, 360 - diff) >= minDifference;
    });

    if (isValid) {
      angles.push(newAngle);
    }
    attempts++;
  }

  return angles;
}

// ========== 弧线绘制函数 ==========
function arcFromPoint(centerX: number, centerY: number, branches: Branch[], activePercent: number, maxArcAngle: number): number[][] {
  const points = branches.map((b) => b.start)
  if (points.length < 2) return [];

  const arcAngles: number[][] = [];
  const totalPoints = points.length;
  const activeCount = Math.ceil((activePercent / 100) * totalPoints);
  const activeIndices = new Set<number>();

  // 随机选择活跃点的位置（保持起点固定）
  while (activeIndices.size < activeCount && activeIndices.size < totalPoints - 1) {
    const randomIndex = Math.floor(Math.random() * (totalPoints - 1)) + 1;
    activeIndices.add(randomIndex);
  }

  let startIdx = 0;

  for (let i = 0; i <= totalPoints; i++) {
    const isEnd = i === totalPoints;
    const isActive = activeIndices.has(i);

    // 遇到活跃点或终点时结束当前弧线段
    if (isEnd || isActive) {
      const endIdx = isEnd ? 0 : i;

      // 判断是否需要绘制这个弧线段
      const shouldDraw =
        (!isEnd && i - startIdx >= 1) ||
        (isEnd && (startIdx !== totalPoints || !activeIndices.has(totalPoints - 1)));

      if (shouldDraw) {
        ctx.beginPath();
        let segmentAngles: number[] = [];

        // 绘制弧线段中的每一对相邻点
        for (let j = startIdx; j !== endIdx; j = (j + 1) % totalPoints) {
          const current = points[j];
          const next = points[(j + 1) % totalPoints];

          const currentAngle = Math.atan2(current.y - centerY, current.x - centerX) * 180 / Math.PI;
          const nextAngle = Math.atan2(next.y - centerY, next.x - centerX) * 180 / Math.PI;
          const radius = Math.hypot(current.x - centerX, current.y - centerY);

          if (j === startIdx) {
            ctx.moveTo(current.x, current.y);
            segmentAngles = [
              Math.round((currentAngle + 360) % 360),
              Math.round((nextAngle + 360) % 360)
            ];
          }

          // 判断弧线绘制方向
          const clockwise = ((nextAngle - currentAngle + 360) % 360) < 180 ? false : true;

          // 角度差超过最大值时进行幸运判定
          if (maxArcAngle && ((nextAngle - currentAngle + 360) % 360) > maxArcAngle && (Math.random() > 1 - (maxArcAngle / 360))) continue;

          ctx.arc(centerX, centerY, radius, currentAngle * Math.PI / 180, nextAngle * Math.PI / 180, clockwise);
        }

        ctx.stroke();
        arcAngles.push(segmentAngles);
      }

      startIdx = i + 1;
    }
  }

  return arcAngles;
}

// ========== 工具函数 ==========
function isAngleValid(angle1: number, angle2: number, minDifference: number = 3): boolean {
  // 标准化角度到0-360范围
  angle1 = (angle1 + 360) % 360;
  angle2 = (angle2 + 360) % 360;

  // 计算最小角度差（考虑跨越0/360度的情况）
  const diff = Math.abs(angle1 - angle2);
  const minDiff = Math.min(diff, 360 - diff);

  return minDiff >= minDifference && angle1 !== angle2;
}

function copyRandom(n: number, zeroProbability: number = 0.1, exponent: number = 1): number[] {
  // 概率返回单个零值
  if (Math.random() < zeroProbability) {
    return [0];
  }

  // 使用权重分布选择数组长度
  const weights: number[] = [];
  for (let i = 1; i <= n; i++) {
    weights.push(Math.pow(i, exponent));
  }

  const totalWeight = weights.reduce((acc, w) => acc + w, 0);
  const rand = Math.random() * totalWeight;

  let cumulative = 0;
  let selectedLength = 0;
  for (let i = 0; i < n; i++) {
    cumulative += weights[i];
    if (rand <= cumulative) {
      selectedLength = i + 1;
      break;
    }
  }
  if (!selectedLength) selectedLength = n;

  // 生成连续数组
  return Array.from({ length: selectedLength + 1 }, (_, i) => i);
}

function line(p1: Point, p2: Point) {
  ctx.beginPath()
  ctx.moveTo(p1.x, p1.y)
  ctx.lineTo(p2.x, p2.y)
  ctx.stroke()
}

function computeEndpoint(x: number, y: number, length: number, angleDegrees: number) {
  const angleRadians = angleDegrees * Math.PI / 180;
  return {
    x: x + length * Math.cos(angleRadians),
    y: y + length * Math.sin(angleRadians)
  };
}

// ========== 生命周期 ==========
onMounted(() => {
  init()
})

// 启动动画循环
startFrame()

</script>

<template>
  <canvas ref="el" :width="WIDTH" :height="HEIGHT" />
</template>
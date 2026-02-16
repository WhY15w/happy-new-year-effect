<template>
  <div class="lantern-container pointer-events-none">
    <canvas ref="canvasRef" class="lantern-canvas"></canvas>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'

type LanternState = 'grounded' | 'rising' | 'gathered'

interface SpeedConfig {
  /** 上升基础速度（像素/帧） */
  riseSpeed: number
  /** 上升速度随机范围 */
  riseSpeedVar: number
  /** 水平摇摆幅度 */
  wobbleAmp: number
  /** 水平摇摆速度 */
  wobbleSpeed: number
  /** 缩放衰减速度（变小速度） */
  scaleDecay: number
  /** 聚集后漂浮速度 */
  floatSpeed: number
}

interface LanternConfig {
  baseWidth: number
  baseHeight: number
  x: number
  y: number
  vx: number
  vy: number
  scale: number
  scaleDecay: number
  wobble: number
  wobbleSpeed: number
  wobbleAmp: number
  flickerBase: number
  flickerSpeed: number
  flickerState: number
  candleIntensity: number
  text: string
  state: LanternState
  // 实例特定的速度配置
  speedConfig: SpeedConfig
}

interface Props {
  /** 是否开始放飞 */
  release?: boolean
  /** 是否停止生成 */
  stopSpawning?: boolean
  /** 初始数量 */
  initialCount?: number
  /** 最大数量 */
  maxCount?: number
  /** 生成概率 */
  spawnRate?: number
  /** 祝福语列表 */
  blessings?: string[]
  /** 是否显示 */
  visible?: boolean
  /** 层级 */
  zIndex?: number

  /** 上升速度倍率（1为默认，2为两倍速，0.5为半速） */
  speedMultiplier?: number
  /** 最小上升速度（像素/帧） */
  minRiseSpeed?: number
  /** 最大上升速度（像素/帧） */
  maxRiseSpeed?: number
  /** 水平摇摆幅度（像素） */
  wobbleAmplitude?: number
  /** 变小速度倍率 */
  shrinkMultiplier?: number
  /** 聚集后漂浮活跃度（0-1） */
  floatActivity?: number
}

const props = withDefaults(defineProps<Props>(), {
  release: false,
  stopSpawning: false,
  initialCount: 80,
  maxCount: 200,
  spawnRate: 0.06,
  blessings: () => [
    '学业有成',
    '事业腾飞',
    '身体健康',
    '岁岁平安',
    '幸福美满',
    '前程似锦',
    '万事如意',
    '心想事成',
    '吉星高照',
    '五福临门',
    '平安喜乐',
    '财源广进',

    '日进斗金',
    '金玉满堂',
    '招财进宝',
    '富贵荣华',
    '腰缠万贯',
    '生意兴隆',
    '财源滚滚',
    '八方来财',

    '步步高升',
    '大展宏图',
    '鹏程万里',
    '一帆风顺',
    '马到成功',
    '功成名就',
    '蒸蒸日上',
    '如日中天',

    '阖家欢乐',
    '天伦之乐',
    '吉祥如意',
    '喜乐安宁',
    '岁岁无忧',
    '万事顺意',
    '阖家幸福',
    '春风得意',

    '福星高照',
    '鸿运当头',
    '紫气东来',
    '吉祥高照',
    '万事亨通',
    '大吉大利',
    '十全十美',
    '百福具臻',
  ],
  visible: true,
  zIndex: 5,

  // 速度默认值
  speedMultiplier: 1,
  minRiseSpeed: 0.3,
  maxRiseSpeed: 0.8,
  wobbleAmplitude: 0.4,
  shrinkMultiplier: 1,
  floatActivity: 0.5,
})

const emit = defineEmits<{
  (e: 'lanternGathered', count: number): void
  (e: 'allGathered'): void
  (e: 'update:release', value: boolean): void
}>()

const canvasRef = ref<HTMLCanvasElement | null>(null)
const ctx = ref<CanvasRenderingContext2D | null>(null)
const lanterns = ref<Lantern[]>([])
const animationId = ref<number>(0)
const width = ref(0)
const height = ref(0)
const isActive = ref(false)

// 计算实际速度配置
const getSpeedConfig = (): SpeedConfig => ({
  riseSpeed: props.minRiseSpeed * props.speedMultiplier,
  riseSpeedVar: (props.maxRiseSpeed - props.minRiseSpeed) * props.speedMultiplier,
  wobbleAmp: props.wobbleAmplitude,
  wobbleSpeed: 0.015 + props.floatActivity * 0.02,
  scaleDecay: (0.0002 + Math.random() * 0.0003) * props.shrinkMultiplier,
  floatSpeed: 0.03 + props.floatActivity * 0.05,
})

class Lantern {
  config: LanternConfig

  constructor(isReleased: boolean, canvasWidth: number, canvasHeight: number) {
    const baseWidth = Math.random() * 30 + 20
    const baseHeight = baseWidth * 1.8
    const speedConfig = getSpeedConfig()

    this.config = {
      baseWidth,
      baseHeight,
      x: Math.random() * canvasWidth,
      y: isReleased ? canvasHeight + 50 : canvasHeight + baseHeight * 0.4 + Math.random() * 20,
      vx: (Math.random() - 0.5) * 0.25,
      vy: 0,
      scale: isReleased ? 1 : Math.random() * 0.2 + 0.85,
      scaleDecay: speedConfig.scaleDecay,
      wobble: Math.random() * Math.PI * 2,
      wobbleSpeed: speedConfig.wobbleSpeed,
      wobbleAmp: speedConfig.wobbleAmp,
      flickerBase: Math.random() > 0.5 ? 240 : 210,
      flickerSpeed: Math.random() * 0.08 + 0.04,
      flickerState: Math.random() * Math.PI * 2,
      candleIntensity: Math.random() * 0.3 + 0.7,
      text:
        baseWidth > 28 ? props.blessings[Math.floor(Math.random() * props.blessings.length)] : '',
      state: isReleased ? 'rising' : 'grounded',
      speedConfig,
    }

    if (isReleased) {
      this.launch(canvasHeight)
    }
  }

  launch(canvasHeight: number) {
    this.config.state = 'rising'
    const speedCfg = this.config.speedConfig
    // 根据配置计算上升速度
    this.config.vy = -(speedCfg.riseSpeed + Math.random() * speedCfg.riseSpeedVar)
    this.config.y = canvasHeight + 50
    this.config.scale = 1
  }

  update(canvasWidth: number, canvasHeight: number) {
    const cfg = this.config
    cfg.wobble += cfg.wobbleSpeed
    cfg.flickerState += cfg.flickerSpeed

    if (cfg.state === 'grounded') {
      // 地面摇摆
      cfg.x += cfg.vx + Math.sin(cfg.wobble) * 0.08
      cfg.y = canvasHeight + cfg.baseHeight * 0.4 + Math.sin(cfg.wobble * 0.6) * 4
    } else if (cfg.state === 'rising') {
      // 上升阶段：应用速度配置
      cfg.x += (cfg.vx + Math.sin(cfg.wobble) * cfg.wobbleAmp) * cfg.scale
      cfg.y += cfg.vy * cfg.scale
      cfg.scale -= cfg.scaleDecay

      // 到达天空顶部或缩放过小则聚集
      const targetHeight = canvasHeight * (0.08 + Math.random() * 0.15)
      if ((cfg.y < targetHeight && cfg.scale < 0.22) || cfg.scale <= 0.05) {
        cfg.state = 'gathered'
      }
    } else if (cfg.state === 'gathered') {
      // 聚集后缓慢漂浮
      if (cfg.scale > 0.05) {
        cfg.scale -= 0.0008 * props.shrinkMultiplier
      } else {
        cfg.scale = 0.05
      }
      const floatSpeed = cfg.speedConfig.floatSpeed
      cfg.x += Math.sin(cfg.wobble) * floatSpeed * cfg.scale
      cfg.y += Math.cos(cfg.wobble * 0.7) * floatSpeed * cfg.scale
    }
  }

  draw(context: CanvasRenderingContext2D) {
    const cfg = this.config
    if (cfg.scale <= 0.01 || cfg.y < -100) return

    const fVal = cfg.flickerBase + Math.sin(cfg.flickerState) * 35
    const cRgb = `${255},${Math.min(255, fVal + 8)},${Math.max(0, fVal - 90)}`

    context.save()
    context.translate(cfg.x, cfg.y)
    context.scale(cfg.scale, cfg.scale)

    // 灯身渐变
    const grad = context.createRadialGradient(
      0,
      cfg.baseHeight * 0.35,
      0,
      0,
      0,
      cfg.baseHeight * 0.85,
    )
    grad.addColorStop(0, `rgba(255,255,220,${0.9 * cfg.candleIntensity})`)
    grad.addColorStop(0.5, `rgba(${cRgb},${0.8 * cfg.candleIntensity})`)
    grad.addColorStop(1, `rgba(${cRgb.split(',')[0]},${cRgb.split(',')[1]},50,0)`)

    context.fillStyle = grad
    context.beginPath()
    context.moveTo(-cfg.baseWidth / 2, -cfg.baseHeight * 0.45)
    context.lineTo(cfg.baseWidth / 2, -cfg.baseHeight * 0.45)
    context.lineTo(cfg.baseWidth * 0.4, cfg.baseHeight / 2)
    context.lineTo(-cfg.baseWidth * 0.4, cfg.baseHeight / 2)
    context.closePath()
    context.fill()

    // 文字
    if (cfg.text && cfg.scale > 0.3 && cfg.state === 'rising') {
      const fs = Math.min(cfg.baseWidth * 0.35, (cfg.baseHeight * 0.7) / cfg.text.length)
      context.font = `bold ${fs}px "KaiTi","STKaiti","Microsoft YaHei",sans-serif`
      context.fillStyle = 'rgba(30,15,0,0.8)'
      context.textAlign = 'center'
      context.textBaseline = 'middle'

      for (let i = 0; i < cfg.text.length; i++) {
        context.fillText(cfg.text[i], 0, -((cfg.text.length - 1) * fs * 1.05) / 2 + i * fs * 1.05)
      }
    }

    // 底部光晕
    context.beginPath()
    context.ellipse(
      0,
      cfg.baseHeight / 2,
      cfg.baseWidth * 0.4,
      cfg.baseWidth * 0.1,
      0,
      0,
      Math.PI * 2,
    )
    context.fillStyle = `rgba(${cRgb},0.9)`
    context.shadowBlur = 25
    context.shadowColor = 'rgba(255,170,40,0.8)'
    context.fill()

    context.restore()
  }

  isGathered(): boolean {
    return this.config.state === 'gathered'
  }

  // 动态更新速度配置
  updateSpeedConfig() {
    this.config.speedConfig = getSpeedConfig()
    // 如果正在上升，更新vy保持连贯
    if (this.config.state === 'rising' && this.config.vy < 0) {
      const speedCfg = this.config.speedConfig
      // 保持方向，更新速度大小
      const currentSpeedRatio =
        Math.abs(this.config.vy) / (props.minRiseSpeed * props.speedMultiplier)
      const newSpeed = speedCfg.riseSpeed + Math.random() * speedCfg.riseSpeedVar
      this.config.vy = -newSpeed * Math.max(0.5, Math.min(1.5, currentSpeedRatio))
    }
  }
}

const initCanvas = () => {
  const canvas = canvasRef.value
  if (!canvas) return

  ctx.value = canvas.getContext('2d')
  resizeCanvas()

  for (let i = 0; i < props.initialCount; i++) {
    lanterns.value.push(new Lantern(false, width.value, height.value))
  }

  isActive.value = true
  animate()
}

const resizeCanvas = () => {
  const canvas = canvasRef.value
  if (!canvas) return

  const dpr = window.devicePixelRatio || 1
  const rect = canvas.getBoundingClientRect()

  width.value = rect.width
  height.value = rect.height

  canvas.width = rect.width * dpr
  canvas.height = rect.height * dpr

  ctx.value?.scale(dpr, dpr)
}

const animate = () => {
  if (!isActive.value) return

  const context = ctx.value
  if (!context) return

  context.clearRect(0, 0, width.value, height.value)

  if (props.release && !props.stopSpawning && Math.random() < props.spawnRate) {
    if (lanterns.value.length < props.maxCount) {
      lanterns.value.push(new Lantern(true, width.value, height.value))
    }
  }

  lanterns.value = lanterns.value.filter((lantern) => {
    lantern.update(width.value, height.value)
    lantern.draw(context)
    return lantern.config.scale > 0.01 && lantern.config.y > -150
  })

  const gatheredCount = lanterns.value.filter((l) => l.isGathered()).length
  emit('lanternGathered', gatheredCount)

  if (props.stopSpawning && gatheredCount === lanterns.value.length && lanterns.value.length > 0) {
    emit('allGathered')
  }

  animationId.value = requestAnimationFrame(animate)
}

const handleResize = () => {
  nextTick(() => {
    resizeCanvas()
  })
}

// 监听 release 变化
watch(
  () => props.release,
  (newVal) => {
    if (newVal) {
      lanterns.value.forEach((lantern) => {
        if (lantern.config.state === 'grounded') {
          lantern.launch(height.value)
        }
      })
    }
  },
)

// 监听速度参数变化，实时更新现有孔明灯
watch(
  [
    () => props.speedMultiplier,
    () => props.minRiseSpeed,
    () => props.maxRiseSpeed,
    () => props.wobbleAmplitude,
    () => props.shrinkMultiplier,
    () => props.floatActivity,
  ],
  () => {
    lanterns.value.forEach((l) => l.updateSpeedConfig())
  },
  { immediate: false },
)

// 监听 visible
watch(
  () => props.visible,
  (newVal) => {
    if (!newVal) {
      isActive.value = false
      cancelAnimationFrame(animationId.value)
    } else if (!isActive.value) {
      isActive.value = true
      animate()
    }
  },
)

onMounted(() => {
  nextTick(() => {
    initCanvas()
    window.addEventListener('resize', handleResize)
  })
})

onUnmounted(() => {
  isActive.value = false
  cancelAnimationFrame(animationId.value)
  window.removeEventListener('resize', handleResize)
})

defineExpose({
  addLantern: (customSpeed?: Partial<SpeedConfig>) => {
    const lantern = new Lantern(props.release, width.value, height.value)
    if (customSpeed) {
      Object.assign(lantern.config.speedConfig, customSpeed)
    }
    lanterns.value.push(lantern)
  },
  clear: () => {
    lanterns.value = []
  },
  getCount: () => lanterns.value.length,
  getGatheredCount: () => lanterns.value.filter((l) => l.isGathered()).length,
  releaseAll: () => {
    lanterns.value.forEach((l) => {
      if (l.config.state === 'grounded') l.launch(height.value)
    })
  },
  // 批量更新速度
  updateAllSpeeds: (multiplier: number) => {
    lanterns.value.forEach((l) => {
      l.config.speedConfig.riseSpeed = props.minRiseSpeed * multiplier
      l.config.speedConfig.riseSpeedVar = (props.maxRiseSpeed - props.minRiseSpeed) * multiplier
      if (l.config.state === 'rising') {
        l.config.vy = -(
          l.config.speedConfig.riseSpeed +
          Math.random() * l.config.speedConfig.riseSpeedVar
        )
      }
    })
  },
})
</script>

<style scoped>
.lantern-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: v-bind(zIndex);
}

.lantern-canvas {
  width: 100%;
  height: 100%;
  display: block;
}
</style>

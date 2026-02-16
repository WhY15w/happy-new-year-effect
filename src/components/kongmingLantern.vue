<template>
  <div
    class="pointer-events-none fixed top-0 left-0 h-full w-full"
    :style="{ zIndex }"
  >
    <canvas
      ref="canvasRef"
      class="lantern-canvas block h-full w-full"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'

type LanternState = 'grounded' | 'rising' | 'gathered'

interface SpeedConfig {
  riseSpeed: number
  riseSpeedVar: number
  wobbleAmp: number
  wobbleSpeed: number
  scaleDecay: number
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
  wobble: number
  flickerBase: number
  flickerSpeed: number
  flickerState: number
  candleIntensity: number
  text: string
  state: LanternState
  targetHeight: number
  speedConfig: SpeedConfig
}

interface Props {
  release?: boolean
  stopSpawning?: boolean
  initialCount?: number
  maxCount?: number
  spawnRate?: number
  blessings?: string[]
  visible?: boolean
  zIndex?: number
  speedMultiplier?: number
  minRiseSpeed?: number
  maxRiseSpeed?: number
  wobbleAmplitude?: number
  shrinkMultiplier?: number
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
const animationId = ref(0)
const width = ref(0)
const height = ref(0)
const isActive = ref(false)
let lastGatheredCount = -1

const getSpeedConfig = (): SpeedConfig => ({
  riseSpeed: props.minRiseSpeed * props.speedMultiplier,
  riseSpeedVar:
    (props.maxRiseSpeed - props.minRiseSpeed) * props.speedMultiplier,
  wobbleAmp: props.wobbleAmplitude,
  wobbleSpeed: 0.015 + props.floatActivity * 0.02,
  scaleDecay: (0.0002 + Math.random() * 0.0003) * props.shrinkMultiplier,
  floatSpeed: 0.03 + props.floatActivity * 0.05,
})

const randomBlessing = (): string =>
  props.blessings[Math.floor(Math.random() * props.blessings.length)]

class Lantern {
  config: LanternConfig

  constructor(isReleased: boolean, canvasWidth: number, canvasHeight: number) {
    const baseWidth = Math.random() * 30 + 20
    const speedConfig = getSpeedConfig()

    this.config = {
      baseWidth,
      baseHeight: baseWidth * 1.8,
      x: Math.random() * canvasWidth,
      y: isReleased
        ? canvasHeight + 50
        : canvasHeight + baseWidth * 1.8 * 0.4 + Math.random() * 20,
      vx: (Math.random() - 0.5) * 0.25,
      vy: 0,
      scale: isReleased ? 1 : Math.random() * 0.2 + 0.85,
      wobble: Math.random() * Math.PI * 2,
      flickerBase: Math.random() > 0.5 ? 240 : 210,
      flickerSpeed: Math.random() * 0.08 + 0.04,
      flickerState: Math.random() * Math.PI * 2,
      candleIntensity: Math.random() * 0.3 + 0.7,
      text: baseWidth > 22 ? randomBlessing() : '',
      state: isReleased ? 'rising' : 'grounded',
      targetHeight: canvasHeight * (0.08 + Math.random() * 0.15),
      speedConfig,
    }

    if (isReleased) this.launch(canvasHeight)
  }

  launch(canvasHeight: number) {
    const { speedConfig } = this.config
    this.config.state = 'rising'
    this.config.vy = -(
      speedConfig.riseSpeed +
      Math.random() * speedConfig.riseSpeedVar
    )
    this.config.y = canvasHeight + 50
    this.config.scale = 1
    this.config.targetHeight = canvasHeight * (0.08 + Math.random() * 0.15)
  }

  update(canvasWidth: number, canvasHeight: number) {
    const cfg = this.config
    const { wobbleAmp, floatSpeed } = cfg.speedConfig

    cfg.wobble += cfg.speedConfig.wobbleSpeed
    cfg.flickerState += cfg.flickerSpeed

    switch (cfg.state) {
      case 'grounded':
        cfg.x += cfg.vx + Math.sin(cfg.wobble) * 0.08
        cfg.y =
          canvasHeight + cfg.baseHeight * 0.4 + Math.sin(cfg.wobble * 0.6) * 4
        break

      case 'rising':
        cfg.x += (cfg.vx + Math.sin(cfg.wobble) * wobbleAmp) * cfg.scale
        cfg.y += cfg.vy * cfg.scale
        cfg.scale -= cfg.speedConfig.scaleDecay
        if (
          (cfg.y < cfg.targetHeight && cfg.scale < 0.22) ||
          cfg.scale <= 0.05
        ) {
          cfg.state = 'gathered'
        }
        break

      case 'gathered':
        cfg.scale =
          cfg.scale > 0.05 ? cfg.scale - 0.0008 * props.shrinkMultiplier : 0.05
        cfg.x += Math.sin(cfg.wobble) * floatSpeed * cfg.scale
        cfg.y += Math.cos(cfg.wobble * 0.7) * floatSpeed * cfg.scale
        break
    }
  }

  draw(context: CanvasRenderingContext2D) {
    const cfg = this.config
    if (cfg.scale <= 0.01 || cfg.y < -100) return

    const fVal = cfg.flickerBase + Math.sin(cfg.flickerState) * 35
    const r = 255
    const g = Math.min(255, fVal + 8)
    const b = Math.max(0, fVal - 90)
    const cRgb = `${r},${g},${b}`

    context.save()
    context.translate(cfg.x, cfg.y)
    context.scale(cfg.scale, cfg.scale)

    // 灯身
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
    grad.addColorStop(1, `rgba(${r},${g},50,0)`)

    context.fillStyle = grad
    context.beginPath()
    context.moveTo(-cfg.baseWidth / 2, -cfg.baseHeight * 0.45)
    context.lineTo(cfg.baseWidth / 2, -cfg.baseHeight * 0.45)
    context.lineTo(cfg.baseWidth * 0.4, cfg.baseHeight / 2)
    context.lineTo(-cfg.baseWidth * 0.4, cfg.baseHeight / 2)
    context.closePath()
    context.fill()

    // 祝福文字（仅上升阶段且缩放足够大时显示）
    if (cfg.text && cfg.scale > 0.15 && cfg.state === 'rising') {
      const fs = Math.min(
        cfg.baseWidth * 0.35,
        (cfg.baseHeight * 0.7) / cfg.text.length,
      )
      context.font = `bold ${fs}px "KaiTi","STKaiti","Microsoft YaHei",sans-serif`
      context.fillStyle = 'rgba(30,15,0,0.8)'
      context.textAlign = 'center'
      context.textBaseline = 'middle'
      const lineHeight = fs * 1.05
      const offsetY = -((cfg.text.length - 1) * lineHeight) / 2
      for (let i = 0; i < cfg.text.length; i++) {
        context.fillText(cfg.text[i], 0, offsetY + i * lineHeight)
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

  updateSpeedConfig() {
    const oldVy = this.config.vy
    this.config.speedConfig = getSpeedConfig()
    if (this.config.state === 'rising' && oldVy < 0) {
      const { riseSpeed, riseSpeedVar } = this.config.speedConfig
      this.config.vy = -(riseSpeed + Math.random() * riseSpeedVar)
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

  // 放飞状态下按概率生成新灯笼
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

  // 仅在聚集数量变化时触发事件
  const gatheredCount = lanterns.value.filter((l) => l.isGathered()).length
  if (gatheredCount !== lastGatheredCount) {
    lastGatheredCount = gatheredCount
    emit('lanternGathered', gatheredCount)
  }

  if (
    props.stopSpawning &&
    gatheredCount === lanterns.value.length &&
    lanterns.value.length > 0
  ) {
    emit('allGathered')
  }

  animationId.value = requestAnimationFrame(animate)
}

let resizeTimer: ReturnType<typeof setTimeout> | null = null
const handleResize = () => {
  if (resizeTimer) clearTimeout(resizeTimer)
  resizeTimer = setTimeout(() => {
    nextTick(resizeCanvas)
  }, 100)
}

watch(
  () => props.release,
  (released) => {
    if (released) {
      lanterns.value.forEach((l) => {
        if (l.config.state === 'grounded') l.launch(height.value)
      })
    }
  },
)

watch(
  [
    () => props.speedMultiplier,
    () => props.minRiseSpeed,
    () => props.maxRiseSpeed,
    () => props.wobbleAmplitude,
    () => props.shrinkMultiplier,
    () => props.floatActivity,
  ],
  () => lanterns.value.forEach((l) => l.updateSpeedConfig()),
)

watch(
  () => props.visible,
  (visible) => {
    if (!visible) {
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
  if (resizeTimer) clearTimeout(resizeTimer)
})

defineExpose({
  addLantern: (customSpeed?: Partial<SpeedConfig>) => {
    const lantern = new Lantern(props.release, width.value, height.value)
    if (customSpeed) Object.assign(lantern.config.speedConfig, customSpeed)
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
  updateAllSpeeds: (multiplier: number) => {
    lanterns.value.forEach((l) => {
      l.config.speedConfig.riseSpeed = props.minRiseSpeed * multiplier
      l.config.speedConfig.riseSpeedVar =
        (props.maxRiseSpeed - props.minRiseSpeed) * multiplier
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

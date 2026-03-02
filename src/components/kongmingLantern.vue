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
  wobblePhase: number
  flickerBase: number
  flickerSpeed: number
  flickerState: number
  candleIntensity: number
  text: string
  state: LanternState
  targetHeight: number
  speedConfig: SpeedConfig
  hue: number
  swayOffset: number
  flameHeight: number
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
    const baseWidth = Math.random() * 25 + 22
    const speedConfig = getSpeedConfig()
    const hueVariation = Math.random() * 30 - 15 // -15 to 15 hue variation

    this.config = {
      baseWidth,
      baseHeight: baseWidth * 1.6,
      x: Math.random() * canvasWidth,
      y: isReleased
        ? canvasHeight + 50
        : canvasHeight + baseWidth * 1.8 * 0.4 + Math.random() * 20,
      vx: (Math.random() - 0.5) * 0.2,
      vy: 0,
      scale: isReleased ? 1 : Math.random() * 0.15 + 0.88,
      wobble: Math.random() * Math.PI * 2,
      wobblePhase: Math.random() * Math.PI * 2,
      flickerBase: Math.random() > 0.5 ? 245 : 220,
      flickerSpeed: Math.random() * 0.06 + 0.03,
      flickerState: Math.random() * Math.PI * 2,
      candleIntensity: Math.random() * 0.25 + 0.75,
      text: baseWidth > 24 ? randomBlessing() : '',
      state: isReleased ? 'rising' : 'grounded',
      targetHeight: canvasHeight * (0.08 + Math.random() * 0.15),
      speedConfig,
      hue: 45 + hueVariation, // Orange-yellow base with variation
      swayOffset: Math.random() * 100,
      flameHeight: Math.random() * 0.3 + 0.85,
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
    cfg.wobblePhase += cfg.speedConfig.wobbleSpeed * 0.7
    cfg.flickerState += cfg.flickerSpeed

    switch (cfg.state) {
      case 'grounded':
        cfg.x += cfg.vx + Math.sin(cfg.wobble) * 0.06
        cfg.y =
          canvasHeight + cfg.baseHeight * 0.4 + Math.sin(cfg.wobble * 0.5) * 3
        break

      case 'rising':
        // More natural swaying motion combining multiple sine waves
        const sway = Math.sin(cfg.wobble) * wobbleAmp + 
                     Math.sin(cfg.wobble * 0.5 + cfg.swayOffset) * (wobbleAmp * 0.3)
        cfg.x += (cfg.vx + sway) * cfg.scale
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
          cfg.scale > 0.05 ? cfg.scale - 0.0006 * props.shrinkMultiplier : 0.05
        cfg.x += Math.sin(cfg.wobble) * floatSpeed * cfg.scale * 0.8
        cfg.y += Math.cos(cfg.wobble * 0.6) * floatSpeed * cfg.scale * 0.6
        break
    }
  }

  draw(context: CanvasRenderingContext2D) {
    const cfg = this.config
    if (cfg.scale <= 0.01 || cfg.y < -150) return

    const fVal = cfg.flickerBase + Math.sin(cfg.flickerState) * 30
    const r = 255
    const g = Math.min(255, fVal + 5)
    const b = Math.max(0, fVal - 80)
    const cRgb = `${r},${g},${b}`
    
    const w = cfg.baseWidth
    const h = cfg.baseHeight
    const halfW = w / 2
    const halfH = h / 2

    context.save()
    context.translate(cfg.x, cfg.y)
    context.scale(cfg.scale, cfg.scale)
    
    // Apply slight rotation based on sway for more natural movement
    const rotation = Math.sin(cfg.wobblePhase) * 0.05
    context.rotate(rotation)

    // ===== 绘制孔明灯主体（梨形/椭圆造型） =====
    
    // 1. 外部纸质发光层（柔光效果）
    const outerGlow = context.createRadialGradient(0, 0, 0, 0, 0, halfH * 1.2)
    outerGlow.addColorStop(0, `rgba(${cRgb}, 0.15)`)
    outerGlow.addColorStop(0.6, `rgba(${cRgb}, 0.05)`)
    outerGlow.addColorStop(1, 'rgba(255, 100, 0, 0)')
    
    context.fillStyle = outerGlow
    context.beginPath()
    context.ellipse(0, 0, halfW * 1.3, halfH * 1.25, 0, 0, Math.PI * 2)
    context.fill()

    // 2. 主体形状 - 梨形孔明灯
    context.beginPath()
    // 顶部圆弧
    context.arc(0, -halfH * 0.3, halfW * 0.85, Math.PI * 0.15, Math.PI * 0.85, true)
    // 两侧曲线
    context.bezierCurveTo(
      halfW * 0.9, halfH * 0.2,
      halfW * 0.7, halfH * 0.7,
      halfW * 0.5, halfH * 0.9
    )
    // 底部圆弧
    context.arc(0, halfH * 0.9, halfW * 0.5, 0, Math.PI, true)
    // 左侧曲线
    context.bezierCurveTo(
      -halfW * 0.7, halfH * 0.7,
      -halfW * 0.9, halfH * 0.2,
      -halfW * 0.85, -halfH * 0.47
    )
    context.closePath()

    // 3. 主体渐变填充（模拟纸质透光效果）
    const bodyGrad = context.createRadialGradient(
      0, -halfH * 0.1,
      0,
      0, halfH * 0.3,
      halfH
    )
    const alpha = 0.75 * cfg.candleIntensity
    bodyGrad.addColorStop(0, `rgba(255, 248, 220, ${alpha})`)
    bodyGrad.addColorStop(0.3, `rgba(${cRgb}, ${alpha * 0.9})`)
    bodyGrad.addColorStop(0.7, `rgba(${r * 0.9},${g * 0.7},${b * 0.3}, ${alpha * 0.7})`)
    bodyGrad.addColorStop(1, `rgba(${r * 0.7},${g * 0.4},30, ${alpha * 0.5})`)
    
    context.fillStyle = bodyGrad
    context.fill()

    // 4. 骨架线条（模拟竹骨结构）
    context.strokeStyle = `rgba(80, 40, 20, 0.25)`
    context.lineWidth = 0.8
    
    // 垂直骨架
    context.beginPath()
    context.moveTo(0, -halfH * 0.75)
    context.quadraticCurveTo(0, 0, 0, halfH * 0.85)
    context.stroke()
    
    // 横向骨架（弧度）
    for (let i = 0; i < 3; i++) {
      const yPos = -halfH * 0.3 + i * halfH * 0.4
      const wAtY = halfW * (0.6 + 0.25 * Math.sin((i + 1) * Math.PI / 4))
      context.beginPath()
      context.moveTo(-wAtY, yPos)
      context.quadraticCurveTo(0, yPos + 5, wAtY, yPos)
      context.stroke()
    }

    // 5. 内部光晕（烛光效果）
    const flameGrad = context.createRadialGradient(
      0, halfH * 0.5,
      0,
      0, halfH * 0.5,
      halfW * 0.6
    )
    const flameAlpha = 0.6 + Math.sin(cfg.flickerState * 1.5) * 0.15
    flameGrad.addColorStop(0, `rgba(255, 255, 200, ${flameAlpha})`)
    flameGrad.addColorStop(0.4, `rgba(255, 200, 100, ${flameAlpha * 0.6})`)
    flameGrad.addColorStop(1, 'rgba(255, 150, 50, 0)')
    
    context.fillStyle = flameGrad
    context.beginPath()
    context.ellipse(0, halfH * 0.5, halfW * 0.5, halfW * 0.35, 0, 0, Math.PI * 2)
    context.fill()

    // 6. 祝福文字（竖排，仅上升阶段且缩放足够大时显示）
    if (cfg.text && cfg.scale > 0.18 && cfg.state === 'rising') {
      const fs = Math.min(w * 0.28, (h * 0.6) / cfg.text.length)
      context.font = `bold ${fs}px "KaiTi","STKaiti","Microsoft YaHei",serif`
      context.fillStyle = 'rgba(40, 20, 5, 0.85)'
      context.textAlign = 'center'
      context.textBaseline = 'middle'
      context.shadowColor = 'rgba(255, 200, 100, 0.3)'
      context.shadowBlur = 2
      
      const lineHeight = fs * 1.1
      const offsetY = -((cfg.text.length - 1) * lineHeight) / 2
      
      for (let i = 0; i < cfg.text.length; i++) {
        context.fillText(cfg.text[i], 0, offsetY + i * lineHeight)
      }
      
      context.shadowBlur = 0
    }

    // 7. 顶部装饰（金属环）
    context.fillStyle = 'rgba(139, 90, 43, 0.8)'
    context.beginPath()
    context.ellipse(0, -halfH * 0.72, halfW * 0.12, halfW * 0.06, 0, 0, Math.PI * 2)
    context.fill()

    // 8. 底部开口和飘带
    // 底部开口边缘
    context.strokeStyle = `rgba(100, 60, 30, 0.5)`
    context.lineWidth = 1.5
    context.beginPath()
    context.ellipse(0, halfH * 0.9, halfW * 0.5, halfW * 0.12, 0, 0, Math.PI * 2)
    context.stroke()

    // 飘带（随风摆动）
    const ribbonSway = Math.sin(cfg.wobble * 1.2 + cfg.swayOffset) * 3
    context.strokeStyle = `rgba(180, 80, 40, 0.7)`
    context.lineWidth = 1.2
    context.lineCap = 'round'
    
    // 左飘带
    context.beginPath()
    context.moveTo(-halfW * 0.3, halfH * 0.95)
    context.quadraticCurveTo(
      -halfW * 0.3 + ribbonSway, halfH * 1.1,
      -halfW * 0.3 - ribbonSway * 0.5, halfH * 1.25
    )
    context.stroke()
    
    // 右飘带
    context.beginPath()
    context.moveTo(halfW * 0.3, halfH * 0.95)
    context.quadraticCurveTo(
      halfW * 0.3 + ribbonSway, halfH * 1.1,
      halfW * 0.3 + ribbonSway * 0.5, halfH * 1.25
    )
    context.stroke()
    
    // 飘带末端小球
    context.fillStyle = 'rgba(200, 100, 50, 0.8)'
    context.beginPath()
    context.arc(-halfW * 0.3 - ribbonSway * 0.5, halfH * 1.25, 1.5, 0, Math.PI * 2)
    context.arc(halfW * 0.3 + ribbonSway * 0.5, halfH * 1.25, 1.5, 0, Math.PI * 2)
    context.fill()

    // 9. 底部光晕（照亮效果）
    const bottomGlow = context.createRadialGradient(
      0, halfH * 0.9,
      0,
      0, halfH * 0.9,
      halfW * 0.8
    )
    bottomGlow.addColorStop(0, `rgba(${cRgb}, 0.6)`)
    bottomGlow.addColorStop(0.5, `rgba(255, 150, 50, 0.2)`)
    bottomGlow.addColorStop(1, 'rgba(255, 100, 0, 0)')
    
    context.fillStyle = bottomGlow
    context.beginPath()
    context.ellipse(0, halfH * 0.9, halfW * 0.8, halfW * 0.25, 0, 0, Math.PI * 2)
    context.fill()

    // 10. 顶部高光（增强立体感）
    const highlightGrad = context.createRadialGradient(
      -halfW * 0.3, -halfH * 0.5,
      0,
      -halfW * 0.3, -halfH * 0.5,
      halfW * 0.4
    )
    highlightGrad.addColorStop(0, 'rgba(255, 255, 255, 0.25)')
    highlightGrad.addColorStop(0.5, 'rgba(255, 255, 255, 0.08)')
    highlightGrad.addColorStop(1, 'rgba(255, 255, 255, 0)')
    
    context.fillStyle = highlightGrad
    context.beginPath()
    context.ellipse(-halfW * 0.3, -halfH * 0.5, halfW * 0.25, halfH * 0.2, -0.3, 0, Math.PI * 2)
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
    return lantern.config.scale > 0.01 && lantern.config.y > -200
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

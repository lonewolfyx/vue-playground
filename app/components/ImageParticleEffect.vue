<template>
    <div ref="containerRef" class="relative w-full h-full overflow-hidden">
        <canvas
            ref="canvasRef"
            :style="{ display: 'block' }"
            class="block w-full h-full"
        />
    </div>
</template>

<script lang="ts" setup>
import { onMounted, onUnmounted, ref, watch } from 'vue'

interface Particle {
    ox: number
    oy: number
    x: number
    y: number
    vx: number
    vy: number
    size: number
    alpha: number
    phase: number
    springJitter: number
    appear: number
    fading: boolean
}

const props = withDefaults(defineProps<{
    src: string
    sampleStep?: number // 默认 3
    threshold?: number // 默认 50
    renderScale?: number // 默认 1
    dotSize?: number // 默认 1.15
    mouseForce?: number // 默认 90
    mouseRadius?: number // 默认 110
    spring?: number // 默认 0.035
    damping?: number // 默认 0.86
    color?: string // 默认根据主题
    invert?: boolean // 默认 false
    adaptToTheme?: boolean // 默认 true
    align?: 'center' | 'bottom' // 默认 'center'
    denseParticles?: boolean // 默认 false
    typingImpulse?: number // 外部传入的打字冲击值
}>(), {
    sampleStep: 3,
    threshold: 50,
    renderScale: 1,
    dotSize: 1.15,
    mouseForce: 90,
    mouseRadius: 110,
    spring: 0.035,
    damping: 0.86,
    color: 'rgba(255, 255, 255, 0.92)',
    invert: false,
    adaptToTheme: true,
    align: 'center',
    denseParticles: false,
})

const emit = defineEmits<{
    (e: 'ready'): void
}>()

const canvasRef = ref<HTMLCanvasElement | null>(null)
const containerRef = ref<HTMLDivElement | null>(null)

let ctx: CanvasRenderingContext2D | null = null
let particles: Particle[] = []
let animationFrame: number | null = null
let imageRef: HTMLImageElement | null = null
const mouse = { x: -9999, y: -9999, active: false }
let time = 0
const impulse = ref(props.typingImpulse || 0)

const currentColor = ref(props.color)

// 响应深色/浅色主题（如果你使用 Tailwind dark:）
const isDark = () => document.documentElement.classList.contains('dark')

watch(() => props.src, (newSrc) => {
    if (newSrc)
        loadImage(newSrc, true)
})

watch(() => props.typingImpulse, (val) => {
    if (val !== undefined)
        impulse.value = val
})

// 主动画循环
function animate() {
    if (!ctx || !canvasRef.value)
        return

    const canvas = canvasRef.value
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    ctx.fillStyle = currentColor.value

    const force = props.mouseForce
    const radius = props.mouseRadius * (window.devicePixelRatio || 1)
    const radiusSq = radius * radius
    const springForce = props.spring
    const damp = props.damping
    const typingImp = impulse.value

    let writeIndex = 0

    for (let i = 0; i < particles.length; i++) {
        const p = particles[i]!

        // 弹簧力
        const dx = p.ox - p.x
        const dy = p.oy - p.y
        p.vx += dx * springForce * p.springJitter
        p.vy += dy * springForce * p.springJitter

        // 鼠标交互
        if (mouse.active) {
            const mx = mouse.x * (window.devicePixelRatio || 1)
            const my = mouse.y * (window.devicePixelRatio || 1)
            const pdx = p.x - mx
            const pdy = p.y - my
            const distSq = pdx * pdx + pdy * pdy

            if (distSq < radiusSq && distSq > 1) {
                const dist = Math.sqrt(distSq)
                const f = (1 - dist / radius) * force * 0.04
                p.vx += (pdx / dist) * f
                p.vy += (pdy / dist) * f
            }
        }

        // 轻微呼吸 + typing 冲击
        const breath = Math.sin(time * 0.8 + p.phase) * 0.08
        p.vx += breath * 0.05 * (1 + typingImp * 10)
        p.vy += Math.cos(time * 0.9 + p.phase) * 0.04 * (1 + typingImp * 10)

        if (typingImp > 0.001) {
            const cx = canvas.width / 2
            const cy = canvas.height * 0.48
            const tx = p.x - cx
            const ty = p.y - cy
            const td = Math.sqrt(tx * tx + ty * ty) + 0.5
            const push = typingImp * 22 / td
            p.vx += (tx / td) * push * 0.018
            p.vy += (ty / td) * push * 0.018

            // 随机扰动
            p.vx += (Math.random() - 0.5) * typingImp * 2.8
            p.vy += (Math.random() - 0.5) * typingImp * 2.8
        }

        p.vx *= damp
        p.vy *= damp
        p.x += p.vx
        p.y += p.vy

        // appear & fading
        const targetAppear = p.fading ? 0 : 1
        p.appear += (targetAppear - p.appear) * 0.08

        if (p.fading && p.appear < 0.02)
            continue

        const alphaMod = 0.85 + Math.sin(time * (1.4 + typingImp * 2.2) + p.phase) * (0.15 + typingImp * 0.35)

        ctx.globalAlpha = p.alpha * p.appear * alphaMod
        ctx.beginPath()
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2)
        ctx.fill()

        // 压缩数组（移除已消失的粒子）
        if (writeIndex !== i)
            particles[writeIndex] = p
        writeIndex++
    }

    particles.length = writeIndex
    ctx.globalAlpha = 1

    time += 0.016
    impulse.value *= 0.93 // 冲击衰减

    animationFrame = requestAnimationFrame(animate)
}

// 采样图片生成粒子
function sampleImage(img: HTMLImageElement): Particle[] {
    if (!ctx || !canvasRef.value)
        return []

    const canvas = canvasRef.value
    const dpr = Math.min(window.devicePixelRatio || 1, 2)

    const targetW = canvas.width / dpr
    const targetH = canvas.height / dpr

    // 计算保持比例的绘制尺寸
    const ratio = img.width / img.height
    let drawW = targetW
    let drawH = targetH

    if (ratio > targetW / targetH) {
        drawH = targetW / ratio
    }
    else {
        drawW = targetH * ratio
    }

    drawW *= props.renderScale
    drawH *= props.renderScale

    const tempCanvas = document.createElement('canvas')
    const sampleStep = props.sampleStep

    const sampleW = Math.max(80, Math.floor(drawW / sampleStep))
    const sampleH = Math.max(80, Math.floor(drawH / sampleStep))

    tempCanvas.width = sampleW
    tempCanvas.height = sampleH

    const tempCtx = tempCanvas.getContext('2d', { willReadFrequently: true })
    if (!tempCtx)
        return []

    tempCtx.drawImage(img, 0, 0, sampleW, sampleH)

    const imageData = tempCtx.getImageData(0, 0, sampleW, sampleH).data

    const particlesList: Particle[] = []
    const offsetX = (targetW - drawW) / 2
    const offsetY = props.align === 'bottom'
        ? targetH - drawH - Math.min(40, targetH * 0.04)
        : (targetH - drawH) / 2

    for (let y = 0; y < sampleH; y++) {
        for (let x = 0; x < sampleW; x++) {
            const idx = (y * sampleW + x) * 4
            const r = imageData[idx]!
            const g = imageData[idx + 1]!
            const b = imageData[idx + 2]!
            const a = imageData[idx + 3]!

            const brightness = (r + g + b) / 3
            const value = props.invert ? 255 - brightness : brightness

            if (a < 200 || value < props.threshold)
                continue

            const p = value / 255
            if (!props.denseParticles && !(p > 0.8 || (p > 0.5 ? Math.random() < 0.85 : p > 0.25 ? Math.random() < 0.55 : Math.random() < 0.28))) {
                continue
            }

            const worldX = (offsetX + x * (drawW / sampleW) + drawW / sampleW / 2) * dpr
            const worldY = (offsetY + y * (drawH / sampleH) + drawH / sampleH / 2) * dpr

            particlesList.push({
                ox: worldX,
                oy: worldY,
                x: worldX + (Math.random() - 0.5) * 40,
                y: worldY + (Math.random() - 0.5) * 40,
                vx: 0,
                vy: 0,
                size: (props.dotSize + p * 0.9) * dpr,
                alpha: 0.35 + p * 0.6,
                phase: Math.random() * Math.PI * 2,
                springJitter: 0.9 + Math.random() * 0.2,
                appear: 1,
                fading: false,
            })
        }
    }

    return particlesList
}

function loadImage(src: string, updateExisting = false) {
    const img = new Image()
    img.crossOrigin = 'anonymous'
    img.decoding = 'async'

    img.onload = () => {
        imageRef = img
        if (updateExisting && particles.length > 0) {
            updateParticles(img)
        }
        else {
            resetParticles(img)
        }
    }

    img.src = src
}

function resetParticles(img: HTMLImageElement) {
    resizeCanvas()
    particles = sampleImage(img)
}

function updateParticles(img: HTMLImageElement) {
    resizeCanvas()
    // 这里可以实现更智能的粒子复用（与原代码类似使用 Fisher-Yates shuffle），此处简化处理
    particles = sampleImage(img)
}

function resizeCanvas() {
    const container = containerRef.value
    const canvas = canvasRef.value
    if (!container || !canvas)
        return

    const dpr = Math.min(window.devicePixelRatio || 1, 2)
    const rect = container.getBoundingClientRect()

    canvas.width = Math.max(1, Math.floor(rect.width)) * dpr
    canvas.height = Math.max(1, Math.floor(rect.height)) * dpr
    canvas.style.width = `${rect.width}px`
    canvas.style.height = `${rect.height}px`
}

// 鼠标事件
function onPointerMove(e: PointerEvent) {
    const rect = containerRef.value?.getBoundingClientRect()
    if (!rect)
        return
    mouse.x = e.clientX - rect.left
    mouse.y = e.clientY - rect.top
    mouse.active = true
}

function onPointerLeave() {
    mouse.active = false
    mouse.x = -9999
    mouse.y = -9999
}

onMounted(async () => {
    const canvas = canvasRef.value
    if (!canvas)
        return

    ctx = canvas.getContext('2d', { alpha: true })
    if (!ctx)
        return

    // 适配主题颜色
    if (props.adaptToTheme) {
        currentColor.value = isDark()
            ? 'rgba(255, 255, 255, 0.92)'
            : 'rgba(10, 12, 16, 1)'
    }

    resizeCanvas()

    if (props.src) {
        loadImage(props.src)
    }

    window.addEventListener('resize', () => {
        if (imageRef)
            resetParticles(imageRef)
    })

    const container = containerRef.value
    if (container) {
        container.addEventListener('pointermove', onPointerMove)
        container.addEventListener('pointerleave', onPointerLeave)
    }

    animate()
    emit('ready')
})

onUnmounted(() => {
    if (animationFrame)
        cancelAnimationFrame(animationFrame)

    const container = containerRef.value
    if (container) {
        container.removeEventListener('pointermove', onPointerMove)
        container.removeEventListener('pointerleave', onPointerLeave)
    }
})

// 暴露方法给父组件调用（例如动态换图片）
defineExpose({
    updateImage: (newSrc: string) => loadImage(newSrc, true),
    triggerImpulse: (strength = 1) => {
        impulse.value = Math.max(impulse.value, strength)
    },
})
</script>

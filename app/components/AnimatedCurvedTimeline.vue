<template>
    <div
        ref="container"
        class="relative isolate w-full"
    >
        <svg
            v-if="containerHeight > 0"
            aria-hidden="true"
            class="pointer-events-none absolute left-0 top-0 z-0 overflow-visible"
            :style="{ width: `${SVG_WIDTH}px`, height: `${containerHeight}px` }"
            :viewBox="`0 0 ${SVG_WIDTH} ${Math.max(containerHeight, 1)}`"
            fill="none"
            preserveAspectRatio="none"
        >
            <defs>
                <linearGradient
                    :id="lineFadeGradientId"
                    gradientUnits="userSpaceOnUse"
                    x1="0"
                    x2="0"
                    :y1="fadeStartY"
                    :y2="fadeEndY"
                >
                    <stop offset="0%" stop-color="white" />
                    <stop offset="100%" stop-color="transparent" />
                </linearGradient>

                <mask :id="lineMaskId">
                    <rect
                        fill="black"
                        height="100%"
                        width="100%"
                        x="0"
                        y="0"
                    />
                    <rect
                        fill="white"
                        :height="solidAccentHeight"
                        width="100%"
                        x="0"
                        y="0"
                    />
                    <rect
                        v-if="fadeHeight > 0"
                        :fill="`url(#${lineFadeGradientId})`"
                        :height="fadeHeight"
                        width="100%"
                        x="0"
                        :y="fadeStartY"
                    />
                </mask>
            </defs>

            <path
                v-for="segment in segments"
                :key="`base:${segment.key}`"
                class="fill-none stroke-[#e4e4e7] [stroke-linecap:round]"
                :d="segment.d"
                stroke-width="1.5"
            />

            <g :mask="`url(#${lineMaskId})`">
                <path
                    v-for="segment in segments"
                    :key="`accent:${segment.key}`"
                    class="fill-none [stroke-linecap:round]"
                    :d="segment.d"
                    :stroke="accentColor"
                    stroke-width="1.5"
                />
            </g>

            <template
                v-for="node in nodes"
                :key="node.key"
            >
                <circle
                    v-if="node.type === 'item' && node.item && isCurrentItem(node.item.id)"
                    class="opacity-95"
                    :cx="node.x"
                    :cy="node.y"
                    fill="color-mix(in oklab, var(--timeline-accent, #ff5c87) 18%, white)"
                    r="15"
                />
                <circle
                    v-if="node.type === 'item' && node.item && isCurrentItem(node.item.id)"
                    class="drop-shadow-[0_10px_16px_rgba(255,92,135,0.35)] transition-[fill] duration-200"
                    :cx="node.x"
                    :cy="node.y"
                    :fill="accentColor"
                    r="8"
                    stroke="white"
                    stroke-width="3"
                />
                <circle
                    v-else
                    class="transition-[fill] duration-200"
                    :cx="node.x"
                    :cy="node.y"
                    :fill="nodeFill(node)"
                    r="5.5"
                    stroke="white"
                    stroke-width="2"
                />
            </template>
        </svg>

        <div
            class="relative z-10"
            :style="{ '--timeline-accent': accentColor }"
        >
            <section
                v-for="group in groups"
                :key="group.id"
                class="space-y-4 py-2"
            >
                <div
                    :ref="createRowRef(`header:${group.id}`)"
                    class="relative flex min-h-12 items-center"
                >
                    <div
                        class="w-full"
                        :style="contentInsetStyle"
                    >
                        <h2 class="text-[clamp(1.5rem,1.75vw,1.95rem)] font-semibold tracking-[-0.04em] text-zinc-950">
                            {{ group.label }}
                        </h2>
                    </div>
                </div>

                <div
                    v-for="item in group.items"
                    :key="item.id"
                    :ref="createRowRef(`item:${item.id}`)"
                    class="relative flex min-h-12 items-center"
                >
                    <div
                        class="w-full"
                        :style="contentInsetStyle"
                    >
                        <Button
                            size="lg"
                            variant="ghost"
                            class="h-auto w-full justify-start rounded-[26px] px-5 py-3 text-left whitespace-normal shadow-none transition-all duration-300 sm:px-6"
                            :class="[
                                isCurrentItem(item.id)
                                    ? 'bg-zinc-100/95 shadow-[0_24px_60px_-38px_rgba(0,0,0,0.35)] ring-1 ring-zinc-200/80 hover:bg-zinc-100'
                                    : 'hover:bg-zinc-50/85',
                            ]"
                            @click="handleItemSelect(group.id, item)"
                        >
                            <p
                                class="text-[clamp(0.925rem,1.1vw,1.08rem)] font-medium tracking-[-0.02em] transition-colors duration-300"
                                :class="itemTextClass(item.id)"
                            >
                                {{ item.title }}
                            </p>
                        </Button>
                    </div>
                </div>
            </section>
        </div>
    </div>
</template>

<script lang="ts" setup>
import type { ComponentPublicInstance } from 'vue'

import { useResizeObserver, useTransition } from '@vueuse/core'

interface TimelineEntry {
    id: string
    title: string
    current?: boolean
}

interface TimelineGroup {
    id: string
    label: string
    items: TimelineEntry[]
}

interface TimelineRow {
    key: string
    type: 'header' | 'item'
    groupId: string
    label: string
    item?: TimelineEntry
}

interface TimelineNode extends TimelineRow {
    x: number
    y: number
}

interface TimelineSegment {
    key: string
    d: string
}

interface Props {
    groups: TimelineGroup[]
    activeItemId?: string
    accentColor?: string
}

const props = withDefaults(defineProps<Props>(), {
    accentColor: '#ff5c87',
})

const emit = defineEmits<{
    select: [payload: { groupId: string, item: TimelineEntry }]
}>()

const SVG_WIDTH = 60
const layout = {
    headerNodeX: Math.round(SVG_WIDTH * 0.304),
    itemNodeX: Math.round(SVG_WIDTH * 0.739),
    contentInset: SVG_WIDTH - 4,
} as const

const instanceId = useId()
const lineFadeGradientId = `${instanceId}-timeline-fade-gradient`
const lineMaskId = `${instanceId}-timeline-line-mask`
const containerRef = useTemplateRef<HTMLElement>('container')
const rowElements = new Map<string, HTMLElement>()
const rowCenters = shallowRef<Record<string, number>>({})
const containerHeight = shallowRef(0)
const internalActiveItemId = shallowRef<string | null>(null)

const rows = computed<TimelineRow[]>(() => {
    return props.groups.flatMap((group) => {
        const groupRows: TimelineRow[] = [
            {
                key: `header:${group.id}`,
                type: 'header',
                groupId: group.id,
                label: group.label,
            },
        ]

        group.items.forEach((item) => {
            groupRows.push({
                key: `item:${item.id}`,
                type: 'item',
                groupId: group.id,
                label: item.title,
                item,
            })
        })

        return groupRows
    })
})

const rowIndexMap = computed(() => {
    return new Map(rows.value.map((row, index) => [row.key, index]))
})

function getDefaultActiveItemId(groups: TimelineGroup[]) {
    return groups
        .flatMap(group => group.items)
        .find(item => item.current)
        ?.id ?? groups[0]?.items[0]?.id ?? null
}

watch(
    () => [props.activeItemId, props.groups] as const,
    ([activeItemId, groups]) => {
        if (activeItemId) {
            internalActiveItemId.value = activeItemId
            return
        }

        const itemIds = new Set(groups.flatMap(group => group.items.map(item => item.id)))

        if (internalActiveItemId.value && itemIds.has(internalActiveItemId.value))
            return

        internalActiveItemId.value = getDefaultActiveItemId(groups)
    },
    { deep: true, immediate: true },
)

const resolvedActiveItemId = computed(() => internalActiveItemId.value)

const activeRowKey = computed(() => {
    return resolvedActiveItemId.value
        ? `item:${resolvedActiveItemId.value}`
        : null
})

const activeRowIndex = computed(() => {
    if (!activeRowKey.value)
        return -1

    return rowIndexMap.value.get(activeRowKey.value) ?? -1
})

const activeNodeYTarget = computed(() => {
    if (!activeRowKey.value)
        return 0

    return rowCenters.value[activeRowKey.value] ?? 0
})

const animatedActiveNodeY = useTransition(activeNodeYTarget, {
    duration: 420,
})

const contentInsetStyle = computed(() => ({
    paddingLeft: `${layout.contentInset}px`,
}))

function setRowRef(key: string, element: Element | ComponentPublicInstance | null) {
    if (element instanceof HTMLElement)
        rowElements.set(key, element)
    else
        rowElements.delete(key)
}

function createRowRef(key: string) {
    return (element: Element | ComponentPublicInstance | null) => {
        setRowRef(key, element)
    }
}

function updateLayout() {
    const container = containerRef.value

    if (!container)
        return

    const containerRect = container.getBoundingClientRect()
    const nextCenters: Record<string, number> = {}

    rows.value.forEach((row) => {
        const element = rowElements.get(row.key)

        if (!element)
            return

        const rect = element.getBoundingClientRect()
        nextCenters[row.key] = rect.top - containerRect.top + (rect.height / 2)
    })

    rowCenters.value = nextCenters
    containerHeight.value = containerRect.height
}

function queueLayoutUpdate() {
    if (typeof window === 'undefined')
        return

    requestAnimationFrame(updateLayout)
}

useResizeObserver(containerRef, queueLayoutUpdate)

watch(
    () => props.groups,
    async () => {
        await nextTick()
        queueLayoutUpdate()
    },
    { deep: true, immediate: true },
)

onMounted(async () => {
    await nextTick()
    queueLayoutUpdate()
})

const nodes = computed<TimelineNode[]>(() => {
    return rows.value
        .map((row) => {
            const y = rowCenters.value[row.key]

            if (!y)
                return null

            return {
                ...row,
                x: row.type === 'header' ? layout.headerNodeX : layout.itemNodeX,
                y,
            }
        })
        .filter((node): node is TimelineNode => node !== null)
})

function buildSegmentPath(start: TimelineNode, end: TimelineNode) {
    if (Math.abs(start.x - end.x) < 1)
        return `M ${start.x} ${start.y} L ${end.x} ${end.y}`

    const deltaY = end.y - start.y
    const stem = Math.min(Math.max(deltaY * 0.18, 10), 18)
    const topY = start.y + stem
    const bottomY = end.y - stem
    const midY = start.y + (deltaY * 0.48)
    const midX = start.x + ((end.x - start.x) * 0.36)

    return [
        `M ${start.x} ${start.y}`,
        `L ${start.x} ${topY}`,
        `Q ${start.x} ${midY} ${midX} ${midY}`,
        `Q ${end.x} ${midY} ${end.x} ${bottomY}`,
        `L ${end.x} ${end.y}`,
    ].join(' ')
}

const segments = computed<TimelineSegment[]>(() => {
    return nodes.value
        .slice(0, -1)
        .map((node, index) => {
            const nextNode = nodes.value[index + 1]

            if (!nextNode)
                return null

            return {
                key: `${node.key}:${nextNode.key}`,
                d: buildSegmentPath(node, nextNode),
            }
        })
        .filter((segment): segment is TimelineSegment => segment !== null)
})

const fadeStartY = computed(() => {
    return Math.max(animatedActiveNodeY.value + 10, 0)
})

const fadeEndY = computed(() => {
    return Math.min(animatedActiveNodeY.value + 70, containerHeight.value)
})

const fadeHeight = computed(() => {
    return Math.max(fadeEndY.value - fadeStartY.value, 0)
})

const solidAccentHeight = computed(() => {
    return Math.min(fadeStartY.value, containerHeight.value)
})

function isCurrentItem(itemId: string) {
    return resolvedActiveItemId.value === itemId
}

function handleItemSelect(groupId: string, item: TimelineEntry) {
    internalActiveItemId.value = item.id
    emit('select', { groupId, item })
}

function isAfterActiveItem(itemId: string) {
    if (activeRowIndex.value < 0)
        return false

    const rowIndex = rowIndexMap.value.get(`item:${itemId}`) ?? -1
    return rowIndex > activeRowIndex.value
}

function nodeFill(node: TimelineNode) {
    if (node.type === 'item' && node.item && isAfterActiveItem(node.item.id))
        return '#cfd2d8'

    return '#a1a1aa'
}

function itemTextClass(itemId: string) {
    if (isCurrentItem(itemId))
        return 'text-zinc-950'

    return isAfterActiveItem(itemId) ? 'text-zinc-400' : 'text-zinc-500'
}
</script>

<template>
    <div class="relative w-full overflow-hidden rounded-2xl border border-border/40 bg-background/60 p-4 backdrop-blur sm:p-6">
        <div class="mb-4 flex flex-wrap items-center justify-between gap-3">
            <div class="flex items-center gap-3">
                <Badge
                    variant="outline"
                    class="rounded-full border-emerald-400/40 bg-emerald-400/10 px-3 py-1 text-xs font-semibold uppercase tracking-[0.25em] text-emerald-400"
                >
                    Active
                </Badge>
                <span class="text-xs uppercase tracking-[0.25em] text-foreground/50 sm:text-sm">
                    Workflow Builder
                </span>
            </div>

            <Button
                variant="outline"
                size="sm"
                class="h-8 rounded-lg text-xs uppercase tracking-[0.2em] text-foreground/70 hover:text-foreground"
                aria-label="Add new node"
                @click="addNode"
            >
                <Plus class="size-3.5" aria-hidden="true" />
                <span class="hidden sm:inline">Add Node</span>
            </Button>
        </div>

        <div
            ref="canvas"
            class="relative h-[400px] w-full overflow-auto rounded-xl border border-border/30 bg-background/40 sm:h-[500px] md:h-[600px]"
            style="min-height: 400px;"
            role="region"
            aria-label="Workflow canvas"
            tabindex="0"
        >
            <div
                class="relative"
                :style="{
                    minWidth: `${contentSize.width}px`,
                    minHeight: `${contentSize.height}px`,
                }"
            >
                <svg
                    class="pointer-events-none absolute top-0 left-0"
                    :width="contentSize.width"
                    :height="contentSize.height"
                    style="overflow: visible;"
                    aria-hidden="true"
                >
                    <path
                        v-for="path in connectionPaths"
                        :key="path.id"
                        :d="path.d"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-dasharray="8,6"
                        stroke-linecap="round"
                        opacity="0.35"
                        class="text-foreground"
                    />
                </svg>

                <Motion
                    v-for="node in nodes"
                    :key="node.id"
                    as="div"
                    :style="getNodeStyle(node.position)"
                    :initial="{ scale: 0.8, opacity: 0 }"
                    :animate="{
                        scale: draggingNodeId === node.id ? 1.05 : 1,
                        opacity: 1,
                    }"
                    :transition="{ duration: 0.2 }"
                    :while-hover="draggingNodeId === node.id ? undefined : { scale: 1.02 }"
                    :class="cn(
                        'absolute select-none touch-none',
                        draggingNodeId === node.id ? 'z-50 cursor-grabbing' : 'cursor-grab',
                    )"
                    :aria-grabbed="draggingNodeId === node.id"
                    @pointerdown="startDragging(node.id, $event)"
                >
                    <Card
                        :class="cn(
                            'group/node relative w-full overflow-hidden rounded-xl bg-background/70 p-3 backdrop-blur transition-all hover:shadow-lg',
                            colorClasses[node.color],
                            draggingNodeId === node.id && 'shadow-xl ring-2 ring-primary/50',
                        )"
                        role="article"
                        :aria-label="`${node.type} node: ${node.title}`"
                    >
                        <div class="absolute inset-0 bg-gradient-to-br from-foreground/[0.04] via-transparent to-transparent opacity-0 transition-opacity duration-300 group-hover/node:opacity-100" />

                        <div class="relative space-y-2">
                            <div class="flex items-start gap-2">
                                <div
                                    :class="cn(
                                        'flex size-8 shrink-0 items-center justify-center rounded-lg border bg-background/80 backdrop-blur',
                                        colorClasses[node.color],
                                    )"
                                    aria-hidden="true"
                                >
                                    <component :is="node.icon" class="size-4" />
                                </div>

                                <div class="min-w-0 flex-1">
                                    <Badge
                                        variant="outline"
                                        class="mb-0.5 rounded-full border-border/40 bg-background/80 px-1.5 py-0 text-[9px] uppercase tracking-[0.15em] text-foreground/60"
                                    >
                                        {{ node.type }}
                                    </Badge>
                                    <h3 class="truncate text-xs font-semibold tracking-tight text-foreground">
                                        {{ node.title }}
                                    </h3>
                                </div>

                                <Button
                                    v-if="getParentConnection(node.id)"
                                    variant="ghost"
                                    size="icon-sm"
                                    class="mt-0.5 size-7 rounded-md text-foreground/50 hover:text-foreground"
                                    aria-label="Add parallel flow"
                                    @pointerdown.stop
                                    @click.stop="addParallelNode(node.id)"
                                >
                                    <Plus class="size-3.5" aria-hidden="true" />
                                </Button>
                            </div>

                            <p class="line-clamp-2 text-[10px] leading-relaxed text-foreground/70">
                                {{ node.description }}
                            </p>

                            <div class="flex items-center gap-1.5 text-[10px] text-foreground/50">
                                <ArrowRight class="size-2.5" aria-hidden="true" />
                                <span class="uppercase tracking-[0.1em]">
                                    Connected
                                </span>
                            </div>
                        </div>
                    </Card>
                </Motion>
            </div>
        </div>

        <div
            class="mt-4 flex flex-wrap items-center justify-between gap-3 rounded-lg border border-border/30 bg-background/40 px-4 py-2.5 backdrop-blur-sm"
            role="status"
            aria-live="polite"
        >
            <div class="flex flex-wrap items-center gap-4 text-xs text-foreground/60">
                <div class="flex items-center gap-2">
                    <div class="size-1.5 rounded-full bg-emerald-500" aria-hidden="true" />
                    <span class="uppercase tracking-[0.15em]">
                        {{ nodes.length }} {{ nodes.length === 1 ? 'Node' : 'Nodes' }}
                    </span>
                </div>

                <div class="flex items-center gap-2">
                    <div class="size-1.5 rounded-full bg-primary" aria-hidden="true" />
                    <span class="uppercase tracking-[0.15em]">
                        {{ connections.length }}
                        {{ connections.length === 1 ? 'Connection' : 'Connections' }}
                    </span>
                </div>
            </div>

            <p class="text-[10px] uppercase tracking-[0.2em] text-foreground/40">
                Drag nodes to reposition
            </p>
        </div>
    </div>
</template>

<script setup lang="ts">
import type { Component, HTMLAttributes } from 'vue'
import { useEventListener } from '@vueuse/core'
import {
    ArrowRight,
    Database,
    Mail,
    Plus,
    Settings,
    Webhook,
    Zap,
} from 'lucide-vue-next'
import { Motion } from 'motion-v'
import { computed, nextTick, onUnmounted, ref, shallowRef, useTemplateRef } from 'vue'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Card } from '@/components/ui/card'
import { cn } from '@/lib/utils'

interface Position {
    x: number
    y: number
}

type WorkflowNodeType = 'trigger' | 'action' | 'condition'
type WorkflowNodeColor = 'emerald' | 'blue' | 'amber' | 'purple' | 'indigo'

interface WorkflowNode {
    id: string
    type: WorkflowNodeType
    title: string
    description: string
    icon: Component
    color: WorkflowNodeColor
    position: Position
}

interface WorkflowConnection {
    from: string
    to: string
}

interface WorkflowConnectionPath {
    id: string
    d: string
}

interface DragState {
    nodeId: string
    pointerId: number
    pointerStart: Position
    nodeStart: Position
}

type WorkflowNodeTemplate = Omit<WorkflowNode, 'id' | 'position'>

const NODE_WIDTH = 200
const NODE_HEIGHT = 100
const CANVAS_PADDING = 50

const nodeTemplates: WorkflowNodeTemplate[] = [
    {
        type: 'trigger',
        title: 'Webhook',
        description: 'Receive data from external service',
        icon: Webhook,
        color: 'emerald',
    },
    {
        type: 'action',
        title: 'Database Query',
        description: 'Fetch user records',
        icon: Database,
        color: 'blue',
    },
    {
        type: 'condition',
        title: 'Condition',
        description: 'Check user status',
        icon: Settings,
        color: 'amber',
    },
    {
        type: 'action',
        title: 'Send Email',
        description: 'Notify user',
        icon: Mail,
        color: 'purple',
    },
    {
        type: 'action',
        title: 'Log Event',
        description: 'Record activity',
        icon: Zap,
        color: 'indigo',
    },
]

const nodes = ref<WorkflowNode[]>([
    {
        id: 'node-1',
        type: 'trigger',
        title: 'Webhook',
        description: 'Receive data from external service',
        icon: Webhook,
        color: 'emerald',
        position: { x: 50, y: 100 },
    },
    {
        id: 'node-2',
        type: 'action',
        title: 'Database Query',
        description: 'Fetch user records',
        icon: Database,
        color: 'blue',
        position: { x: 300, y: 100 },
    },
    {
        id: 'node-3',
        type: 'condition',
        title: 'Condition',
        description: 'Check user status',
        icon: Settings,
        color: 'amber',
        position: { x: 550, y: 100 },
    },
])

const connections = ref<WorkflowConnection[]>([
    { from: 'node-1', to: 'node-2' },
    { from: 'node-2', to: 'node-3' },
])

const colorClasses: Record<WorkflowNodeColor, HTMLAttributes['class']> = {
    emerald: 'border-emerald-400/40 bg-emerald-400/10 text-emerald-400',
    blue: 'border-blue-400/40 bg-blue-400/10 text-blue-400',
    amber: 'border-amber-400/40 bg-amber-400/10 text-amber-400',
    purple: 'border-purple-400/40 bg-purple-400/10 text-purple-400',
    indigo: 'border-indigo-400/40 bg-indigo-400/10 text-indigo-400',
}

const canvasRef = useTemplateRef<HTMLDivElement>('canvas')
const dragState = shallowRef<DragState | null>(null)
const previousUserSelect = shallowRef('')

const draggingNodeId = computed(() => dragState.value?.nodeId ?? null)

const nodeMap = computed(() => {
    return new Map(nodes.value.map(node => [node.id, node]))
})

const contentSize = computed(() => {
    const maxX = Math.max(0, ...nodes.value.map(node => node.position.x + NODE_WIDTH))
    const maxY = Math.max(0, ...nodes.value.map(node => node.position.y + NODE_HEIGHT))

    return {
        width: maxX + CANVAS_PADDING,
        height: maxY + CANVAS_PADDING,
    }
})

const connectionPaths = computed<WorkflowConnectionPath[]>(() => {
    return connections.value.flatMap((connection) => {
        const fromNode = nodeMap.value.get(connection.from)
        const toNode = nodeMap.value.get(connection.to)

        if (!fromNode || !toNode) {
            return []
        }

        const startX = fromNode.position.x + NODE_WIDTH
        const startY = fromNode.position.y + NODE_HEIGHT / 2
        const endX = toNode.position.x
        const endY = toNode.position.y + NODE_HEIGHT / 2
        const cp1X = startX + (endX - startX) * 0.5
        const cp2X = endX - (endX - startX) * 0.5

        return [{
            id: `${connection.from}-${connection.to}`,
            d: `M${startX},${startY} C${cp1X},${startY} ${cp2X},${endY} ${endX},${endY}`,
        }]
    })
})

function getParentConnection(nodeId: string) {
    return connections.value.find(connection => connection.to === nodeId) ?? null
}

function getSiblingNodes(parentId: string) {
    return connections.value
        .filter(connection => connection.from === parentId)
        .map(connection => nodeMap.value.get(connection.to))
        .filter((node): node is WorkflowNode => Boolean(node))
}

function startDragging(nodeId: string, event: PointerEvent) {
    if (event.button !== 0) {
        return
    }

    const node = nodeMap.value.get(nodeId)

    if (!node) {
        return
    }

    event.preventDefault()

    previousUserSelect.value = document.body.style.userSelect
    document.body.style.userSelect = 'none'

    dragState.value = {
        nodeId,
        pointerId: event.pointerId,
        pointerStart: { x: event.clientX, y: event.clientY },
        nodeStart: { ...node.position },
    }

    const target = event.currentTarget as HTMLElement | null
    target?.setPointerCapture?.(event.pointerId)
}

function updateDragging(event: PointerEvent) {
    const activeDrag = dragState.value

    if (!activeDrag || event.pointerId !== activeDrag.pointerId) {
        return
    }

    const node = nodeMap.value.get(activeDrag.nodeId)

    if (!node) {
        stopDragging(event.pointerId)
        return
    }

    const offsetX = event.clientX - activeDrag.pointerStart.x
    const offsetY = event.clientY - activeDrag.pointerStart.y

    node.position.x = Math.max(0, activeDrag.nodeStart.x + offsetX)
    node.position.y = Math.max(0, activeDrag.nodeStart.y + offsetY)
}

function stopDragging(pointerId?: number) {
    if (pointerId !== undefined && dragState.value && dragState.value.pointerId !== pointerId) {
        return
    }

    dragState.value = null
    document.body.style.userSelect = previousUserSelect.value
}

async function addNode() {
    const template = nodeTemplates[Math.floor(Math.random() * nodeTemplates.length)]!
    const lastNode = nodes.value.at(-1)
    const position = lastNode
        ? { x: lastNode.position.x + 250, y: lastNode.position.y }
        : { x: 50, y: 100 }

    const newNode: WorkflowNode = {
        id: `node-${Date.now()}`,
        ...template,
        position,
    }

    nodes.value = [...nodes.value, newNode]

    if (lastNode) {
        connections.value = [
            ...connections.value,
            { from: lastNode.id, to: newNode.id },
        ]
    }

    await scrollToNode(position)
}

async function addParallelNode(nodeId: string) {
    const parentConnection = getParentConnection(nodeId)

    if (!parentConnection) {
        return
    }

    const anchorNode = nodeMap.value.get(nodeId)

    if (!anchorNode) {
        return
    }

    const template = nodeTemplates[Math.floor(Math.random() * nodeTemplates.length)]!
    const siblingNodes = getSiblingNodes(parentConnection.from)
    const maxSiblingY = Math.max(...siblingNodes.map(node => node.position.y))
    const position = {
        x: anchorNode.position.x,
        y: maxSiblingY + NODE_HEIGHT + 50,
    }

    const newNode: WorkflowNode = {
        id: `node-${Date.now()}`,
        ...template,
        position,
    }

    nodes.value = [...nodes.value, newNode]
    connections.value = [
        ...connections.value,
        { from: parentConnection.from, to: newNode.id },
    ]

    await scrollToNode(position)
}

async function scrollToNode(position: Position) {
    await nextTick()

    const canvas = canvasRef.value

    if (!canvas) {
        return
    }

    canvas.scrollTo({
        left: Math.max(0, position.x + NODE_WIDTH - canvas.clientWidth + 100),
        top: Math.max(0, position.y + NODE_HEIGHT - canvas.clientHeight + 100),
        behavior: 'smooth',
    })
}

function getNodeStyle(position: Position) {
    return {
        left: `${position.x}px`,
        top: `${position.y}px`,
        width: `${NODE_WIDTH}px`,
        transformOrigin: '0 0',
    }
}

if (import.meta.client) {
    useEventListener(window, 'pointermove', updateDragging)
    useEventListener(window, 'pointerup', event => stopDragging(event.pointerId))
    useEventListener(window, 'pointercancel', event => stopDragging(event.pointerId))
}

onUnmounted(() => stopDragging())
</script>

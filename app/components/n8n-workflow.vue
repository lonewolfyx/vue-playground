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

            <div class="flex flex-wrap items-center gap-2">
                <div class="flex items-center rounded-lg border border-border/40 bg-background/60 p-1 backdrop-blur">
                    <Button
                        variant="ghost"
                        size="icon-sm"
                        class="size-7 rounded-md text-foreground/60 hover:text-foreground"
                        aria-label="Zoom out"
                        @click="zoomOut"
                    >
                        <Minus class="size-3.5" aria-hidden="true" />
                    </Button>
                    <span class="min-w-14 px-2 text-center text-[10px] font-medium uppercase tracking-[0.2em] text-foreground/50">
                        {{ zoomPercentage }}
                    </span>
                    <Button
                        variant="ghost"
                        size="icon-sm"
                        class="size-7 rounded-md text-foreground/60 hover:text-foreground"
                        aria-label="Zoom in"
                        @click="zoomIn"
                    >
                        <Plus class="size-3.5" aria-hidden="true" />
                    </Button>
                    <Button
                        variant="ghost"
                        size="icon-sm"
                        class="size-7 rounded-md text-foreground/50 hover:text-foreground"
                        aria-label="Reset zoom"
                        @click="resetZoom"
                    >
                        <RotateCcw class="size-3.5" aria-hidden="true" />
                    </Button>
                </div>

                <Button
                    variant="outline"
                    size="sm"
                    class="h-8 rounded-lg text-xs uppercase tracking-[0.2em] text-foreground/70 hover:text-foreground"
                    aria-label="Copy workflow data"
                    @click="copyWorkflowData"
                >
                    <Copy class="size-3.5" aria-hidden="true" />
                    <span class="hidden sm:inline">Copy Data</span>
                </Button>

                <Button
                    variant="outline"
                    size="sm"
                    class="h-8 rounded-lg text-xs uppercase tracking-[0.2em] text-foreground/70 hover:text-destructive"
                    aria-label="Clear workflow"
                    @click="clearWorkflow"
                >
                    <Trash2 class="size-3.5" aria-hidden="true" />
                    <span class="hidden sm:inline">Clear All</span>
                </Button>

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
        </div>

        <div
            ref="canvas"
            :class="cn(
                'relative h-[400px] w-full overflow-hidden rounded-xl border border-border/30 bg-background/40 sm:h-[500px] md:h-[600px]',
                isPanning ? 'cursor-grabbing' : 'cursor-grab',
            )"
            style="min-height: 400px;"
            role="region"
            aria-label="Workflow canvas"
            tabindex="0"
            @pointerdown="startPanning"
        >
            <div class="absolute inset-0">
                <div
                    class="absolute inset-0"
                    :style="canvasBackgroundStyle"
                    aria-hidden="true"
                />

                <div
                    class="absolute top-0 left-0 origin-top-left"
                    :style="sceneTransformStyle"
                >
                    <svg
                        class="pointer-events-none absolute top-0 left-0"
                        :width="sceneBounds.width"
                        :height="sceneBounds.height"
                        style="overflow: visible;"
                        aria-hidden="true"
                    >
                        <path
                            v-for="path in connectionPaths"
                            :key="path.id"
                            :d="path.d"
                            fill="none"
                            stroke="currentColor"
                            :stroke-width="CONNECTION_STROKE_WIDTH"
                            :stroke-dasharray="connectionDashArray"
                            stroke-linecap="round"
                            opacity="0.35"
                            class="text-foreground"
                        />
                        <path
                            v-if="activeConnectionPreviewPath"
                            :d="activeConnectionPreviewPath"
                            fill="none"
                            stroke="rgb(59 130 246 / 0.9)"
                            stroke-width="1.6"
                            stroke-dasharray="6,6"
                            stroke-linecap="round"
                        />
                    </svg>

                    <Motion
                        v-for="node in nodes"
                        :key="node.id"
                        as="div"
                        data-workflow-node
                        :data-workflow-node-id="node.id"
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
                                'group/node relative h-full w-full overflow-hidden rounded-xl bg-background/70 p-3 backdrop-blur transition-all hover:shadow-lg',
                                colorClasses[node.color],
                                draggingNodeId === node.id && 'shadow-xl ring-2 ring-primary/50',
                            )"
                            role="article"
                            :aria-label="`${node.type} node: ${node.title}`"
                        >
                            <div class="absolute inset-0 bg-gradient-to-br from-foreground/[0.04] via-transparent to-transparent opacity-0 transition-opacity duration-300 group-hover/node:opacity-100" />
                            <button
                                v-if="getParentConnection(node.id)"
                                type="button"
                                class="absolute top-1/2 -left-2 z-10 flex size-4 -translate-y-1/2 items-center justify-center rounded-full border border-blue-400/60 bg-background/95 shadow-sm transition-transform hover:scale-110"
                                aria-label="Reconnect incoming edge"
                                @pointerdown="startConnectionDrag(node.id, 'retarget', $event)"
                            >
                                <span class="block size-1.5 rounded-full bg-blue-500/90" />
                            </button>
                            <button
                                type="button"
                                class="absolute top-1/2 -right-2 z-10 flex size-4 -translate-y-1/2 items-center justify-center rounded-full border border-emerald-400/60 bg-background/95 shadow-sm transition-transform hover:scale-110"
                                aria-label="Create outgoing edge"
                                @pointerdown="startConnectionDrag(node.id, 'create', $event)"
                            >
                                <span class="block size-1.5 rounded-full bg-emerald-500/90" />
                            </button>

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

                                    <div class="mt-0.5 flex shrink-0 items-center gap-1">
                                        <Button
                                            v-if="getParentConnection(node.id)"
                                            variant="ghost"
                                            size="icon-sm"
                                            class="size-7 rounded-md text-foreground/50 hover:text-foreground"
                                            aria-label="Add parallel flow"
                                            @pointerdown.stop
                                            @click.stop="addParallelNode(node.id)"
                                        >
                                            <Plus class="size-3.5" aria-hidden="true" />
                                        </Button>

                                        <Button
                                            variant="ghost"
                                            size="icon-sm"
                                            class="size-7 rounded-md text-foreground/40 hover:text-destructive"
                                            aria-label="Delete node"
                                            @pointerdown.stop
                                            @click.stop="removeNode(node.id)"
                                        >
                                            <Trash2 class="size-3.5" aria-hidden="true" />
                                        </Button>
                                    </div>
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

                                <Button
                                    variant="outline"
                                    size="sm"
                                    class="mt-2 h-7 w-full rounded-lg border-dashed text-[10px] uppercase tracking-[0.15em] text-foreground/60 hover:text-foreground"
                                    aria-label="Add child node"
                                    @pointerdown.stop
                                    @click.stop="addChildNode(node.id)"
                                >
                                    <ArrowDown class="size-3.5" aria-hidden="true" />
                                    <span>Add Next</span>
                                </Button>
                            </div>
                        </Card>
                    </Motion>
                </div>

                <div
                    class="absolute right-4 bottom-4 z-20 overflow-hidden rounded-xl border border-border/50 bg-background/85 p-2 shadow-lg backdrop-blur-sm"
                >
                    <div class="mb-1 flex items-center justify-between gap-3 px-1">
                        <span class="text-[9px] font-medium uppercase tracking-[0.24em] text-foreground/45">
                            Mini Map
                        </span>
                        <span class="text-[9px] uppercase tracking-[0.18em] text-foreground/35">
                            Drag to navigate
                        </span>
                    </div>

                    <div
                        ref="miniMap"
                        class="relative h-[116px] w-[176px] cursor-pointer rounded-lg bg-background/90"
                        @pointerdown="startMiniMapDrag"
                    >
                        <svg
                            class="absolute inset-0"
                            :width="MINIMAP_WIDTH"
                            :height="MINIMAP_HEIGHT"
                            aria-hidden="true"
                        >
                            <rect
                                x="0"
                                y="0"
                                :width="MINIMAP_WIDTH"
                                :height="MINIMAP_HEIGHT"
                                rx="10"
                                fill="rgba(148, 163, 184, 0.08)"
                            />

                            <path
                                v-for="path in miniMapConnectionPaths"
                                :key="path.id"
                                :d="path.d"
                                fill="none"
                                stroke="rgba(148, 163, 184, 0.65)"
                                :stroke-width="Math.max(0.6, CONNECTION_STROKE_WIDTH * 0.7)"
                                :stroke-dasharray="`${Math.max(2, CONNECTION_DASH_LENGTH * 0.4)},${Math.max(2.5, CONNECTION_DASH_GAP * 0.4)}`"
                                stroke-linecap="round"
                            />

                            <rect
                                v-for="node in miniMapNodes"
                                :key="node.id"
                                :x="node.x"
                                :y="node.y"
                                :width="Math.max(node.width, 8)"
                                :height="Math.max(node.height, 8)"
                                rx="4"
                                :fill="miniMapColors[node.color]"
                                fill-opacity="0.22"
                                :stroke="miniMapColors[node.color]"
                                stroke-opacity="0.7"
                                :stroke-width="1"
                            />

                            <rect
                                :x="miniMapViewport.x"
                                :y="miniMapViewport.y"
                                :width="Math.max(miniMapViewport.width, 14)"
                                :height="Math.max(miniMapViewport.height, 14)"
                                rx="6"
                                fill="rgba(255,255,255,0.16)"
                                stroke="rgba(255,255,255,0.9)"
                                :stroke-width="1.2"
                            />
                        </svg>
                    </div>
                </div>
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
                Drag nodes or hold Ctrl/Cmd + scroll to zoom
            </p>
        </div>
    </div>
</template>

<script setup lang="ts">
import type { Component, HTMLAttributes } from 'vue'
import { useClipboard, useElementSize, useEventListener } from '@vueuse/core'
import {
    ArrowDown,
    ArrowRight,
    Copy,
    Database,
    Mail,
    Minus,
    Plus,
    RotateCcw,
    Settings,
    Trash2,
    Webhook,
    Zap,
} from 'lucide-vue-next'
import { Motion } from 'motion-v'
import { computed, nextTick, onUnmounted, ref, shallowRef, useTemplateRef, watch } from 'vue'
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

interface PanState {
    pointerId: number
    pointerStart: Position
    viewportStart: Position
}

interface MiniMapDragState {
    pointerId: number
    mode: 'viewport' | 'jump'
    grabOffset: Position
}

interface ConnectionDragState {
    pointerId: number
    mode: 'create' | 'retarget'
    sourceNodeId: string
    originalTargetNodeId?: string
    currentWorldPoint: Position
}

type WorkflowNodeTemplate = Omit<WorkflowNode, 'id' | 'position'>

const NODE_WIDTH = 200
const NODE_HEIGHT = 180
const NODE_HORIZONTAL_GAP = 50
const NODE_VERTICAL_GAP = 40
const CANVAS_PADDING = 50
const MIN_ZOOM = 0.5
const MAX_ZOOM = 1.75
const ZOOM_STEP = 0.01
const CONNECTION_STROKE_WIDTH = 1
const CONNECTION_DASH_LENGTH = 4
const CONNECTION_DASH_GAP = 12
const MINIMAP_WIDTH = 176
const MINIMAP_HEIGHT = 116
const MINIMAP_PADDING = 10
const GRID_SPACING = 20

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

const miniMapColors: Record<WorkflowNodeColor, string> = {
    emerald: '#34D399',
    blue: '#60A5FA',
    amber: '#FBBF24',
    purple: '#C084FC',
    indigo: '#818CF8',
}

const canvasRef = useTemplateRef<HTMLDivElement>('canvas')
const miniMapRef = useTemplateRef<HTMLDivElement>('miniMap')
const dragState = shallowRef<DragState | null>(null)
const panState = shallowRef<PanState | null>(null)
const miniMapDragState = shallowRef<MiniMapDragState | null>(null)
const connectionDragState = shallowRef<ConnectionDragState | null>(null)
const hasAutoCenteredViewport = shallowRef(false)
const previousUserSelect = shallowRef('')
const previousCanvasCursor = shallowRef('')
const zoomScale = shallowRef(1)
const viewportPosition = ref<Position>({ x: 120, y: 120 })
const { width: canvasWidth, height: canvasHeight } = useElementSize(canvasRef)
const { copy } = useClipboard()

const draggingNodeId = computed(() => dragState.value?.nodeId ?? null)
const isPanning = computed(() => panState.value !== null)
const zoomPercentage = computed(() => `${Math.round(zoomScale.value * 100)}%`)

const nodeMap = computed(() => {
    return new Map(nodes.value.map(node => [node.id, node]))
})

function buildConnectionPath(
    fromX: number,
    fromY: number,
    toX: number,
    toY: number,
    nodeWidth: number,
    nodeHeight: number,
) {
    const direction = toX >= fromX ? 1 : -1
    const startX = fromX + (direction === 1 ? nodeWidth : 0)
    const startY = fromY + nodeHeight / 2
    const endX = toX + (direction === 1 ? 0 : nodeWidth)
    const endY = toY + nodeHeight / 2
    const handleOffset = Math.max(nodeWidth * 0.3, Math.abs(endX - startX) * 0.5)
    const cp1X = startX + direction * handleOffset
    const cp2X = endX - direction * handleOffset

    return `M${startX},${startY} C${cp1X},${startY} ${cp2X},${endY} ${endX},${endY}`
}

function buildCurvePath(startX: number, startY: number, endX: number, endY: number) {
    const direction = endX >= startX ? 1 : -1
    const handleOffset = Math.max(NODE_WIDTH * 0.3, Math.abs(endX - startX) * 0.5)
    const cp1X = startX + direction * handleOffset
    const cp2X = endX - direction * handleOffset

    return `M${startX},${startY} C${cp1X},${startY} ${cp2X},${endY} ${endX},${endY}`
}

const sceneBounds = computed(() => {
    if (nodes.value.length === 0) {
        return {
            minX: -CANVAS_PADDING,
            minY: -CANVAS_PADDING,
            width: CANVAS_PADDING * 2,
            height: CANVAS_PADDING * 2,
            offsetX: CANVAS_PADDING,
            offsetY: CANVAS_PADDING,
        }
    }

    const minX = Math.min(...nodes.value.map(node => node.position.x)) - CANVAS_PADDING
    const minY = Math.min(...nodes.value.map(node => node.position.y)) - CANVAS_PADDING
    const maxX = Math.max(...nodes.value.map(node => node.position.x + NODE_WIDTH)) + CANVAS_PADDING
    const maxY = Math.max(...nodes.value.map(node => node.position.y + NODE_HEIGHT)) + CANVAS_PADDING

    return {
        minX,
        minY,
        width: maxX - minX,
        height: maxY - minY,
        offsetX: -minX,
        offsetY: -minY,
    }
})

const visibleWorldRect = computed(() => {
    return {
        x: -viewportPosition.value.x / zoomScale.value,
        y: -viewportPosition.value.y / zoomScale.value,
        width: canvasWidth.value / zoomScale.value,
        height: canvasHeight.value / zoomScale.value,
    }
})

const navigationBounds = computed(() => {
    const minX = Math.min(0, visibleWorldRect.value.x)
    const minY = Math.min(0, visibleWorldRect.value.y)
    const maxX = Math.max(
        sceneBounds.value.width,
        visibleWorldRect.value.x + visibleWorldRect.value.width,
    )
    const maxY = Math.max(
        sceneBounds.value.height,
        visibleWorldRect.value.y + visibleWorldRect.value.height,
    )

    return {
        minX,
        minY,
        width: maxX - minX,
        height: maxY - minY,
    }
})

const miniMapScale = computed(() => {
    const availableWidth = MINIMAP_WIDTH - MINIMAP_PADDING * 2
    const availableHeight = MINIMAP_HEIGHT - MINIMAP_PADDING * 2

    return Math.min(
        availableWidth / Math.max(navigationBounds.value.width, 1),
        availableHeight / Math.max(navigationBounds.value.height, 1),
    )
})

const miniMapScene = computed(() => {
    const scaledWidth = navigationBounds.value.width * miniMapScale.value
    const scaledHeight = navigationBounds.value.height * miniMapScale.value

    return {
        x: (MINIMAP_WIDTH - scaledWidth) / 2,
        y: (MINIMAP_HEIGHT - scaledHeight) / 2,
        width: scaledWidth,
        height: scaledHeight,
    }
})

const miniMapNodes = computed(() => {
    return nodes.value.map(node => ({
        id: node.id,
        x: miniMapScene.value.x + ((node.position.x + sceneBounds.value.offsetX) - navigationBounds.value.minX) * miniMapScale.value,
        y: miniMapScene.value.y + ((node.position.y + sceneBounds.value.offsetY) - navigationBounds.value.minY) * miniMapScale.value,
        width: NODE_WIDTH * miniMapScale.value,
        height: NODE_HEIGHT * miniMapScale.value,
        color: node.color,
    }))
})

const miniMapViewport = computed(() => {
    const visibleLocalX = visibleWorldRect.value.x
    const visibleLocalY = visibleWorldRect.value.y
    const visibleLocalWidth = visibleWorldRect.value.width
    const visibleLocalHeight = visibleWorldRect.value.height

    return {
        x: miniMapScene.value.x + (visibleLocalX - navigationBounds.value.minX) * miniMapScale.value,
        y: miniMapScene.value.y + (visibleLocalY - navigationBounds.value.minY) * miniMapScale.value,
        width: visibleLocalWidth * miniMapScale.value,
        height: visibleLocalHeight * miniMapScale.value,
        localX: visibleLocalX,
        localY: visibleLocalY,
    }
})

const activeConnectionPreviewPath = computed(() => {
    const activeConnectionDrag = connectionDragState.value

    if (!activeConnectionDrag) {
        return null
    }

    const sourceNode = nodeMap.value.get(activeConnectionDrag.sourceNodeId)

    if (!sourceNode) {
        return null
    }

    const startX = sourceNode.position.x + sceneBounds.value.offsetX + (
        activeConnectionDrag.currentWorldPoint.x >= sourceNode.position.x
            ? NODE_WIDTH
            : 0
    )
    const startY = sourceNode.position.y + sceneBounds.value.offsetY + NODE_HEIGHT / 2
    const endX = activeConnectionDrag.currentWorldPoint.x + sceneBounds.value.offsetX
    const endY = activeConnectionDrag.currentWorldPoint.y + sceneBounds.value.offsetY

    return buildCurvePath(startX, startY, endX, endY)
})

const connectionPaths = computed<WorkflowConnectionPath[]>(() => {
    return connections.value.flatMap((connection) => {
        const fromNode = nodeMap.value.get(connection.from)
        const toNode = nodeMap.value.get(connection.to)

        if (!fromNode || !toNode) {
            return []
        }

        return [{
            id: `${connection.from}-${connection.to}`,
            d: buildConnectionPath(
                fromNode.position.x + sceneBounds.value.offsetX,
                fromNode.position.y + sceneBounds.value.offsetY,
                toNode.position.x + sceneBounds.value.offsetX,
                toNode.position.y + sceneBounds.value.offsetY,
                NODE_WIDTH,
                NODE_HEIGHT,
            ),
        }]
    })
})

const miniMapConnectionPaths = computed<WorkflowConnectionPath[]>(() => {
    return connections.value.flatMap((connection) => {
        const fromNode = nodeMap.value.get(connection.from)
        const toNode = nodeMap.value.get(connection.to)

        if (!fromNode || !toNode) {
            return []
        }

        return [{
            id: `${connection.from}-${connection.to}`,
            d: buildConnectionPath(
                miniMapScene.value.x + ((fromNode.position.x + sceneBounds.value.offsetX) - navigationBounds.value.minX) * miniMapScale.value,
                miniMapScene.value.y + ((fromNode.position.y + sceneBounds.value.offsetY) - navigationBounds.value.minY) * miniMapScale.value,
                miniMapScene.value.x + ((toNode.position.x + sceneBounds.value.offsetX) - navigationBounds.value.minX) * miniMapScale.value,
                miniMapScene.value.y + ((toNode.position.y + sceneBounds.value.offsetY) - navigationBounds.value.minY) * miniMapScale.value,
                NODE_WIDTH * miniMapScale.value,
                NODE_HEIGHT * miniMapScale.value,
            ),
        }]
    })
})

const connectionDashArray = `${CONNECTION_DASH_LENGTH},${CONNECTION_DASH_GAP}`

const sceneTransformStyle = computed(() => {
    return {
        width: `${sceneBounds.value.width}px`,
        height: `${sceneBounds.value.height}px`,
        transform: `translate(${viewportPosition.value.x - sceneBounds.value.offsetX * zoomScale.value}px, ${viewportPosition.value.y - sceneBounds.value.offsetY * zoomScale.value}px) scale(${zoomScale.value})`,
    }
})

const canvasBackgroundStyle = computed(() => {
    const spacing = GRID_SPACING * zoomScale.value
    const offsetX = viewportPosition.value.x % spacing
    const offsetY = viewportPosition.value.y % spacing
    const dotSize = Math.max(0.35, Math.min(0.8, 0.8 * zoomScale.value))
    const transparentStop = Number((dotSize + 0.1).toFixed(2))

    return {
        backgroundColor: 'rgba(248, 250, 252, 0.92)',
        backgroundImage: `radial-gradient(circle, rgba(15, 23, 42, 0.28) ${dotSize}px, transparent ${transparentStop}px)`,
        backgroundPosition: `${offsetX}px ${offsetY}px`,
        backgroundSize: `${spacing}px ${spacing}px`,
    }
})

function getParentConnection(nodeId: string) {
    return connections.value.find(connection => connection.to === nodeId) ?? null
}

function getCanvasWorldPoint(clientX: number, clientY: number) {
    const canvas = canvasRef.value

    if (!canvas) {
        return { x: 0, y: 0 }
    }

    const rect = canvas.getBoundingClientRect()

    return {
        x: (clientX - rect.left - viewportPosition.value.x) / zoomScale.value,
        y: (clientY - rect.top - viewportPosition.value.y) / zoomScale.value,
    }
}

function getNodeIdFromPointer(clientX: number, clientY: number) {
    if (!import.meta.client) {
        return null
    }

    const element = document.elementFromPoint(clientX, clientY) as HTMLElement | null
    const nodeElement = element?.closest<HTMLElement>('[data-workflow-node-id]')

    return nodeElement?.dataset.workflowNodeId ?? null
}

function hasConnection(fromNodeId: string, toNodeId: string, ignoredTargetNodeId?: string) {
    return connections.value.some(connection => (
        connection.from === fromNodeId
        && connection.to === toNodeId
        && connection.to !== ignoredTargetNodeId
    ))
}

function getSiblingNodes(parentId: string) {
    return connections.value
        .filter(connection => connection.from === parentId)
        .map(connection => nodeMap.value.get(connection.to))
        .filter((node): node is WorkflowNode => Boolean(node))
}

function getChildNodes(nodeId: string) {
    return getSiblingNodes(nodeId)
}

function collectBranchNodeIds(nodeId: string) {
    const branchNodeIds = new Set<string>([nodeId])
    const queue = [nodeId]

    while (queue.length > 0) {
        const currentNodeId = queue.shift()

        if (!currentNodeId) {
            continue
        }

        const childConnections = connections.value.filter(connection => connection.from === currentNodeId)

        for (const connection of childConnections) {
            if (branchNodeIds.has(connection.to)) {
                continue
            }

            branchNodeIds.add(connection.to)
            queue.push(connection.to)
        }
    }

    return branchNodeIds
}

function startConnectionDrag(nodeId: string, mode: 'create' | 'retarget', event: PointerEvent) {
    if (event.button !== 0) {
        return
    }

    if (mode === 'retarget' && !getParentConnection(nodeId)) {
        return
    }

    event.preventDefault()
    event.stopPropagation()

    const worldPoint = getCanvasWorldPoint(event.clientX, event.clientY)
    const sourceNodeId = mode === 'retarget'
        ? getParentConnection(nodeId)?.from ?? nodeId
        : nodeId

    connectionDragState.value = {
        pointerId: event.pointerId,
        mode,
        sourceNodeId,
        originalTargetNodeId: mode === 'retarget' ? nodeId : undefined,
        currentWorldPoint: worldPoint,
    }

    const target = event.currentTarget as HTMLElement | null
    target?.setPointerCapture?.(event.pointerId)
}

async function copyWorkflowData() {
    const payload = JSON.stringify({
        nodes: nodes.value.map(node => ({
            id: node.id,
            type: node.type,
            title: node.title,
            description: node.description,
            color: node.color,
            position: node.position,
        })),
        connections: connections.value,
    }, null, 2)

    await copy(payload)
}

function clearWorkflow() {
    stopDragging()
    stopPanning()
    stopMiniMapDragging()

    nodes.value = []
    connections.value = []
}

function clampZoom(value: number) {
    return Math.min(MAX_ZOOM, Math.max(MIN_ZOOM, Number(value.toFixed(2))))
}

function getViewportCenter() {
    const canvas = canvasRef.value

    if (!canvas) {
        return null
    }

    const rect = canvas.getBoundingClientRect()

    return {
        clientX: rect.left + rect.width / 2,
        clientY: rect.top + rect.height / 2,
    }
}

function setZoom(nextZoom: number, focusPoint?: { clientX: number, clientY: number }) {
    const canvas = canvasRef.value
    const clampedZoom = clampZoom(nextZoom)

    if (clampedZoom === zoomScale.value) {
        return
    }

    if (!canvas || !focusPoint) {
        zoomScale.value = clampedZoom
        return
    }

    const rect = canvas.getBoundingClientRect()
    const offsetX = focusPoint.clientX - rect.left
    const offsetY = focusPoint.clientY - rect.top
    const worldX = (offsetX - viewportPosition.value.x) / zoomScale.value
    const worldY = (offsetY - viewportPosition.value.y) / zoomScale.value

    zoomScale.value = clampedZoom

    nextTick(() => {
        viewportPosition.value = {
            x: offsetX - worldX * clampedZoom,
            y: offsetY - worldY * clampedZoom,
        }
    })
}

function centerViewportAtScenePoint(localX: number, localY: number) {
    viewportPosition.value = {
        x: canvasWidth.value / 2 - localX * zoomScale.value,
        y: canvasHeight.value / 2 - localY * zoomScale.value,
    }
}

function setViewportTopLeft(localX: number, localY: number) {
    viewportPosition.value = {
        x: -localX * zoomScale.value,
        y: -localY * zoomScale.value,
    }
}

function centerSceneInViewport(nextZoom = zoomScale.value) {
    zoomScale.value = clampZoom(nextZoom)
    centerViewportAtScenePoint(
        sceneBounds.value.minX + sceneBounds.value.width / 2,
        sceneBounds.value.minY + sceneBounds.value.height / 2,
    )
}

function zoomIn() {
    setZoom(zoomScale.value + ZOOM_STEP, getViewportCenter() ?? undefined)
}

function zoomOut() {
    setZoom(zoomScale.value - ZOOM_STEP, getViewportCenter() ?? undefined)
}

function resetZoom() {
    centerSceneInViewport(1)
}

function handleCanvasWheel(event: WheelEvent) {
    event.preventDefault()

    if (!event.ctrlKey && !event.metaKey) {
        viewportPosition.value = {
            x: viewportPosition.value.x - event.deltaX,
            y: viewportPosition.value.y - event.deltaY,
        }
        return
    }

    event.preventDefault()

    const nextZoom = event.deltaY < 0
        ? zoomScale.value + ZOOM_STEP
        : zoomScale.value - ZOOM_STEP

    setZoom(nextZoom, {
        clientX: event.clientX,
        clientY: event.clientY,
    })
}

function updateViewportFromMiniMapEvent(event: PointerEvent) {
    const miniMap = miniMapRef.value

    if (!miniMap) {
        return
    }

    const rect = miniMap.getBoundingClientRect()
    const relativeX = event.clientX - rect.left
    const relativeY = event.clientY - rect.top
    const localX = (relativeX - miniMapScene.value.x) / miniMapScale.value + navigationBounds.value.minX
    const localY = (relativeY - miniMapScene.value.y) / miniMapScale.value + navigationBounds.value.minY

    if (miniMapDragState.value?.mode === 'viewport') {
        setViewportTopLeft(
            localX - miniMapDragState.value.grabOffset.x,
            localY - miniMapDragState.value.grabOffset.y,
        )
        return
    }

    centerViewportAtScenePoint(localX, localY)
}

function startMiniMapDrag(event: PointerEvent) {
    if (event.button !== 0) {
        return
    }

    event.preventDefault()
    event.stopPropagation()

    const target = event.currentTarget as HTMLElement | null
    target?.setPointerCapture?.(event.pointerId)

    const miniMap = miniMapRef.value

    if (!miniMap) {
        return
    }

    const rect = miniMap.getBoundingClientRect()
    const relativeX = event.clientX - rect.left
    const relativeY = event.clientY - rect.top
    const isInsideViewport = relativeX >= miniMapViewport.value.x
        && relativeX <= miniMapViewport.value.x + Math.max(miniMapViewport.value.width, 14)
        && relativeY >= miniMapViewport.value.y
        && relativeY <= miniMapViewport.value.y + Math.max(miniMapViewport.value.height, 14)

    miniMapDragState.value = {
        pointerId: event.pointerId,
        mode: isInsideViewport ? 'viewport' : 'jump',
        grabOffset: isInsideViewport
            ? {
                    x: (relativeX - miniMapViewport.value.x) / miniMapScale.value,
                    y: (relativeY - miniMapViewport.value.y) / miniMapScale.value,
                }
            : {
                    x: canvasWidth.value / zoomScale.value / 2,
                    y: canvasHeight.value / zoomScale.value / 2,
                },
    }

    updateViewportFromMiniMapEvent(event)
}

function startPanning(event: PointerEvent) {
    if (event.button !== 0) {
        return
    }

    const target = event.target as HTMLElement | null

    if (target?.closest('[data-workflow-node]')) {
        return
    }

    event.preventDefault()

    previousUserSelect.value = document.body.style.userSelect
    previousCanvasCursor.value = canvasRef.value?.style.cursor ?? ''
    document.body.style.userSelect = 'none'

    if (canvasRef.value) {
        canvasRef.value.style.cursor = 'grabbing'
        canvasRef.value.setPointerCapture?.(event.pointerId)
    }

    panState.value = {
        pointerId: event.pointerId,
        pointerStart: { x: event.clientX, y: event.clientY },
        viewportStart: { ...viewportPosition.value },
    }
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

    node.position.x = activeDrag.nodeStart.x + offsetX / zoomScale.value
    node.position.y = activeDrag.nodeStart.y + offsetY / zoomScale.value
}

function updatePanning(event: PointerEvent) {
    const activePan = panState.value

    if (!activePan || event.pointerId !== activePan.pointerId) {
        return
    }

    viewportPosition.value = {
        x: activePan.viewportStart.x + (event.clientX - activePan.pointerStart.x),
        y: activePan.viewportStart.y + (event.clientY - activePan.pointerStart.y),
    }
}

function updateMiniMapDragging(event: PointerEvent) {
    if (!miniMapDragState.value || event.pointerId !== miniMapDragState.value.pointerId) {
        return
    }

    updateViewportFromMiniMapEvent(event)
}

function updateConnectionDragging(event: PointerEvent) {
    if (!connectionDragState.value || event.pointerId !== connectionDragState.value.pointerId) {
        return
    }

    connectionDragState.value = {
        ...connectionDragState.value,
        currentWorldPoint: getCanvasWorldPoint(event.clientX, event.clientY),
    }
}

function stopDragging(pointerId?: number) {
    if (pointerId !== undefined && dragState.value && dragState.value.pointerId !== pointerId) {
        return
    }

    dragState.value = null
    document.body.style.userSelect = previousUserSelect.value
}

function stopPanning(pointerId?: number) {
    if (pointerId !== undefined && panState.value && panState.value.pointerId !== pointerId) {
        return
    }

    panState.value = null
    document.body.style.userSelect = previousUserSelect.value

    if (canvasRef.value) {
        canvasRef.value.style.cursor = previousCanvasCursor.value
    }
}

function stopMiniMapDragging(pointerId?: number) {
    if (pointerId !== undefined && miniMapDragState.value && miniMapDragState.value.pointerId !== pointerId) {
        return
    }

    miniMapDragState.value = null
}

function stopConnectionDragging(pointerId?: number, event?: PointerEvent) {
    const activeConnectionDrag = connectionDragState.value

    if (!activeConnectionDrag) {
        return
    }

    if (pointerId !== undefined && activeConnectionDrag.pointerId !== pointerId) {
        return
    }

    if (event) {
        const targetNodeId = getNodeIdFromPointer(event.clientX, event.clientY)

        if (
            targetNodeId
            && targetNodeId !== activeConnectionDrag.sourceNodeId
            && (
                activeConnectionDrag.mode !== 'retarget'
                || targetNodeId !== activeConnectionDrag.originalTargetNodeId
            )
            && !hasConnection(
                activeConnectionDrag.sourceNodeId,
                targetNodeId,
                activeConnectionDrag.mode === 'retarget'
                    ? activeConnectionDrag.originalTargetNodeId
                    : undefined,
            )
        ) {
            if (activeConnectionDrag.mode === 'create') {
                connections.value = [
                    ...connections.value,
                    { from: activeConnectionDrag.sourceNodeId, to: targetNodeId },
                ]
            }
            else if (activeConnectionDrag.originalTargetNodeId) {
                connections.value = connections.value.map(connection => (
                    connection.from === activeConnectionDrag.sourceNodeId
                    && connection.to === activeConnectionDrag.originalTargetNodeId
                        ? { ...connection, to: targetNodeId }
                        : connection
                ))
            }
        }
    }

    connectionDragState.value = null
}

async function addNode() {
    const lastNode = nodes.value.at(-1)

    if (!lastNode) {
        const template = nodeTemplates[Math.floor(Math.random() * nodeTemplates.length)]!
        const position = { x: 50, y: 100 }

        nodes.value = [
            ...nodes.value,
            {
                id: `node-${Date.now()}`,
                ...template,
                position,
            },
        ]

        await scrollToNode(position)
        return
    }

    await addChildNode(lastNode.id)
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
        y: maxSiblingY + NODE_HEIGHT + NODE_VERTICAL_GAP,
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

async function addChildNode(nodeId: string) {
    const parentNode = nodeMap.value.get(nodeId)

    if (!parentNode) {
        return
    }

    const template = nodeTemplates[Math.floor(Math.random() * nodeTemplates.length)]!
    const childNodes = getChildNodes(nodeId)
    const position = childNodes.length === 0
        ? {
                x: parentNode.position.x + NODE_WIDTH + NODE_HORIZONTAL_GAP,
                y: parentNode.position.y,
            }
        : {
                x: Math.max(parentNode.position.x + NODE_WIDTH + NODE_HORIZONTAL_GAP, ...childNodes.map(node => node.position.x)),
                y: Math.max(...childNodes.map(node => node.position.y)) + NODE_HEIGHT + NODE_VERTICAL_GAP,
            }

    const newNode: WorkflowNode = {
        id: `node-${Date.now()}`,
        ...template,
        position,
    }

    nodes.value = [...nodes.value, newNode]
    connections.value = [
        ...connections.value,
        { from: nodeId, to: newNode.id },
    ]

    await scrollToNode(position)
}

function removeNode(nodeId: string) {
    const branchNodeIds = collectBranchNodeIds(nodeId)

    if (draggingNodeId.value && branchNodeIds.has(draggingNodeId.value)) {
        stopDragging()
    }

    nodes.value = nodes.value.filter(node => !branchNodeIds.has(node.id))
    connections.value = connections.value.filter((connection) => {
        return !branchNodeIds.has(connection.from) && !branchNodeIds.has(connection.to)
    })
}

async function scrollToNode(position: Position) {
    await nextTick()

    const canvas = canvasRef.value

    if (!canvas) {
        return
    }

    const margin = 100
    const screenLeft = viewportPosition.value.x + position.x * zoomScale.value
    const screenTop = viewportPosition.value.y + position.y * zoomScale.value
    const screenRight = screenLeft + NODE_WIDTH * zoomScale.value
    const screenBottom = screenTop + NODE_HEIGHT * zoomScale.value

    let nextViewportX = viewportPosition.value.x
    let nextViewportY = viewportPosition.value.y

    if (screenLeft < margin) {
        nextViewportX += margin - screenLeft
    }
    else if (screenRight > canvas.clientWidth - margin) {
        nextViewportX -= screenRight - (canvas.clientWidth - margin)
    }

    if (screenTop < margin) {
        nextViewportY += margin - screenTop
    }
    else if (screenBottom > canvas.clientHeight - margin) {
        nextViewportY -= screenBottom - (canvas.clientHeight - margin)
    }

    viewportPosition.value = {
        x: nextViewportX,
        y: nextViewportY,
    }
}

function getNodeStyle(position: Position) {
    return {
        left: `${position.x + sceneBounds.value.offsetX}px`,
        top: `${position.y + sceneBounds.value.offsetY}px`,
        width: `${NODE_WIDTH}px`,
        height: `${NODE_HEIGHT}px`,
        transformOrigin: '0 0',
    }
}

if (import.meta.client) {
    useEventListener(window, 'pointermove', updateDragging)
    useEventListener(window, 'pointermove', updatePanning)
    useEventListener(window, 'pointermove', updateMiniMapDragging)
    useEventListener(window, 'pointermove', updateConnectionDragging)
    useEventListener(window, 'pointerup', event => stopDragging(event.pointerId))
    useEventListener(window, 'pointerup', event => stopPanning(event.pointerId))
    useEventListener(window, 'pointerup', event => stopMiniMapDragging(event.pointerId))
    useEventListener(window, 'pointerup', event => stopConnectionDragging(event.pointerId, event))
    useEventListener(window, 'pointercancel', event => stopDragging(event.pointerId))
    useEventListener(window, 'pointercancel', event => stopPanning(event.pointerId))
    useEventListener(window, 'pointercancel', event => stopMiniMapDragging(event.pointerId))
    useEventListener(window, 'pointercancel', event => stopConnectionDragging(event.pointerId))
    useEventListener(canvasRef, 'wheel', handleCanvasWheel, { passive: false })
}

watch([canvasWidth, canvasHeight, sceneBounds], () => {
    if (hasAutoCenteredViewport.value) {
        return
    }

    if (canvasWidth.value <= 0 || canvasHeight.value <= 0) {
        return
    }

    centerSceneInViewport(1)
    hasAutoCenteredViewport.value = true
}, { flush: 'post' })

onUnmounted(() => {
    stopDragging()
    stopPanning()
    stopMiniMapDragging()
    stopConnectionDragging()
})
</script>

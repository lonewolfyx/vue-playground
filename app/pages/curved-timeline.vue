<template>
    <div class="min-h-screen bg-[radial-gradient(circle_at_top,_rgba(255,92,135,0.08),_transparent_28%),linear-gradient(180deg,_#ffffff_0%,_#fff8fb_42%,_#ffffff_100%)] px-4 py-10 sm:px-6 lg:px-8">
        <div class="mx-auto max-w-6xl">
            <Card class="overflow-hidden border-zinc-200/70 bg-white/88 shadow-[0_32px_120px_-48px_rgba(17,24,39,0.25)] backdrop-blur-sm">
                <CardHeader class="gap-4 border-b border-zinc-100/80 pb-6">
                    <div class="flex flex-wrap gap-2">
                        <Badge variant="outline">
                            Vue 3
                        </Badge>
                        <Badge variant="outline">
                            SVG Path
                        </Badge>
                        <Badge variant="outline">
                            shadcn/ui-vue
                        </Badge>
                    </div>

                    <div class="space-y-2">
                        <CardTitle class="text-3xl font-semibold tracking-[-0.04em] text-zinc-950 sm:text-4xl">
                            Animated Curved Timeline
                        </CardTitle>
                        <CardDescription class="max-w-3xl text-sm leading-6 text-zinc-500 sm:text-base">
                            按照附件与 Morphin 的 React 版本思路，做了一个 Vue 3 + SVG 的可复用时间线组件。
                            预览页保留了分组标题、弯曲连线、当前节点高亮和淡化的历史轨迹层次。
                        </CardDescription>
                    </div>

                    <a
                        class="inline-flex w-fit text-sm font-medium text-zinc-600 underline decoration-zinc-300 underline-offset-4 transition-colors hover:text-zinc-950"
                        href="https://morphin.dev/components/animated-curved-timeline-with-dynamic-svg-path"
                        rel="noreferrer"
                        target="_blank"
                    >
                        查看 React 参考实现
                    </a>
                </CardHeader>

                <CardContent class="p-0">
                    <div class="max-h-[80vh] overflow-y-auto px-4 py-6 sm:px-6 md:px-8 lg:px-10">
                        <AnimatedCurvedTimeline :groups="timelineGroups" />
                    </div>
                </CardContent>
            </Card>
        </div>
    </div>
</template>

<script lang="ts" setup>
useHead({
    title: 'Animated Curved Timeline',
})

const timelineGroups = [
    {
        id: 'today',
        label: 'Today',
        items: [
            {
                id: 'meal-plan',
                title: 'Asked for a high-protein meal plan',
            },
            {
                id: 'dashboard-ux',
                title: 'Worked on the b402 dashboard UX',
            },
            {
                id: 'side-projects',
                title: 'Brainstormed side projects',
            },
        ],
    },
    {
        id: 'yesterday',
        label: 'Yesterday',
        items: [
            {
                id: 'etf-strategy',
                title: 'Researched ETF investing strategies',
            },
            {
                id: 'client-email',
                title: 'Drafted a polite client email',
            },
            {
                id: 'sleep-quality',
                title: 'Asked about improving sleep quality',
            },
            {
                id: 'landing-tagline',
                title: 'Generated tagline ideas for a landing page',
                current: true,
            },
        ],
    },
    {
        id: 'feb-8-2026',
        label: 'Feb 8, 2026',
        items: [
            {
                id: 'typescript-generic',
                title: 'Explained a TypeScript generic error',
            },
            {
                id: 'linkedin-summary',
                title: 'Helped rewrite a LinkedIn profile summary',
            },
            {
                id: 'next-remix',
                title: 'Compared Next.js vs Remix',
            },
            {
                id: 'react-hook',
                title: 'Built a custom React hook',
            },
        ],
    },
]
</script>

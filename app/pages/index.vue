<template>
    <!--    <div class="relative w-full h-full flex justify-center items-center"> -->
    <!--        <div class="rounded-xl border bg-card xl:w-xl"> -->
    <!--            <div class="flex items-center justify-between gap-3 px-4 py-4 border-b"> -->
    <!--                <h2 class="text-sm font-medium text-pretty sm:text-base"> -->
    <!--                    Vue Playground Demo -->
    <!--                </h2> -->
    <!--            </div> -->
    <!--            <ScrollArea class="relative h-96 px-4 pt-2 pb-4 text-xs sm:px-6 sm:text-sm"> -->
    <!--                <div class="divide-y"> -->
    <!--                    <NuxtLink -->
    <!--                        v-for="play in playground" -->
    <!--                        :key="play.label" -->
    <!--                        class="flex flex-col py-3" -->
    <!--                        :to="`/${play.label}`" -->
    <!--                        target="_blank" -->
    <!--                    > -->
    <!--                        <span class="truncate font-medium text-foreground capitalize">{{ play.label }}</span> -->
    <!--                        <span class="text-[10px] break-words text-muted-foreground sm:text-xs">Source: {{ play.source }}</span> -->
    <!--                    </NuxtLink> -->
    <!--                </div> -->
    <!--            </ScrollArea> -->
    <!--        </div> -->
    <!--    </div> -->
    <div class="min-h-svh flex flex-col bg-background">
        <div class="flex-1 flex overflow-hidden">
            <main class="flex-1 w-full h-full">
                <div
                    class="min-h-screen bg-white dark:bg-[#111] text-zinc-900 dark:text-zinc-100 font-sans overflow-x-hidden"
                >
                    <main class="max-w-[1400px] mx-auto pt-32 pb-32 px-6 sm:px-8 relative z-10">
                        <div class="mb-12 max-w-3xl">
                            <h1 class="text-4xl lg:text-5xl font-bold tracking-tighter bg-gradient-to-br from-zinc-900 via-zinc-500 to-zinc-900 dark:from-white dark:via-zinc-400 dark:to-white bg-clip-text text-transparent leading-[1.1] mb-2 inline-block">
                                Vue Playground
                            </h1>
                            <p class="text-lg text-zinc-500 dark:text-zinc-400 max-w-xl leading-relaxed">
                                Constantly improve the playing field for vue components
                            </p>
                        </div>

                        <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                            <div
                                v-for="demo in playground"
                                :key="demo.label"
                            >
                                <NuxtLink
                                    :class="cn(
                                        'group relative flex flex-col rounded-2xl',
                                        'border border-zinc-200/60 dark:border-zinc-800/60',
                                        'bg-white dark:bg-zinc-900/50',
                                        'overflow-hidden transition-all duration-300',
                                        'hover:shadow-[0_8px_30px_rgb(0,0,0,0.04)] dark:hover:shadow-[0_8px_30px_rgb(255,255,255,0.02)]',
                                        'hover:border-zinc-300 dark:hover:border-zinc-700',
                                    )"
                                    :to="demo.label"
                                    target="_blank"
                                >
                                    <div class="p-1.5">
                                        <div
                                            class="relative h-[220px] w-full rounded-xl bg-zinc-50 dark:bg-zinc-900/80 group-hover:bg-zinc-100/50 dark:group-hover:bg-zinc-800/80 transition-colors border border-dashed border-zinc-200/50 dark:border-zinc-800/50 overflow-hidden"
                                        >
                                            <img
                                                :alt="demo.label"
                                                class="w-full h-full object-contain"
                                                :src="`/${demo.label}.png`"
                                            >
                                        </div>
                                    </div>
                                    <div class="flex flex-col gap-1.5 px-4 pb-4 pt-2">
                                        <div class="flex items-center justify-between">
                                            <h3
                                                :class="cn(
                                                    'text-sm font-medium',
                                                    'text-zinc-900 dark:text-zinc-100',
                                                    'group-hover:text-zinc-700 dark:group-hover:text-zinc-300',
                                                    'transition-colors capitalize',
                                                )"
                                            >
                                                {{ demo.label }}
                                            </h3>
                                            <Tooltip>
                                                <TooltipTrigger>
                                                    <Badge variant="secondary">
                                                        Source
                                                    </Badge>
                                                </TooltipTrigger>
                                                <TooltipContent>
                                                    <p>{{ demo.source }}</p>
                                                </TooltipContent>
                                            </Tooltip>
                                        </div>
                                        <!--                                        <p class="text-xs text-zinc-500 dark:text-zinc-400 line-clamp-1" /> -->
                                    </div>
                                </NuxtLink>
                            </div>
                        </div>
                    </main>
                </div>
            </main>
        </div>
    </div>
</template>

<script lang="ts" setup>
import { cn } from '~/lib/utils'

const playground = [
    {
        label: 'circle-packing',
        source: 'https://observablehq.com/@d3/bubble-chart/2',
    },
    {
        label: 'particle',
        source: 'https://wereorbit.com',
    },
    {
        label: 'contribution-graph',
        source: 'https://github.com',
    },
]

// useHead({
//     htmlAttrs: {
//         class: 'relative w-full h-full',
//     },
//     bodyAttrs: {
//         class: 'relative w-full h-full',
//     },
// })
</script>

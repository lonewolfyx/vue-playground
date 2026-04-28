<template>
    <div class="relative w-full h-full flex justify-center items-center">
        <div class="rounded-xl border bg-card xl:w-xl">
            <div class="flex items-center justify-between gap-3 px-4 py-4 border-b">
                <h2 class="text-sm font-medium text-pretty sm:text-base">
                    Vue Playground Demo
                </h2>
            </div>
            <ScrollArea class="relative h-96 px-4 pt-2 pb-4 text-xs sm:px-6 sm:text-sm">
                <div class="divide-y">
                    <NuxtLink
                        v-for="play in playground"
                        :key="play.label"
                        class="flex flex-col py-3"
                        :to="`/${play.label}`"
                        target="_blank"
                    >
                        <span class="truncate font-medium text-foreground capitalize">{{ play.label }}</span>
                        <span class="text-[10px] break-words text-muted-foreground sm:text-xs">Source: {{ play.source }}</span>
                    </NuxtLink>
                </div>
            </ScrollArea>
        </div>
    </div>
</template>

<script lang="ts" setup>
import { ScrollArea } from '~/components/ui/scroll-area'

const playground = [
    {
        label: 'circle-packing',
        source: 'https://observablehq.com/@d3/bubble-chart/2',
    },
    {
        label: 'particle',
        source: 'https://wereorbit.com',
    },
]

useHead({
    htmlAttrs: {
        class: 'relative w-full h-full',
    },
    bodyAttrs: {
        class: 'relative w-full h-full',
    },
})
</script>

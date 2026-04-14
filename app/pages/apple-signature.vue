<template>
    <div>
        <Motion
            :key="paths.length"
            :animate="inView ? undefined : 'visible'"
            :height="height"
            :viewBox="`0 0 ${width} ${height}`"
            :viewport="once"
            :while-in-view="inView ? 'visible' : undefined"
            :width="width"
            as="svg"
            fill="none"
            initial="hidden"
        >
            <!--            <defs> -->
            <!--                <mask :id="maskId" maskUnits="userSpaceOnUse"> -->
            <!--                    <Motion -->
            <!--                        v-for="(d, i) in paths" -->
            <!--                        :key="i" -->
            <!--                        :d="d" -->
            <!--                        :stroke-width="fontSize * 0.22" -->
            <!--                        :transition="{ -->
            <!--                            pathLength: { -->
            <!--                                delay: delay + i * 0.2, -->
            <!--                                duration, -->
            <!--                                ease: 'easeInOut', -->
            <!--                            }, -->
            <!--                            opacity: { -->
            <!--                                delay: delay + i * 0.2 + 0.01, -->
            <!--                                duration: 0.01, -->
            <!--                            }, -->
            <!--                        }" -->
            <!--                        :variants="variants" -->
            <!--                        as="path" -->
            <!--                        fill="none" -->
            <!--                        stroke="white" -->
            <!--                        stroke-linecap="round" -->
            <!--                        stroke-linejoin="round" -->
            <!--                        vector-effect="non-scaling-stroke" -->
            <!--                    /> -->
            <!--                </mask> -->
            <!--            </defs> -->

            <Motion
                v-for="(d, i) in paths"
                :key="i"
                :d="d"
                :stroke="color"
                :transition="{
                    pathLength: {
                        delay: delay + i * 0.2,
                        duration,
                        ease: 'easeInOut',
                    },
                    opacity: {
                        delay: delay + i * 0.2 + 0.01,
                        duration: 0.01,
                    },
                }"
                :variants="variants"
                as="path"
                fill="none"
                stroke-linecap="butt"
                stroke-linejoin="round"
                stroke-width="1"
                vector-effect="non-scaling-stroke"
            />
            <!--            <g mask="{`url(#${maskId})`}"> -->
            <!--                <path -->
            <!--                    v-for="(d, i) in paths" -->
            <!--                    :key="i" -->
            <!--                    :d="d" -->
            <!--                    :fill="color" -->
            <!--                /> -->
            <!--            </g> -->
        </Motion>
    </div>
</template>

<script lang="ts" setup>
import { Motion } from 'motion-v'
import opentype from 'opentype.js'

const text = 'hello'
const color = '#1D1D1F'
const duration = 2
const delay = 0
const inView = false
const once = true
const fontSize = 35
const height = 100
const width = ref(300)
const paths = ref([])

const horizontalPadding = fontSize * 0.1
const topMargin = Math.max(5, (height - fontSize) / 2)
const baseline = Math.min(height - 5, topMargin + fontSize)
const maskId = `signature-reveal-${useId().replace(/:/g, '')}`

const variants = {
    hidden: { pathLength: 0, opacity: 0 },
    visible: { pathLength: 1, opacity: 1 },
}

onMounted(async () => {
    const font = await opentype.load('/Bumbbled.otf')
    let x = horizontalPadding
    const newPaths: string[] = []

    for (const char of text) {
        const glyph = font.charToGlyph(char)
        const path = glyph.getPath(x, baseline, fontSize)
        newPaths.push(path.toPathData(3))

        const advanceWidth = glyph.advanceWidth ?? font.unitsPerEm
        x += advanceWidth * (fontSize / font.unitsPerEm)
    }

    paths.value = newPaths
    console.log(newPaths)
    console.log(x + horizontalPadding)
})
</script>

<style scoped>

</style>

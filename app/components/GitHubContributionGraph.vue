<template>
    <div v-if="status === 'pending'" class="flex items-center justify-center py-8">
        <span class="text-sm text-muted-foreground">Loading...</span>
    </div>
    <div v-else-if="error" class="flex items-center justify-center py-8">
        <span class="text-sm text-destructive">Failed to load contributions</span>
    </div>
    <ContributionGraph
        v-else-if="contributions"
        :data="contributions"
        :color-theme="colorTheme"
    />
</template>

<script lang="ts" setup>
import type { ContributionDay, ThemeName } from './ContributionGraph.vue'

interface Props {
    username: string
    colorTheme?: ThemeName
}

const props = defineProps<Props>()

interface ApiResponse {
    total: { lastYear: number }
    contributions: ContributionDay[]
}

const { data, error, status } = useFetch<ApiResponse>(
    () => `https://github-contributions-api.jogruber.de/v4/${props.username}`,
    { query: { y: 'last' } },
)

const contributions = computed(() => data.value?.contributions)
</script>

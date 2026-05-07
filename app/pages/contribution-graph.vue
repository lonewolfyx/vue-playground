<template>
    <div class="min-h-screen bg-background p-8">
        <div class="mx-auto max-w-4xl space-y-8">
            <div>
                <h1 class="text-2xl font-bold text-foreground">
                    GitHub Contribution Graph
                </h1>
                <p class="mt-1 text-sm text-muted-foreground">
                    Hover over cells to see contribution details
                </p>
            </div>

            <!-- GitHub API (default theme) -->
            <div class="rounded-xl border bg-card p-6">
                <h2 class="mb-4 text-sm font-medium text-foreground">
                    {{ username }} (GitHub green)
                </h2>
                <GitHubContributionGraph :username="username" />
            </div>

            <!-- GitHub API (custom theme) -->
            <!--            <div class="rounded-xl border bg-card p-6"> -->
            <!--                <h2 class="mb-4 text-sm font-medium text-foreground"> -->
            <!--                    {{username}} (blue) -->
            <!--                </h2> -->
            <!--                <GitHubContributionGraph :username="username" color-theme="blue" /> -->
            <!--            </div> -->

            <!--            &lt;!&ndash; GitHub API (purple) &ndash;&gt; -->
            <!--            <div class="rounded-xl border bg-card p-6"> -->
            <!--                <h2 class="mb-4 text-sm font-medium text-foreground"> -->
            <!--                    {{username}} (purple) -->
            <!--                </h2> -->
            <!--                <GitHubContributionGraph :username="username" color-theme="purple" /> -->
            <!--            </div> -->

            <!-- Resizable container -->
            <div class="rounded-xl border bg-card p-6">
                <h2 class="mb-4 text-sm font-medium text-foreground">
                    Resizable (drag the bottom-right corner)
                </h2>
                <div
                    class="overflow-hidden rounded-lg border p-4"
                    style="resize: horizontal; min-width: 300px; max-width: 100%;"
                >
                    <GitHubContributionGraph :username="username" color-theme="gray" />
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
const theme = ['default']
// const username = 'torvalds'
const username = 'lonewolfyx'
</script>

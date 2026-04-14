<template>
    <div style="max-width: 900px; margin: 0 auto; padding: 24px;">
        <BubblePack
            :data="data"
            :size="900"
            :min-radius="20"
            :max-radius="150"
            @select="handleSelect"
        />
    </div>
</template>

<script setup lang="ts">
const data = Array.from({ length: 2000 }, (_, index) => {
    const id = index + 1

    return {
        name: `user-${id}`,
        avatar: `https://picsum.photos/seed/avatar-${id}/120/120`,
        count:
            index < 20
                ? Math.floor(Math.random() * 5000) + 3000
                : index < 200
                    ? Math.floor(Math.random() * 1200) + 300
                    : Math.floor(Math.random() * 120) + 10,

    }
})

function handleSelect(node: unknown) {
    console.log('click:', node)
}
</script>

<style>
body{
    background: aquamarine;
}
</style>

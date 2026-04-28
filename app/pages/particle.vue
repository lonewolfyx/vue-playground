<template>
    <div class="relative w-full h-screen flex justify-center items-center">
        <div class="relative h-[420px] md:h-[560px]">
            <ImageParticleEffect
                src="/rocket.png"
                :sample-step="3"
                :threshold="36"
                :dot-size="1.2"
                :render-scale="1"
                align="center"
                :mouse-force="95"
                :dense-particles="false"
            />
        </div>
    </div>
</template>

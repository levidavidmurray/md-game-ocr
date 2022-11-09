<template>
    <div class="p-6 mx-auto max-w-screen-sm">
        <h1 class="text-center">Test</h1>
        <n-divider />

        <div class="space-x-4 mb-8">
            <n-button @click="() => doCamera()">{{ enabled ? 'Stop' : 'Start' }}</n-button>
            <n-button v-if="enabled" @click="doCapture">Capture</n-button>
            <n-button v-else @click="() => doCamera(true)">Reset</n-button>
            <n-button @click="doMessage">Message</n-button>
        </div>

        <video v-if="enabled" ref="videoRef" autoplay muted></video>
        <img v-if="imgSrc" :src="imgSrc">
        <canvas ref="canvasRef" class="hidden"></canvas>

    </div>
</template>

<script lang="ts" setup>
import { NDivider, NButton } from 'naive-ui'
import { io } from 'socket.io-client'

const updateFps = 5

const socket = io("http://localhost:5000")

socket.on('connect', () => {
    console.log('connected', socket.connected)
})

// @ts-ignore
window.data = []
socket.on('md_detect', (data) => {
    console.log('md_detect', data)
    const { b64_image: arrayBuffer } = data
    const b64Img = 'data:image/jpeg;base64,' + arrayBuffer
    imgSrc.value = b64Img
})

const videoRef = ref<HTMLVideoElement>(null)
const canvasRef = ref<HTMLCanvasElement>(null)
const imgSrc = ref<string>()

const constraints = {
    width: { min: 480, ideal: 1080 },
    height: { min: 480, ideal: 1080 },
    aspectRatio: 1,
};

const { stream, start, stop, restart, enabled } = useUserMedia({ audioDeviceId: false })

watchEffect(() => {
    console.log(videoRef.value, stream.value?.id)
    if (videoRef.value && stream.value) {
        const videoTrack = stream.value.getVideoTracks()[0]
        const capabilities = videoTrack.getCapabilities()
        const { max: maxFrameRate } = capabilities.frameRate
        videoTrack?.applyConstraints?.({ ...constraints, frameRate: { min: 15, max: maxFrameRate, ideal: maxFrameRate } })
        videoRef.value.srcObject = stream.value
    }

    // @ts-ignore
    window.stream = stream.value
})

const getCurrentVideoFrame = () => {
    const { value: video } = videoRef
    const { value: canvas } = canvasRef

    canvas.width = video.clientWidth
    canvas.height = video.clientHeight
    const context = canvas.getContext('2d')

    context.drawImage(video, 0, 0, canvas.width, canvas.height)
    return canvas.toDataURL('image/jpeg')
}

// @ts-ignore
window.getFrame = getCurrentVideoFrame
// @ts-ignore
window.socket = socket

const doMessage = () => {
    socket.emit('message', 1)
}

const doCamera = (overrideFlag?: boolean) => {
    if (overrideFlag !== undefined) {
        enabled.value = !overrideFlag
    }

    enabled.value = !enabled.value

    if (enabled.value) {
        imgSrc.value = null
    }
}

const doCapture = () => {
    const { value: video } = videoRef
    const { value: canvas } = canvasRef

    canvas.width = video.videoWidth
    canvas.height = video.videoHeight
    const context = canvas.getContext('2d')

    context.drawImage(video, 0, 0, canvas.width, canvas.height)
    imgSrc.value = canvas.toDataURL('image/jpeg')

    doCamera(false)
}

onMounted(() => {
    enabled.value = true
})

</script>

<style>
video {
    width: 100%;
    background: black;
}

body {
    /* Light */
    @apply bg-neutral-50 text-neutral-900;

    /* Dark */
    @apply dark: bg-neutral-900 dark:text-neutral-100;
}

h1 {
    @apply text-4xl font-extrabold;
}

h2 {
    @apply text-xl font-semibold;
}
</style>

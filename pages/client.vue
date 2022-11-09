<template>
    <div class="p-6 mx-auto max-w-screen-sm">
        <h1 class="text-center">Test</h1>
        <n-divider />

        <div class="space-x-4 mb-8">
            <n-button @click="() => doCamera()">{{ enabled ? 'Stop' : 'Start' }}</n-button>
            <n-button v-if="enabled" @click="doCapture">Capture</n-button>
            <n-button v-else @click="() => doCamera(true)">Reset</n-button>
            <n-button @click="predictWebcam">Predict</n-button>
        </div>

        <div ref="liveView">
            <video v-show="enabled" ref="videoRef" autoplay muted></video>
        </div>

        <img v-if="imgSrc" :src="imgSrc">
        <canvas ref="canvasRef" class="hidden"></canvas>

    </div>
</template>

<script lang="ts" setup>
import { NDivider, NButton } from 'naive-ui'
import * as tf from '@tensorflow/tfjs'
import { GraphModel } from '@tensorflow/tfjs';

const videoRef = ref<HTMLVideoElement>(null)
const canvasRef = ref<HTMLCanvasElement>(null)
const liveViewRef = ref<HTMLDivElement>(null)
const imgSrc = ref<string>()

const constraints = {
    width: { min: 480, ideal: 1080 },
    height: { min: 480, ideal: 1080 },
    aspectRatio: 1,
};

const { stream, start, stop, restart, enabled } = useUserMedia({ audioDeviceId: false })

let model: GraphModel | undefined;

const loadModel = async () => {
    model = await tf.loadGraphModel('/tfjs_model/model.json')
    console.log('Model Loaded')
}

const children = []

const predictWebcam = async () => {
    await detectTF(videoRef.value)
    window.requestAnimationFrame(predictWebcam)
}

const imageSize = 416
const classProbThreshold = 0.4

const detectTF = async (imgToPredict: HTMLVideoElement) => {
    // Create tensor from image
    const tfImg = tf.browser.fromPixels(imgToPredict)
    // Create smaller image which fits detection size
    const smallImg = tf.image.resizeBilinear(tfImg, [imageSize, imageSize]).toFloat()

    // const resized = tf.cast(smallImg, 'int32')
    // const tf4d_ = tf.tensor4d(Array.from(resized.dataSync()), [1, imageSize, imageSize, 3])
    // const tf4d = tf.cast(tf4d_, 'float32')
    const expanded = smallImg.transpose([0,1,2]).expandDims()

    const predictions = await model.executeAsync(expanded)

    renderPredictionBoxes(predictions[4].dataSync(), predictions[1].dataSync(), predictions[2].dataSync())

    // Dispose of tensors
    tfImg.dispose()
    smallImg.dispose()
    expanded.dispose()
    // smallImg.dispose()
    // resized.dispose()
    // tf4d.dispose()
}

const renderPredictionBoxes = (predBoxes, predClasses, predScores) => {
    const { value: liveView } = liveViewRef
    const { x: xStart, y: yStart, height: vidHeight, width: vidWidth } = videoRef.value.getBoundingClientRect()

    //Remove all detections:
    for (let i = 0; i < children.length; i++) {
        liveView.removeChild(children[i]);
    }
    children.splice(0);
    //Loop through predictions and draw them to the live view if they have a high confidence score.
    for (let i = 0; i < 99; i++) {
        //If we are over 66% sure we are sure we classified it right, draw it!
        const minY = Number((predBoxes[i * 4] * vidHeight+yStart).toFixed(0));
        const minX = Number((predBoxes[i * 4 + 1] * vidWidth+xStart).toFixed(0));
        const maxY = Number((predBoxes[i * 4 + 2] * vidHeight+yStart).toFixed(0));
        const maxX = Number((predBoxes[i * 4 + 3] * vidWidth+xStart).toFixed(0));
        const score = predScores[i * 3] * 100;
        const width_ = (maxX-minX).toFixed(0);
        const height_ = (maxY-minY).toFixed(0);
        //If confidence is above 70%
        if (score > 70 && score < 100){
            const highlighter: HTMLDivElement = document.createElement('div');
            highlighter.setAttribute('class', 'highlighter');
            highlighter.style.left = `${minX}px`
            highlighter.style.top = `${minY}px`
            highlighter.style.width = `${width_}px`
            highlighter.style.height = `${height_}px`
            highlighter.innerHTML = '<p>'+Math.round(score) + '% ' + 'Your Object Name'+'</p>';
            liveView.appendChild(highlighter);
            children.push(highlighter);
        }
    }
}

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

onMounted(async () => {
    await loadModel()
    enabled.value = true
})

</script>

<style>
video {
    width: 100%;
    background: black;
}

.highlighter p {
    padding: 5px;
    z-index: 2;
    font-size: 30px;
    color: white;
    position: absolute;
}
.highlighter {
    border: 2px solid white;
    box-shadow: 0 0 50px 0 white;
    z-index: 1;
    position: fixed;
    display: grid;
    place-content: center;
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

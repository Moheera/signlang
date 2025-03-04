<script lang="ts">
import { onMount } from 'svelte';
import * as tf from '@tensorflow/tfjs';
import * as handpose from '@tensorflow-models/handpose';
import '@tensorflow/tfjs-backend-webgl';
import cameraUtils from '@mediapipe/camera_utils';

const { Camera } = cameraUtils;

let video: any | null;
let model: handpose.HandPose;
let predictions: any[] = [];
let detectedGesture = "";

onMount(async () => {
    model = await handpose.load();
    await setupCamera();
});

async function setupCamera() {
    video = document.getElementById('video');
    const stream = await navigator.mediaDevices.getUserMedia({ video: true });
    video.srcObject = stream;
    
    return new Promise<void>((resolve) => {
        video.onloadedmetadata = () => {
            video.play();
            detectHands();
            resolve();
        };
    });
}

async function detectHands() {
    if (!model || !video.videoWidth || !video.videoHeight) return;
    const predictionsArray = await model.estimateHands(video);
    predictions = predictionsArray;
    
    if (predictionsArray.length > 0) {
        detectedGesture = recognizeGesture(predictionsArray[0].landmarks);
    } else {
        detectedGesture = "No hand detected";
    }
    
    requestAnimationFrame(detectHands);
}

function recognizeGesture(landmarks: any | any[]) {
    if (!landmarks || landmarks.length < 21) return "Unknown";
    
    const fingers = getFingerStates(landmarks);
    
    if (fingers.index && fingers.middle && fingers.ring && fingers.pinky && fingers.thumb) {
        return "Open Hand";
    }
    if (fingers.thumb && !fingers.index && !fingers.middle && !fingers.ring && !fingers.pinky) {
        return "Thumbs Up";
    }
    if (!fingers.thumb && !fingers.index && !fingers.middle && !fingers.ring && !fingers.pinky) {
        return "Fist";
    }
    if (fingers.index && !fingers.middle && !fingers.ring && !fingers.pinky) {
        return "Pointing Up";
    }
    if (fingers.index && fingers.middle && !fingers.ring && !fingers.pinky) {
        return "Peace Sign";
    }
    return "Unknown";
}

function getFingerStates(landmarks: number[][]) {
    function isExtended(tipIndex: number, pipIndex: number, baseIndex: number) {
        return landmarks[tipIndex][1] < landmarks[pipIndex][1] && landmarks[pipIndex][1] < landmarks[baseIndex][1];
    }
    
    return {
        thumb: landmarks[4][0] > landmarks[3][0], // Thumb extended if it's to the right of its base
        index: isExtended(8, 6, 5),
        middle: isExtended(12, 10, 9),
        ring: isExtended(16, 14, 13),
        pinky: isExtended(20, 18, 17),
    };
}
</script>

<main class="flex flex-col gap-4 items-center justify-center h-screen bg-gray-900 text-white">
    <h1 class="text-2xl font-bold">Hand Gesture Recognition</h1>
    <p>Try open hand, thumps up, fist, pointing up, or peace sign.</p>
    <!-- svelte-ignore a11y_media_has_caption -->
    <video id="video" class="w-96 h-72 border border-gray-400 rounded-md" autoplay></video>
    <p class="mt-4">Detected Gesture: {detectedGesture}</p>
</main>

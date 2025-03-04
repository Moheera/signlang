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
    const predictionsArray = await model.estimateHands(video, true); // Detect multiple hands
    predictions = predictionsArray;
    
    if (predictionsArray.length > 0) {
        detectedGesture = recognizeGesture(predictionsArray);
    } else {
        detectedGesture = "No hand detected";
    }
    
    requestAnimationFrame(detectHands);
}

function recognizeGesture(hands: any[]) {
    if (!hands || hands.length === 0) return "Unknown";
    
    const gestures: any = hands.map((hand: { landmarks: any; }) => identifyHandGesture(hand.landmarks));
    return gestures.length === 2 ? recognizeTwoHandGestures(gestures) : gestures[0];
}

function identifyHandGesture(landmarks: any | any[]) {
    if (!landmarks || landmarks.length < 21) return "Unknown";
    
    const fingers = getFingerStates(landmarks);
    
    if (fingers.index && fingers.middle && fingers.ring && fingers.pinky && fingers.thumb) {
        return "Open Hand";
    }
    if (fingers.thumb && !fingers.index && !fingers.middle && !fingers.ring && !fingers.pinky) {
        return "Thumbs Up (Yes)";
    }
    if (!fingers.thumb && fingers.index && !fingers.middle && !fingers.ring && !fingers.pinky) {
        return "No (Index Finger Shaking)";
    }
    if (!fingers.thumb && !fingers.index && !fingers.middle && !fingers.ring && !fingers.pinky) {
        return "Fist (Sorry)";
    }
    if (fingers.index && fingers.middle && !fingers.ring && !fingers.pinky) {
        return "Peace Sign (Play)";
    }
    return "Unknown";
}

function recognizeTwoHandGestures(gestures: [any, any]) {
    const [gesture1, gesture2] = gestures;
    if (gesture1 === "Open Hand" && gesture2 === "Open Hand") return "All Done";
    if (gesture1 === "Fist (Sorry)" && gesture2 === "Fist (Sorry)") return "More";
    if (gesture1 === "Thumbs Up (Yes)" && gesture2 === "Thumbs Up (Yes)") return "Help";
    return "Unknown Two-Hand Gesture";
}

function getFingerStates(landmarks: number[][]) {
    function isExtended(tipIndex: number, pipIndex: number, baseIndex: number) {
        return landmarks[tipIndex][1] < landmarks[pipIndex][1] && landmarks[pipIndex][1] < landmarks[baseIndex][1];
    }
    
    return {
        thumb: landmarks[4][0] > landmarks[3][0],
        index: isExtended(8, 6, 5),
        middle: isExtended(12, 10, 9),
        ring: isExtended(16, 14, 13),
        pinky: isExtended(20, 18, 17),
    };
}
</script>

<main class="flex flex-col items-center justify-center h-screen bg-gray-900 text-white">
    <h1 class="text-2xl font-bold">Hand Gesture Recognition</h1>
    <p class="text-lg text-gray-300">Try the following gestures and see the detected result:</p>
    <ul class="text-gray-400 text-sm mt-2">
        <li>- Open Hand → "Open Hand"</li>
        <li>- Two Open Hands → "All Done"</li>
        <li>- Fist → "Fist (Sorry)"</li>
        <li>- Two Fists → "More"</li>
        <li>- Thumbs Up → "Yes"</li>
        <li>- Two Thumbs Up → "Help"</li>
        <li>- Peace Sign → "Play"</li>
    </ul>
    <!-- svelte-ignore a11y_media_has_caption -->
    <video id="video" class="w-96 h-72 border border-gray-400 rounded-md mt-4" autoplay></video>
    <p class="mt-4 text-lg">Detected Gesture: {detectedGesture}</p>
</main>

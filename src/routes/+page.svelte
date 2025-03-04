<script lang="ts">
    import { onMount, tick } from 'svelte';
    import * as tf from '@tensorflow/tfjs';
    import * as handpose from '@tensorflow-models/handpose';
    import '@tensorflow/tfjs-backend-webgl';
    import cameraUtils from '@mediapipe/camera_utils';

    const { Camera } = cameraUtils;

    let video: HTMLVideoElement | null = null;
    let model: handpose.HandPose | null = null;
    let detectedGesture = "";
    const GESTURE_HISTORY_LENGTH = 5;
    let gestureHistory: string[] = [];
    let isDetecting = false;

    onMount(async () => {
        await tick();
        model = await handpose.load();
        await setupCamera();
    });

    async function setupCamera() {
        video = document.getElementById('video') as HTMLVideoElement;
        if (!video) {
            console.error("Video element not found!");
            return;
        }

        const camera = new Camera(video, {
            onFrame: async () => {
                if (model && !isDetecting) {
                    isDetecting = true;
                    await detectHands();
                    isDetecting = false;
                }
            },
            width: 640,
            height: 480,
            facingMode: 'user'
        });

        await camera.start();
    }

    async function detectHands() {
        if (!model || !video || video.videoWidth === 0 || video.videoHeight === 0) return;

        const predictions = await model.estimateHands(video, true);

        if (predictions.length > 0) {
            const currentGesture = recognizeGesture(predictions);
            updateGestureHistory(currentGesture);
            detectedGesture = getMostFrequentGesture();
        } else {
            detectedGesture = "No hand detected";
            gestureHistory = [];
        }

        // Dispose tensors properly
        predictions.forEach(p => tf.dispose(p.landmarks));
    }

    function updateGestureHistory(gesture: string) {
        gestureHistory.push(gesture);
        if (gestureHistory.length > GESTURE_HISTORY_LENGTH) {
            gestureHistory.shift();
        }
    }

    function getMostFrequentGesture(): string {
        const counts: Record<string, number> = {};
        gestureHistory.forEach(g => counts[g] = (counts[g] || 0) + 1);
        return Object.entries(counts).sort((a, b) => b[1] - a[1])[0]?.[0] || "Unknown";
    }

    function recognizeGesture(hands: Array<{ landmarks: number[][] }>): string {
        if (!hands.length) return "Unknown";

        const gestures = hands.map(hand => {
            const fingers = getFingerStates(hand.landmarks, video!.videoWidth, video!.videoHeight);
            return identifyHandGesture(fingers);
        });

        return gestures.length === 2 ? recognizeTwoHandGestures(gestures as [string, string]) : gestures[0];
    }

    function identifyHandGesture(fingers: ReturnType<typeof getFingerStates>): string {
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

    function getFingerStates(landmarks: number[][], width: number, height: number) {
        const normalized = landmarks.map(l => [l[0] / width, l[1] / height, l[2]]);

        const calculateAngle = (a: number[], b: number[], c: number[]) => {
            const ab = [a[0] - b[0], a[1] - b[1], a[2] - b[2]];
            const bc = [c[0] - b[0], c[1] - b[1], c[2] - b[2]];
            const dot = ab[0] * bc[0] + ab[1] * bc[1] + ab[2] * bc[2];
            const magAB = Math.sqrt(ab[0] ** 2 + ab[1] ** 2 + ab[2] ** 2);
            const magBC = Math.sqrt(bc[0] ** 2 + bc[1] ** 2 + bc[2] ** 2);
            return Math.acos(dot / (magAB * magBC)) * (180 / Math.PI);
        };

        return {
            thumb: calculateAngle(normalized[2], normalized[3], normalized[4]) > 100,
            index: calculateAngle(normalized[5], normalized[6], normalized[8]) > 160,
            middle: calculateAngle(normalized[9], normalized[10], normalized[12]) > 160,
            ring: calculateAngle(normalized[13], normalized[14], normalized[16]) > 160,
            pinky: calculateAngle(normalized[17], normalized[18], normalized[20]) > 160,
        };
    }

    function recognizeTwoHandGestures(gestures: [string, string]): string {
        const [g1, g2] = gestures.sort();
        if (g1 === "Open Hand" && g2 === "Open Hand") return "All Done";
        if (g1 === "Fist (Sorry)" && g2 === "Fist (Sorry)") return "More";
        if (g1 === "Thumbs Up (Yes)" && g2 === "Thumbs Up (Yes)") return "Help";
        return "Unknown Two-Hand Gesture";
    }
</script>

<main class="flex flex-col items-center justify-center h-screen bg-gray-900 text-white">
    <h1 class="text-2xl font-bold mb-4">Enhanced Hand Gesture Recognition</h1>
    <div class="grid grid-cols-2 gap-4 mb-8">
        <div class="bg-gray-800 p-4 rounded-lg">
            <h2 class="text-lg font-semibold mb-2">Single Hand Gestures</h2>
            <ul class="text-sm text-gray-300">
                <li>✋ Open Hand → "Open Hand"</li>
                <li>👍 Thumbs Up → "Yes"</li>
                <li>👊 Fist → "Sorry"</li>
                <li>✌️ Peace Sign → "Play"</li>
                <li>👆 Pointing → "No"</li>
            </ul>
        </div>
        <div class="bg-gray-800 p-4 rounded-lg">
            <h2 class="text-lg font-semibold mb-2">Two Hand Gestures</h2>
            <ul class="text-sm text-gray-300">
                <li>👐 Two Open → "All Done"</li>
                <li>🤜🤛 Two Fists → "More"</li>
                <li>👍👍 Two Thumbs → "Help"</li>
            </ul>
        </div>
    </div>
    <!-- svelte-ignore a11y_media_has_caption -->
    <video id="video" class="w-96 h-72 border-2 border-gray-600 rounded-lg shadow-lg" autoplay></video>
    <div class="mt-6 p-4 bg-gray-800 rounded-lg min-w-[300px] text-center">
        <p class="text-sm text-gray-400 mb-1">Detected Gesture:</p>
        <p class="text-xl font-bold text-green-400">{detectedGesture}</p>
    </div>
</main>
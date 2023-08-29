<template>
    <div class="flex flex-col space-y-4">
        <button :disabled="!userConsentGiven" @click="accessWebcamAndMicrophone"
            class="mb-4 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded shadow disabled:bg-slate-400 disabled:hover:cursor-not-allowed">
            <span class="material-icons-outlined mr-1">📷</span>
            Access Webcam and Microphone
        </button>

        <video v-if="videoDeviceInfo" ref="videoRef" class="mb-4 border rounded h-64" autoplay playsinline></video>

        <div v-if="audioDeviceInfo" class="mb-4 bg-white shadow-md rounded-md p-4">
            <span class="block mb-2">Microphone level:</span>
            <canvas ref="canvasRef" class="border rounded h-36" width="200" height="100"></canvas>
        </div>
    </div>
</template>
  
<script lang="ts">
import { ref, defineEmits } from 'vue';

export default {
    props: {
        userConsentGiven: Boolean,
        videoDeviceInfo: Object,
        audioDeviceInfo: Object,
    },
    emits: ['add-action', 'update-devices-name'],
    setup(props, { emit }) {
        const videoStream = ref<MediaStream | null>(null);
        const videoRef = ref<HTMLVideoElement | null>(null);
        const canvasRef = ref<HTMLCanvasElement | null>(null);
        let audioContext: AudioContext | null = null;
        let analyzerNode: AnalyserNode | null = null;
        let animationId: number | null = null;

        const videoDeviceInfo = ref<MediaDeviceInfo | null | undefined>(null);
        const audioDeviceInfo = ref<MediaDeviceInfo | null | undefined>(null);


        const accessWebcamAndMicrophone = async () => {

            try {
                const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
                await getDeviceNames(stream);

                videoStream.value = stream;

                if (videoRef.value) {
                    videoRef.value.srcObject = stream;
                }

                const audioTrack = stream.getAudioTracks()[0];
                audioContext = new AudioContext();
                const sourceNode = audioContext.createMediaStreamSource(new MediaStream([audioTrack]));
                analyzerNode = audioContext.createAnalyser();
                analyzerNode.fftSize = 128;
                sourceNode.connect(analyzerNode);

                emitAddAction('Webcam and microphone accessed');
                startVisualizer();

            } catch (error) {
                emitAddAction('Failed to access webcam or microphone');
            }
        };

        const emitAddAction = (action) => {
            emit('add-action', action)
        }

        const startVisualizer = () => {
            if (canvasRef.value && analyzerNode && audioContext) {
                const canvas = canvasRef.value;
                const ctx = canvas.getContext('2d');
                const bufferLength = analyzerNode.frequencyBinCount;
                const dataArray = new Uint8Array(bufferLength);

                const draw = () => {
                    if (animationId && ctx && analyzerNode) {
                        ctx.clearRect(0, 0, canvas.width, canvas.height);

                        analyzerNode.getByteFrequencyData(dataArray);

                        const barWidth = canvas.width / bufferLength;
                        let x = 0;

                        for (let i = 0; i < bufferLength; i++) {
                            const barHeight = dataArray[i];

                            ctx.fillStyle = `rgb(${barHeight + 100},50,50)`;
                            ctx.fillRect(x, canvas.height - barHeight / 2, barWidth, barHeight / 2);

                            x += barWidth + 1;
                        }

                        animationId = requestAnimationFrame(draw);
                    }
                };

                animationId = requestAnimationFrame(draw);
            }
        };

        const getDeviceNames = async (stream: MediaStream) => {
            const devices = await navigator.mediaDevices.enumerateDevices();

            const videoDeviceId = stream.getVideoTracks()[0]?.getSettings().deviceId;
            const audioDeviceId = stream.getAudioTracks()[0]?.getSettings().deviceId;

            if (videoDeviceId) {
                videoDeviceInfo.value = devices.find(device => device.deviceId === videoDeviceId);
            }

            if (audioDeviceId) {
                audioDeviceInfo.value = devices.find(device => device.deviceId === audioDeviceId);
            }
            console.log(audioDeviceInfo.value, videoDeviceInfo.value);

            emit('update-devices-name', { videoDeviceInfo, audioDeviceInfo })

        };

        return { accessWebcamAndMicrophone, videoRef, canvasRef, audioDeviceInfo, videoDeviceInfo };
    },
};
</script>
  
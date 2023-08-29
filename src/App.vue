<template>
  <div class="min-h-screen flex flex-col justify-center items-center bg-gray-100  ">
    <h1 class="text-2xl mb-4">Monitoring User Interactions</h1>

    <p class="mb-4 text-green-500" v-if="isTabActive">Tab is active</p>
    <p class="mb-4 text-red-500" v-else>Tab is not active</p>

    <p class="mb-4 text-green-500" v-if="videoStream">Video and microphone are active</p>
    <p class="mb-4 text-red-500" v-else>Video and microphone are not active</p>

    <span id="microphone-label"></span> <span id="video-label"></span>


    <button @click="accessWebcamAndMicrophone"
      class="mb-4 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">Access Webcam and
      Microphone</button>
    <video ref="videoRef" class="mb-4 border rounded h-64" autoplay playsinline></video>
    <span>Microphone level:</span>
    <canvas ref="canvasRef" class="mb-4 border rounded h-36" width="200" height="100"></canvas>

    <h2 class="text-xl mb-2">User Actions:</h2>
    <ul class="bg-white shadow-md rounded p-4 overflow-auto">
      <li v-for="(action, index) in actions" :key="index" class="border-b py-1">{{ action.timestamp }} - {{ action.message
      }}</li>
    </ul>
  </div>
</template>

<script lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue';

interface UserAction {
  message: string;
  timestamp: string;
}

export default {
  name: 'App',
  setup() {
    const isTabActive = ref(true);
    const actions = ref<UserAction[]>([]);
    const videoStream = ref<MediaStream | null>(null);
    const videoRef = ref<HTMLVideoElement | null>(null);
    const canvasRef = ref<HTMLCanvasElement | null>(null);
    let audioContext: AudioContext | null = null;
    let analyzerNode: AnalyserNode | null = null;
    let animationId: number | null = null;

    const addAction = (message: string) => {
      const timestamp = new Date().toLocaleTimeString();
      actions.value.push({ message, timestamp });
    };

    const handleVisibilityChange = () => {
      if (document.visibilityState === 'visible') {
        addAction('Tab activated');
        isTabActive.value = true;
      } else {
        addAction('Tab deactivated');
        isTabActive.value = false;
      }
    };

    const accessWebcamAndMicrophone = async () => {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
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

        addAction('Webcam and microphone accessed');
        startVisualizer();
      } catch (error) {
        console.error('Error accessing webcam or microphone:', error);
        addAction('Failed to access webcam or microphone');
      }
    };

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

    const stopVisualizer = () => {
      if (animationId !== null) {
        cancelAnimationFrame(animationId);
        animationId = null;
      }
    };

    onMounted(() => {
      document.addEventListener('visibilitychange', handleVisibilityChange);
    });

    onBeforeUnmount(() => {
      document.removeEventListener('visibilitychange', handleVisibilityChange);
      stopVisualizer();
      if (videoStream.value) {
        const tracks = videoStream.value.getTracks();
        tracks.forEach(track => {
          track.stop();
        });
      }
      audioContext?.close();
    });

    return {
      isTabActive,
      actions,
      accessWebcamAndMicrophone,
      videoStream,
      videoRef,
      canvasRef,
    };
  },
};

</script>

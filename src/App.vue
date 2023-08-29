<template>
  <div class="min-h-screen flex flex-col justify-center items-center bg-gray-200 p-8">
    <h1 class="text-3xl mb-6 text-gray-800 font-semibold">
      Monitoring User Interactions
    </h1>

    <div class="mb-6 flex space-x-4">
      <a href="https://github.com/eduardocappellotto/visibility-poc" class="text-blue-600 hover:text-blue-800">
        Github
      </a>
      <span class="text-gray-400">|</span>
      <a href="https://www.linkedin.com/in/eduardo-cappellotto-991529170/" class="text-blue-600 hover:text-blue-800">
        LinkedIn
      </a>
    </div>

    <div v-if="!userConsentGiven" class="fixed top-0 left-0 w-full p-4 bg-red-600 text-white z-50">
      <div class="container mx-auto">
        <div class="flex justify-between items-center">
          <div>
            <p class="font-bold">User Consent Required!</p>
            <p>This platform utilizes monitoring mechanisms for educational purposes. By proceeding, you are consenting to
              these tracking actions.</p>
          </div>
          <button @click="giveConsent" class="bg-green-500 hover:bg-green-700 px-4 py-2 rounded">I Understand &
            Consent</button>
        </div>
      </div>
    </div>

    <div class="mb-4 p-4 bg-white shadow-md rounded-md w-full max-w-lg space-y-4">
      <p class="flex items-center space-x-2" :class="{ 'text-green-600': isTabActive, 'text-red-600': !isTabActive }">
        <span class="material-icons" :class="{ 'text-green-600': isTabActive, 'text-red-600': !isTabActive }">
          {{ isTabActive ? '✅' : '❌' }}
        </span>
        <span>
          {{ isTabActive ? 'Tab is active' : 'Tab is not active' }}
        </span>
      </p>

      <p :class="{ 'text-green-600': videoDeviceInfo, 'text-red-600': !videoDeviceInfo }">
        Camera: {{ videoDeviceInfo ? videoDeviceInfo.label : 'Not detected' }}
      </p>

      <p :class="{ 'text-green-600': audioDeviceInfo, 'text-red-600': !audioDeviceInfo }">
        Microphone: {{ audioDeviceInfo ? audioDeviceInfo.label : 'Not detected' }}
      </p>

      <p class="flex items-center space-x-2" :class="{ 'text-green-600': isFocused, 'text-red-600': !isFocused }">
        <span class="material-icons" :class="{ 'text-green-600': isFocused, 'text-red-600': !isFocused }">
          {{ isFocused ? '✅' : '❌' }}
        </span>
        <span>
          {{ isFocused ? 'Exam tab is in focus' : 'Exam tab lost focus' }}
        </span>
      </p>
    </div>


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

    <div class="bg-white shadow-lg rounded-lg p-4 w-full max-w-xl">
      <h2 class="text-xl mb-4 border-b pb-2">User Actions:</h2>
      <ul class="overflow-auto">
        <li v-for="(action, index) in actions" :key="index" class="border-b py-1 text-gray-700">
          {{ action.timestamp }} - {{ action.message }}
        </li>
      </ul>
    </div>
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

    const videoDeviceInfo = ref<MediaDeviceInfo | null | undefined>(null);
    const audioDeviceInfo = ref<MediaDeviceInfo | null | undefined>(null);

    const isFocused: boolean = ref(true);
    const userConsentGiven: boolean = ref(false);


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

    const handleFocus = () => {
      addAction('Exam tab focused');
      isFocused.value = true;
    };

    const handleBlur = () => {
      addAction('Exam tab lost focus');
      isFocused.value = false;
    };

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

        addAction('Webcam and microphone accessed');
        startVisualizer();
      } catch (error) {
        console.error('Error accessing webcam or microphone:', error);
        addAction('Failed to access webcam or microphone');
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
    const giveConsent = () => {
      userConsentGiven.value = true;
    };

    onMounted(() => {
      document.addEventListener('visibilitychange', handleVisibilityChange);
      window.addEventListener('focus', handleFocus);
      window.addEventListener('blur', handleBlur);
    });

    onBeforeUnmount(() => {
      document.removeEventListener('visibilitychange', handleVisibilityChange); window.removeEventListener('focus', handleFocus); window.removeEventListener('blur', handleBlur);

      if (videoStream.value) { videoStream.value.getTracks().forEach(track => track.stop()); }

      if (audioContext) { audioContext.close(); }

      if (animationId) { cancelAnimationFrame(animationId); }
    });

    return { isTabActive, videoDeviceInfo, audioDeviceInfo, isFocused, accessWebcamAndMicrophone, actions, videoRef, canvasRef, giveConsent, userConsentGiven };

  }
}
</script>
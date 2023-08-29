<template>
  <div class="min-h-screen flex flex-col justify-center items-center bg-gray-100">
    <h1 class="text-2xl mb-4">Monitoring User Interactions</h1>

    <p class="mb-4" :class="{ 'text-green-500': isTabActive, 'text-red-500': !isTabActive }">
      {{ isTabActive ? 'Tab is active' : 'Tab is not active' }}
    </p>

    <p class="mb-4" :class="{ 'text-green-500': videoDeviceInfo, 'text-red-500': !videoDeviceInfo }">
      {{ videoDeviceInfo ? 'Camera: ' + videoDeviceInfo.label : 'Camera not detected' }}
    </p>

    <p class="mb-4" :class="{ 'text-green-500': audioDeviceInfo, 'text-red-500': !audioDeviceInfo }">
      {{ audioDeviceInfo ? 'Microphone: ' + audioDeviceInfo.label : 'Microphone not detected' }}
    </p>

    <p class="mb-4" :class="{ 'text-green-500': isFocused, 'text-red-500': !isFocused }">
      {{ isFocused ? 'Exam tab is in focus' : 'Exam tab lost focus' }}
    </p>

    <span id="microphone-label"></span>
    <span id="video-label"></span>

    <button @click="accessWebcamAndMicrophone"
      class="mb-4 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">
      Access Webcam and Microphone
    </button>
    <video ref="videoRef" class="mb-4 border rounded h-64" autoplay playsinline></video>
    <span>Microphone level:</span>
    <canvas ref="canvasRef" class="mb-4 border rounded h-36" width="200" height="100"></canvas>

    <h2 class="text-xl mb-2">User Actions:</h2>
    <ul class="bg-white shadow-md rounded p-4 overflow-auto">
      <li v-for="(action, index) in actions" :key="index" class="border-b py-1">
        {{ action.timestamp }} - {{ action.message }}
      </li>
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

    const videoDeviceInfo = ref<MediaDeviceInfo | null>(null);
    const audioDeviceInfo = ref<MediaDeviceInfo | null>(null);
    const isFocused = ref(true);

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
        videoStream.value = stream;
        getDeviceNames(stream);

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
    const getDeviceNames = (stream: MediaStream) => {
      const videoTracks = stream.getVideoTracks(); const audioTracks = stream.getAudioTracks();

      if (videoTracks.length > 0) {
        const videoTrack = videoTracks[0]; videoDeviceInfo.value = videoTrack.getSettings().deviceId; const videoLabel = videoTrack.getLabel(); const videoLabelElement = document.getElementById('video-label');

        if (videoLabelElement) {
          videoLabelElement.innerText = `Camera: ${videoLabel}`;
        }


      }

      if (audioTracks.length > 0) {
        const audioTrack = audioTracks[0]; audioDeviceInfo.value = audioTrack.getSettings().deviceId; const audioLabel = audioTrack.getLabel(); const audioLabelElement = document.getElementById('microphone-label');

        if (audioLabelElement) {
          audioLabelElement.innerText = `Microphone: ${audioLabel}`;
        }


      }
    };

    const startVisualizer = () => {
      if (canvasRef.value && analyzerNode) {
        const canvas = canvasRef.value; const context = canvas.getContext('2d');

        if (context) {
          const bufferLength = analyzerNode.frequencyBinCount;
          const dataArray = new Uint8Array(bufferLength);

          const draw = () => {
            animationId = requestAnimationFrame(draw);

            analyzerNode.getByteTimeDomainData(dataArray);

            context.clearRect(0, 0, canvas.width, canvas.height);

            context.lineWidth = 2;
            context.strokeStyle = 'rgb(0, 0, 0)';
            context.beginPath();

            const sliceWidth = canvas.width * 1.0 / bufferLength;
            let x = 0;

            for (let i = 0; i < bufferLength; i++) {
              const v = dataArray[i] / 128.0;
              const y = v * canvas.height / 2;

              if (i === 0) {
                context.moveTo(x, y);
              } else {
                context.lineTo(x, y);
              }

              x += sliceWidth;
            }

            context.lineTo(canvas.width, canvas.height / 2);
            context.stroke();
          };

          draw();
        }


      }
    };

    onMounted(() => { document.addEventListener('visibilitychange', handleVisibilityChange); window.addEventListener('focus', handleFocus); window.addEventListener('blur', handleBlur); });

    onBeforeUnmount(() => {
      document.removeEventListener('visibilitychange', handleVisibilityChange); window.removeEventListener('focus', handleFocus); window.removeEventListener('blur', handleBlur);

      if (videoStream.value) { videoStream.value.getTracks().forEach(track => track.stop()); }

      if (audioContext) { audioContext.close(); }

      if (animationId) { cancelAnimationFrame(animationId); }
    });

    return { isTabActive, videoDeviceInfo, audioDeviceInfo, isFocused, accessWebcamAndMicrophone, actions, };

  }
}
</script>
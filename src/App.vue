<template>
  <div class="min-h-screen flex flex-col justify-center items-center bg-gray-200 p-8">

    <HeaderComponent />
    <ConsentAlertComponent :user-consent-given="userConsentGiven" @giveConsent="giveConsent" />
    <StatusComponent :is-tab-active="isTabActive" :video-device-info="videoDeviceInfo"
      :audio-device-info="audioDeviceInfo" :is-focused="isFocused" />
    <MediaAccessComponent :user-consent-given="userConsentGiven" :video-device-info="videoDeviceInfo"
      :audio-device-info="audioDeviceInfo" @updateDevicesName="updateDevicesName" @addAction="addAction" />
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
import HeaderComponent from '@/components/HeaderComponent.vue'
import ConsentAlertComponent from '@/components/ConsentAlertComponent.vue'
import StatusComponent from '@/components/StatusComponent.vue'
import MediaAccessComponent from '@/components/MediaAccessComponent.vue'


interface UserAction {
  message: string;
  timestamp: string;
}

export default {
  name: 'App',
  components: {
    HeaderComponent,
    ConsentAlertComponent,
    StatusComponent,
    MediaAccessComponent
  },
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

    const isFocused = ref<boolean>(true);
    const userConsentGiven = ref<boolean>(false);


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

    const giveConsent = () => {
      userConsentGiven.value = true;
    };
    const updateDevicesName = (payload) => {
      audioDeviceInfo.value = payload.audioDeviceInfo.value
      videoDeviceInfo.value = payload.videoDeviceInfo.value
    }

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

    return { isTabActive, videoDeviceInfo, audioDeviceInfo, isFocused, actions, videoRef, canvasRef, giveConsent, userConsentGiven, updateDevicesName };

  }
}
</script>
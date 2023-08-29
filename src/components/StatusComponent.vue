<template>
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
</template>
  
<script lang="ts">
export default {
    props: {
        isTabActive: Boolean,
        videoDeviceInfo: Object,
        audioDeviceInfo: Object,
        isFocused: Boolean,
    },
};
</script>
  
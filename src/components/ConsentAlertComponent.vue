<template>
    <div v-if="!userConsentGiven" class="fixed top-0 left-0 w-full p-4 bg-red-600 text-white z-50">
        <div class="container mx-auto">
            <div class="flex justify-between items-center">
                <div>
                    <p class="font-bold">User Consent Required!</p>
                    <p>This platform utilizes monitoring mechanisms for educational purposes. By proceeding, you are
                        consenting to these tracking actions.</p>
                </div>
                <button @click="giveConsent" class="bg-green-500 hover:bg-green-700 px-4 py-2 rounded">I Understand &
                    Consent</button>
            </div>
        </div>
    </div>
</template>
  
<script>
export default {
    props: {
        userConsentGiven: Boolean,
    },
    methods: {
        giveConsent() {
            this.$emit('give-consent');
        },
    },
};
</script>
  
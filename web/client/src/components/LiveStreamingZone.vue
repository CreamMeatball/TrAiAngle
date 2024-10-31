<script setup>
import { ref, watch, onMounted, onUnmounted } from "vue";
import { useRouter } from "vue-router";
import { useVideoStore} from "@/videoStore";

const router = useRouter();
const videoStore = useVideoStore()
const camera = ref(null);
const mediaRecorder = ref(null);
const recordedChunks = ref([]);
const isRecording = ref(false);

const getVideo = () => {
    navigator.mediaDevices
        .getUserMedia({
          video: {
            width: { ideal: 1280 },
            height: { ideal: 832 },
            // frameRate: { ideal: 30 },
            facingMode: "user",
          },
        })
        .then((stream) => {
            let video = camera.value;
            video.srcObject = stream;
            video.play();
            mediaRecorder.value = new MediaRecorder(stream);
            mediaRecorder.value.ondataavailable = (event) => {
                if (event.data.size > 0) {
                    recordedChunks.value.push(event.data);
                }
            };
        })
        .catch((err) => console.log(err));
};

const startRecording = () => {
    recordedChunks.value = [];
    mediaRecorder.value.start();
    isRecording.value = true;
};

const stopRecording = () => {
    mediaRecorder.value.stop();
    isRecording.value = false;
};

const saveRecording = () => {
  const blob = new Blob(recordedChunks.value, { type: "video/webm" });
  console.log('Blob:', blob); // Should show type "video/webm"

  // No need to create a File object here unless necessary
  videoStore.videoBlob = blob;
  router.push("/video");
};

watch(camera, () => {
    getVideo();
});

onMounted(() => {
    getVideo();
});

onUnmounted(() => {
    if (mediaRecorder.value && mediaRecorder.value.state !== "inactive") {
        mediaRecorder.value.stop();
    }
});
</script>

<template>
    <div class="camera">
        <div class="camera__wrapper">
            <video ref="camera" class="flipped"></video>
        </div>

        <button @click="isRecording ? stopRecording() : startRecording()" class="recordButton">
            {{ isRecording ? "🟥 Stop Recording" : "🔴 Start Recording" }}
        </button>
        <button @click="saveRecording" :disabled="isRecording" class="recordButton">✅ Save Recording</button>
    </div>
</template>

<style lang="scss" scoped>
.camera {
    //background-color: rgba(0, 255, 255, 0.5);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    gap: 2rem;

    &__wrapper {
        display: flex;
        flex-direction: row;
        justify-content: center;
        width: 100%;
        overflow: hidden;

        video {
            width: 80%;
            height: auto;
        }
    }
}

.flipped {
    transform: scaleX(-1); /* Flip the video horizontally */
}

.recordButton {
    display: flex;
    width: 50%;
    justify-content: center;
    align-items: center;
    padding: 1rem 0;
    flex: 45%;
    background-color: whitesmoke;
    color: var(--secondary-color);
    text-transform: uppercase;
    border: 3px solid var(--primary-color);
    border-radius: 0.3rem;
    cursor: pointer;
    transition: all 0.25s ease;

    &:hover {
      box-shadow: 0 6px 18px 0 rgba(#000, 0.1);
      transform: translateY(-6px);
    }

    &.active {
      background-color: var(--primary-color);
      color: whitesmoke;
      font-weight: 700;
    }
}
</style>
<script setup>
import { ref, computed, onMounted } from "vue";

const apiUrl = import.meta.env.VITE_BASE_URL;

const { videoName, startAt } = defineProps({
    videoName: String,
    startAt: {
        required: false,
        type: Number,
    },
});

const url = computed(
    () => `${apiUrl}/api/video/stream?video_name=${videoName}`
    // () => `http://baobao.com`
);

const video = ref(null);
const videoContainer = ref(null);
onMounted(() => {
    if (startAt) video.value.currentTime = startAt;
});

const handleVideoLoad = () => {
    videoContainer.value.scrollIntoView({
        behavior: "smooth",
        block: "center",
    });
};

// 비디오 재생 위치를 변경하는 메서드
const seekTo = (second) => {
    if (video.value) {
        video.value.currentTime = second;
        video.value.play(); // 필요한 경우 재생
    }
};

// 부모 컴포넌트에 메서드 노출
defineExpose({
    seekTo,
});
</script>

<template>
    <!-- Video Player -->
<!--    <p>{{ url }}</p>-->
    <div class="player" ref="videoContainer">
        <video
            controls
            ref="video"
            @loadeddata="handleVideoLoad"
            autoplay
            loop
            muted
        >
        <source :src="`${url}`" type="video/mp4" />
        </video>
    </div>
</template>

<style lang="scss" scoped>
video {
    width: 100%;
}
</style>

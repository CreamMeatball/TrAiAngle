<script setup>
import { ref, computed, onMounted } from "vue";
import OpenAI from "openai";

import Video from "./Video.vue";

import DOMPurify from 'dompurify';

const { data } = defineProps(["data"]);
// Video 컴포넌트에 대한 참조 생성
const videoPlayer = ref(null);

const summaryData = computed(() => {
    let results = {
        type: data.type,
        total: 0,
        totalInString: "",
        details: {},
    };

    let totalErrors = data.details.length;
    results.total = totalErrors;
    if (totalErrors == 0 || totalErrors == 1)
        results.totalInString = `${totalErrors} error`;
    else results.totalInString = `${totalErrors} errors`;

    data.details.forEach((error) => {
        let stage = error.stage;
        if (!results.details[stage]) {
            results.details[stage] = { total: 0, timestamps: [] };
        }
        results.details[stage].total += 1;
        results.details[stage].timestamps.push(error.timestamp);
    });

    return results;
});
const selectedDisplay = ref("summary");
const videoStart = ref(0);

// const jumpToVideoLocation = (second) => {
//     // selectedDisplay.value = "video";
//     selectedDisplay.value = "detail";
//     videoStart.value = second;
// };

const seekVideo = (second) => {
    if (videoPlayer.value && videoPlayer.value.seekTo) {
        videoPlayer.value.seekTo(second);
    }
};

// ChatGPT
const chatGptMessage = ref("Before Loaded");
// const APIKEY = import.meta.env.VITE_OPENAI_API_KEY;
// console.log('APIKEY :', APIKEY);

const loading = ref(true);
const callChatGPT = async (summaryData) => {
    loading.value = true;
    const openai = new OpenAI({
        apiKey: import.meta.env.VITE_OPENAI_API_KEY,
        dangerouslyAllowBrowser: true,
    });
    const errors = Object.entries(summaryData.details).map(
        ([error, info]) =>
            `${error}: ${info.total} times at timestamps: ${info.timestamps.join(", ")}`
    );
    const prompt = `I have analyzed the video and found ${summaryData.totalInString}.
                    The video is an analysis after doing ${summaryData.type} exercise.
                    Here are the details:\n${errors.join("\n")}.
                    Tell me what wrong pose occurred at a certain time, why the wrong pose is bad, and how to fix it.
                    Please write a pretty look using the symbol and emoji,

                    Please provide feedback on each error in the following JSON format:

                    [
                      {
                        "error": "Error description in English, (line break) Error description in Korean",
                        "timestamp": "Timestamp",
                        "issue": "Explanation of the issue in English, (line break) Explanation of the issue in Korean",
                        "why_bad": "Why it's bad explanation in English, (line break) Why it's bad explanation in Korean",
                        "fix": "How to fix it explanation in English, (line break) How to fix it explanation in Korean"
                      },
                      ...
                    ]
                    `;

    const response = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages: [
            {
                role: 'system',
                content: `You are a helpful ${summaryData.type} fitness trainer. Provide detailed and friendly feedback.`,
            },
            {
                role: 'user',
                content: prompt,
            },
        ]
    });

    const content = response.choices[0].message.content
    console.log(content)

    // // Extract HTML content
    // const { bodyContent, styleContent } = extractHTMLContent(content);
    //
    // const sanitizedContent = DOMPurify.sanitize(bodyContent);
    //
    // const messages = parseChatGPTResponse(sanitizedContent);
    //
    // // Inject style into the DOM
    // if (styleContent) {
    //     const styleElement = document.createElement('style');
    //     styleElement.innerHTML = styleContent;
    //     document.head.appendChild(styleElement);
    // }

    // JSON 파싱 시도
    let messages = [];
    try {
        // 응답에서 JSON 부분 추출
        const jsonStartIndex = content.indexOf('[');
        const jsonEndIndex = content.lastIndexOf(']');
        const jsonString = content.substring(jsonStartIndex, jsonEndIndex + 1);

        messages = JSON.parse(jsonString);
    } catch (error) {
        console.error('ChatGPT 응답에서 JSON 파싱 실패:', error);
        console.error('응답 내용:', content);
        messages = [];
    }

    // Sanitize and set the body content
    chatGptMessage.value = messages;
    loading.value = false;
};

// function extractHTMLContent(text) {
//     const regex = /```html\n([\s\S]*?)```/;
//     const match = text.match(regex);
//     if (match && match[1]) {
//         const htmlContent = match[1];
//
//         // Extract <style> content
//         const styleRegex = /<style[^>]*>([\s\S]*?)<\/style>/i;
//         const styleMatch = htmlContent.match(styleRegex);
//         let styleContent = '';
//         if (styleMatch && styleMatch[1]) {
//             styleContent = styleMatch[1];
//         }
//
//         // Extract <body> content
//         const bodyRegex = /<body[^>]*>([\s\S]*?)<\/body>/i;
//         const bodyMatch = htmlContent.match(bodyRegex);
//         let bodyContent = '';
//         if (bodyMatch && bodyMatch[1]) {
//             bodyContent = bodyMatch[1];
//         } else {
//             // If no <body> tag, remove <style> and use the remaining content
//             bodyContent = htmlContent.replace(styleRegex, '');
//         }
//
//         return { bodyContent, styleContent };
//     } else {
//         // If no HTML code block, return the original text
//         return { bodyContent: text, styleContent: '' };
//     }
// }

// function parseChatGPTResponse(content) {
//   const items = content.split(/\n(?=\d+\.\s)/g);
//   return items.map(item => item.trim()).filter(item => item.length > 0);
// }

onMounted(async () => {
    await callChatGPT(summaryData.value);
});


</script>

<template>
    <section class="result-section">
        <!-- Navigators -->
        <ul class="tab-links">
<!--            <li-->
<!--                :class="{ active: selectedDisplay == 'chatgpt' }"-->
<!--                @click="() => (selectedDisplay = 'chatgpt')"-->
<!--            >-->
<!--                AI's Feedback-->
<!--            </li>-->
            <li
                :class="{ active: selectedDisplay == 'summary' }"
                @click="() => (selectedDisplay = 'summary')"
            >
                AI Summary
            </li>
            <li
                :class="{ active: selectedDisplay == 'detail' }"
                @click="() => (selectedDisplay = 'detail')"
            >
                Detail
            </li>
<!--            <li-->
<!--                :class="{ active: selectedDisplay == 'video' }"-->
<!--                @click="() => (selectedDisplay = 'video')"-->
<!--            >-->
<!--                Full Video-->
<!--            </li>-->
        </ul>

        <!-- Contents -->
        <div class="tab-container">
            <!-- ChatGPT content -->
<!--            <template v-if="selectedDisplay == 'chatgpt'">-->
<!--                <div class="chatgpt-messages">-->
<!--                    <div v-if="loading" class="chatgpt-message">-->
<!--                        Before Loaded-->
<!--                    </div>-->
<!--                    <div v-else>-->
<!--                        <div-->
<!--                            class="chatgpt-message"-->
<!--                            v-for="(message, index) in chatGptMessage"-->
<!--                            :key="index"-->
<!--                        >-->
<!--                            <h3><b>{{ index + 1 }}. {{ message.error }} (🕗 Timestamp: {{ message.timestamp }})</b></h3>-->
<!--                            <p><b>❗️ Issue:</b> {{ message.issue }}</p>-->
<!--                            <p><b>🤔 Why It's Bad:</b> {{ message.why_bad }}</p>-->
<!--                            <p><b>✅ Fix:</b> {{ message.fix }}</p>-->
<!--                        </div>-->
<!--                    </div>-->
<!--                </div>-->
<!--            </template>-->

            <!-- Summary content -->
            <template v-if="selectedDisplay == 'summary'">
                <!-- Display Counter or other information -->
                <p class="main" v-if="data.counter">
                    <span class="info-color" v-if="data.type != 'bicep_curl'">
                        Counter: {{ data.counter }}
                    </span>

                    <span class="info-color" v-else>
                        Left arm counter: {{ data.counter.left_counter }} -
                        Right arm counter: {{ data.counter.right_counter }}
                    </span>
                </p>

                <!-- Display error -->
                <p class="main">
                    There are
                    <span class="error-color">
                        {{ summaryData.totalInString }}
                    </span>
                    found.

                    <!-- Icon -->
                    <i
                        class="fa-solid fa-circle-exclamation error-color"
                        v-if="summaryData.total > 0"
                    ></i>
                    <i class="fa-solid fa-circle-check" v-else></i>
                </p>

                <ul class="errors" v-if="summaryData.total > 0">
                    <li v-for="(detail, error) in summaryData.details">
                        <i class="fa-solid fa-caret-right"></i>

                        {{ error }}: {{ detail.total }} <span v-if="detail.total < 2">time</span><span v-else>times</span>
                    </li>
                </ul>

                <div class="chatgpt-messages">
                    <div v-if="loading" class="chatgpt-message">
                        Before Loaded
                    </div>
                    <div v-else>
                        <div
                            class="chatgpt-message"
                            v-for="(message, index) in chatGptMessage"
                            :key="index"
                        >
                            <h3><b>{{ index + 1 }}. {{ message.error }} (🕗 Timestamp: {{ message.timestamp }})</b></h3>
                            <p><b>❗️ Issue:</b> {{ message.issue }}</p>
<!--                            <p><b>🤔 Why It's Bad:</b> {{ message.why_bad }}</p>-->
                            <p><b>✅ Fix:</b> {{ message.fix }}</p>
                        </div>
                    </div>
                </div>
            </template>

            <!-- Detail Content -->
            <KeepAlive>
                <template v-if="selectedDisplay == 'detail'">
                    <div class="video-container">
                        <Video
                            :video-name="data.file_name"
                            :start-at="videoStart"
                            ref="videoPlayer"
                        />
                    </div>
                    <div v-if="loading" class="chatgpt-message">
                        Before Loaded
                    </div>
                    <div v-else>
                        <div
                            class="box-error"
                            v-for="(error, index) in data.details"
                        >
                            <p>
                              <b> {{ index + 1 }}. {{ error.stage }} at
                                <span
                                    class="error-time"
                                    @click="seekVideo(error.timestamp)"
                                >
                                    🕗 {{ error.timestamp }} second
                                </span> </b>
                            </p>
                            <img :src="`${error.frame}`" style="display:block; margin:auto;"/>
                            <hr />
                            <div v-if="chatGptMessage && chatGptMessage.length > index">
                                <p><b>❗️ Issue:</b> {{ chatGptMessage[index].issue }}</p>
                                <p><b>🤔 Why It's Bad:</b> {{ chatGptMessage[index].why_bad }}</p>
                                <p><b>✅ Fix:</b> {{ chatGptMessage[index].fix }}</p>
                            </div>
                        </div>
                    </div>
                </template>
            </KeepAlive>

            <!-- Full Video content -->
<!--            <KeepAlive>-->
<!--                <template v-if="selectedDisplay == 'video'">-->
<!--                    <div class="video-container">-->
<!--                        <Video-->
<!--                            :video-name="data.file_name"-->
<!--                            :start-at="videoStart"-->
<!--                        />-->
<!--                    </div>-->
<!--                </template>-->
<!--            </KeepAlive>-->
        </div>
    </section>
</template>

<style lang="scss" scoped>
.result-section {
    margin-top: 2rem;
    margin-bottom: 5rem;

    .tab-links {
        display: flex;

        li {
            width: 6em;
            padding: 0.75rem 1rem;
            padding-right: 1.2rem;
            background-color: rgb(180, 179, 179);
            border-top-left-radius: 1rem;
            border-top-right-radius: 1rem;
            font-size: 1rem;
            text-transform: uppercase;
            cursor: pointer;
            transition: all 0.2s ease;

            &.active {
                background-color: var(--primary-color);
            }

            &:hover {
                background-color: rgba($color: #41b883, $alpha: 0.4);
            }
        }
    }

    .tab-container {
        padding: 1rem 2rem;
        border: 3px solid var(--primary-color);

        p.main {
            font-size: 1.5rem;
            margin: 1rem 0;

            i {
                font-size: 1.5rem;
            }
        }

        ul.errors {
            li {
                margin: 0.75rem 0;
                font-size: 1.2rem;
                text-transform: capitalize;

                i {
                    margin-right: 1rem;
                }
            }
        }

        .box-error {
            margin-bottom: 2rem;

            p {
                font-size: 1.2rem;
                text-transform: capitalize;
                margin-bottom: 0.5rem;
            }

            img {
                width: 500px;
            }

            span.error-time {
                color: rgb(85, 149, 171);
                cursor: pointer;
            }

            hr {
                background-color: var(--primary-color);
                color: var(--primary-color);
            }
        }

        .video-container {
            width: 80%;
            margin-inline: auto;
            display: flex;
            justify-content: center;
            align-items: center;
        }
    }

    .error-color {
        color: red;
    }

    .info-color {
        color: rgb(55, 194, 55);
    }

    .chatgpt-messages {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .chatgpt-message {
        padding: 1rem;
        background-color: #f9f9f9;
        border: 1px solid var(--primary-color);
        border-radius: 8px;
    }
}
</style>

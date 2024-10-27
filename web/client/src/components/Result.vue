<script setup>
import { ref, computed, onMounted } from "vue";
import OpenAI from "openai";

import Video from "./Video.vue";

import DOMPurify from 'dompurify';

const { data } = defineProps(["data"]);
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

const jumpToVideoLocation = (second) => {
    selectedDisplay.value = "video";
    videoStart.value = second;
};

// ChatGPT
const chatGptMessage = ref("Before Loaded");
// const APIKEY = import.meta.env.VITE_OPENAI_API_KEY;
// console.log('APIKEY :', APIKEY);
const callChatGPT = async (summaryData) => {
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
                    Please print out each sentence by adding one sentence translated into Korean right below.

                    Please provide feedback on each error in the following JSON format:

                    [
                      {
                        "error": "Error description",
                        "timestamp": "Timestamp",
                        "issue": "Explanation of the issue",
                        "why_bad": "Why it's bad",
                        "fix": "How to fix it"
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
};

function extractHTMLContent(text) {
    const regex = /```html\n([\s\S]*?)```/;
    const match = text.match(regex);
    if (match && match[1]) {
        const htmlContent = match[1];

        // Extract <style> content
        const styleRegex = /<style[^>]*>([\s\S]*?)<\/style>/i;
        const styleMatch = htmlContent.match(styleRegex);
        let styleContent = '';
        if (styleMatch && styleMatch[1]) {
            styleContent = styleMatch[1];
        }

        // Extract <body> content
        const bodyRegex = /<body[^>]*>([\s\S]*?)<\/body>/i;
        const bodyMatch = htmlContent.match(bodyRegex);
        let bodyContent = '';
        if (bodyMatch && bodyMatch[1]) {
            bodyContent = bodyMatch[1];
        } else {
            // If no <body> tag, remove <style> and use the remaining content
            bodyContent = htmlContent.replace(styleRegex, '');
        }

        return { bodyContent, styleContent };
    } else {
        // If no HTML code block, return the original text
        return { bodyContent: text, styleContent: '' };
    }
}

function parseChatGPTResponse(content) {
  const items = content.split(/\n(?=\d+\.\s)/g);
  return items.map(item => item.trim()).filter(item => item.length > 0);
}

onMounted(async () => {
    await callChatGPT(summaryData.value);
});


</script>

<template>
    <section class="result-section">
        <!-- Navigators -->
        <ul class="tab-links">
            <li
                :class="{ active: selectedDisplay == 'chatgpt' }"
                @click="() => (selectedDisplay = 'chatgpt')"
            >
                AI's Feedback
            </li>
            <li
                :class="{ active: selectedDisplay == 'summary' }"
                @click="() => (selectedDisplay = 'summary')"
            >
                Summary
            </li>
            <li
                :class="{ active: selectedDisplay == 'detail' }"
                @click="() => (selectedDisplay = 'detail')"
            >
                Detail
            </li>
            <li
                :class="{ active: selectedDisplay == 'video' }"
                @click="() => (selectedDisplay = 'video')"
            >
                Full Video
            </li>
        </ul>

        <!-- Contents -->
        <div class="tab-container">
            <!-- ChatGPT content -->
            <template v-if="selectedDisplay == 'chatgpt'">
                <div class="chatgpt-messages">
                    <div
                        class="chatgpt-message"
                        v-for="(message, index) in chatGptMessage"
                        :key="index"
                    >
                        <h3>{{ index + 1 }}. {{ message.error }} (Timestamp: {{ message.timestamp }})</h3>
                        <p><strong>Issue:</strong> {{ message.issue }}</p>
                        <p><strong>Why It's Bad:</strong> {{ message.why_bad }}</p>
                        <p><strong>Fix:</strong> {{ message.fix }}</p>
                    </div>
                </div>
            </template>

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
                    <li v-for="(total, error) in summaryData.details">
                        <i class="fa-solid fa-caret-right"></i>

                        {{ error }}: {{ total }}
                    </li>
                </ul>
            </template>

            <!-- Detail Content -->
            <KeepAlive>
                <template v-if="selectedDisplay == 'detail'">
                    <div
                        class="box-error"
                        v-for="(error, index) in data.details"
                    >
                        <p>
                            {{ index + 1 }}. {{ error.stage }} at
                            <span
                                class="error-time"
                                @click="jumpToVideoLocation(error.timestamp)"
                            >
                                {{ error.timestamp }} second
                            </span>
                        </p>
                        <img :src="`${error.frame}`" />
                        <hr />
                        <div v-if="chatGptMessage && chatGptMessage.length > index">
                            <p><strong>Issue:</strong> {{ chatGptMessage[index].issue }}</p>
                            <p><strong>Why It's Bad:</strong> {{ chatGptMessage[index].why_bad }}</p>
                            <p><strong>Fix:</strong> {{ chatGptMessage[index].fix }}</p>
                        </div>
                    </div>
                </template>
            </KeepAlive>

            <!-- Full Video content -->
            <KeepAlive>
                <template v-if="selectedDisplay == 'video'">
                    <div class="video-container">
                        <Video
                            :video-name="data.file_name"
                            :start-at="videoStart"
                        />
                    </div>
                </template>
            </KeepAlive>
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

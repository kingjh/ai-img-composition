<template>
  <v-app>
    <v-main class="container">
      <h1 class="text-center mb-4">AI看图作文</h1>
      <v-file-input
        accept="image/*"
        label="选择图片"
        @change="onFileSelected"
        class="file-input"
        chips
        :prepend-icon="false"
      />
      <div v-if="selectedImage" class="thumbnail-container">
        <v-img :src="selectedImage" max-height="100px" contain @click="enlargeImage = true"></v-img>
      </div>
      <v-dialog v-model="enlargeImage" fullscreen>
        <v-card>
          <v-img :src="selectedImage" contain></v-img>
          <v-btn @click="enlargeImage = false" class="close-button">关闭</v-btn>
        </v-card>
      </v-dialog>
      <v-textarea class="mt-12"
        v-model="prompt"
        label="请输入提示词"
        placeholder="如'以李白风格描述'"
      ></v-textarea>
      <div>
        <v-btn
            color="primary"
            @click="generateComposition"
            :disabled="!selectedImage || loading"
            class="generate-button"
        >{{ loading ? `持续写了${elapsedTime}秒` : '写作！' }}</v-btn>
        <div v-if="finalComposition" class="composition-text mt-12 mb-8">{{ finalComposition }}</div>
        <v-btn
            v-if="finalComposition"
            color="secondary"
            @click="copyComposition"
            class="copy-button"
        >复制文本</v-btn>
      </div>
    </v-main>
  </v-app>
</template>

<script setup>
import { ref } from 'vue';
import axios from 'axios';
import { VApp, VMain, VFileInput, VTextarea, VBtn, VImg, VDialog, VCard } from 'vuetify/components';

const selectedImage = ref(null);
const imageDescription = ref('');
const prompt = ref('以李白风格描述');
const finalComposition = ref('');
const loading = ref(false);
const enlargeImage = ref(false);
const elapsedTime = ref(0);
let timerInterval = null;

const onFileSelected = (event) => {
  const file = event.target.files[0];
  if (!file) {
    selectedImage.value = null;
    return;
  }
  const reader = new FileReader();
  reader.onload = (e) => {
    selectedImage.value = e.target.result;
  };
  reader.readAsDataURL(file);
};

const describeImage = async () => {
  const fileInput = document.querySelector('input[type="file"]');
  const file = fileInput.files[0];

  const url = `https://gai.cfworker.cfd/`;
  const mimeType = file.type;
  const base64Data = await new Promise((resolve) => {
    const reader = new FileReader();
    reader.onloadend = () => resolve(reader.result.split(',')[1]);
    reader.readAsDataURL(file);
  });

  const parts = [
    {
      text: '请描述图片，包括时间、环境、人物等元素，所有元素的大小、颜色等特征',
    },
    {
      inlineData: {
        mimeType,
        data: base64Data,
      },
    },
  ];

  const data = {
    contents: [{ role: 'user', parts }],
  };

  try {
    const response = await axios.post(url, data, { timeout: 600000 });
    return response.data.candidates[0].content.parts[0].text;
  } catch (error) {
    console.error('Error describing image:', error);
    return 'Failed to describe image.';
  }
};

function removeTags(input) {
  return input.replace(/<reasoning>.*?<\/reasoning>|<think>.*?<\/think>/gs, '').trim();
}

const rewriteDescription = async (description) => {
  try {
    const response = await axios.post(
      'https://dsai.cfworker.cfd/',
      {
        // 4个网站：deepseek, siliconFlow, nvidia, fireworks
        site: 'fireworks',
        messages: [
          {
            role: 'system',
            content: '你是文学大师，精通古今中外、书本、影视作品里的所有文学，精通著名人物的写作、语言、表演风格',
          },
          {
            role: 'user',
            content: `根据提示词：\`${prompt.value}\`，重写以下描述，只输出最终文字(和标题，如果有标题的话)，不要输出思考过程，不要输出额外的说明: \`${description}\``,
          },
        ],
      },
      { timeout: 600000 }
    );
    return removeTags(response.data.choices[0].message.content);
  } catch (error) {
    console.error('Error rewriting description:', error);
    return '写作失败，请稍后重试。';
  }
};

const generateComposition = async () => {
  loading.value = true;
  elapsedTime.value = 0;
  timerInterval = setInterval(() => {
    elapsedTime.value++;
  }, 1000);
  imageDescription.value = '';
  finalComposition.value = '';
  const description = await describeImage();
  imageDescription.value = description;
  finalComposition.value = await rewriteDescription(description);
  loading.value = false;
  clearInterval(timerInterval);
};

const copyComposition = () => {
  const textarea = document.createElement('textarea');
  textarea.value = finalComposition.value;
  textarea.style.position = 'fixed';
  textarea.style.opacity = 0;
  document.body.appendChild(textarea);

  try {
    textarea.select();
    const successful = document.execCommand('copy');
    if (successful) {
      alert('作文已复制到剪贴板！');
    } else {
      alert('复制失败，请手动复制作文内容。');
    }
  } catch (err) {
    console.error('复制文本时发生错误:', err);
    alert('复制文本时发生未知错误，请手动复制。');
  } finally {
    document.body.removeChild(textarea);
  }
};
</script>

<style scoped>
.container {
  gap: 20px;
  padding: 20px;
}

.generate-button,
.copy-button,
.file-input {
  font-size: 1rem;
  border-radius: 4px;
}

.generate-button,
.copy-button {
  display: block;
  margin: auto;
}

.composition-text {
  padding: 20px;
  white-space: pre-wrap;
  word-wrap: break-word;
  max-width: 100%;
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-family: monospace;
  line-height: 1.6;
}

.thumbnail-container {
  margin-top: 10px;
  cursor: pointer;
  gap: 8px;
}

:deep(.v-responsive__sizer) {
  width: 100px;
}

.close-button {
  position: absolute;
  top: 10px;
  right: 10px;
}
</style>

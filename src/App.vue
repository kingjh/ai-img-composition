<template>
  <v-app>
    <v-main class="container">
      <h1 class="text-center">AI计运</h1>

      <v-select
        v-model="gender"
        :items="['男性', '女性']"
        label="性别"
      ></v-select>

      <VueDatePicker
        v-model="birthDateTime"
        :enable-time-picker
        locale="zh-CN"
        placeholder="出生日期时间"
      />

      <v-select
        v-model="province"
        :items="provinces"
        label="省份"
        @update:model-value="onProvinceChange"
      ></v-select>

      <v-select
        v-model="city"
        :items="cities"
        label="城市"
      ></v-select>

      <div>
        <v-btn
          color="primary"
          @click="generateConclusion"
          :disabled="loading"
          class="generate-button"
        >{{ loading ? `持续写了${elapsedTime}秒` : '推算！' }}</v-btn>
        <div v-if="conclusion" class="composition-text mt-12 mb-8">{{ conclusion }}</div>
        <v-btn
          v-if="conclusion"
          color="secondary"
          @click="copyConclusion"
          class="copy-button"
        >复制文本</v-btn>
      </div>
    </v-main>
  </v-app>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import axios from 'axios';
import { VApp, VMain, VSelect, VBtn } from 'vuetify/components';
import VueDatePicker from '@vuepic/vue-datepicker';
import '@vuepic/vue-datepicker/dist/main.css';
import Datepicker from '@vuepic/vue-datepicker'
import '@vuepic/vue-datepicker/dist/main.css'

const gender = ref('');
const birthDateTime = ref(null);
const province = ref('');
const city = ref('');
const prompt = computed(() => {
  if (gender.value && birthDateTime.value && province.value && city.value) {
    const formattedDate = new Date(birthDateTime.value).toLocaleString('zh-CN', {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
      hour: 'numeric',
      minute: 'numeric',
      second: 'numeric',
      hour12: false,
    });
    return `${gender.value}，${formattedDate}出生于${province.value}${city.value}`;
  }
  return '';
});
const conclusion = ref('');
const loading = ref(false);
const elapsedTime = ref(0);
let timerInterval = null;

const provinces = ref([]);
const cities = ref([]);

onMounted(async () => {
  // 模拟获取省市数据
  provinces.value = ['北京', '上海', '广东'];
  cities.value = ['广州', '深圳', '珠海'];
});

const onProvinceChange = (newProvince) => {
  // 根据选择的省份更新城市数据
  if (newProvince === '广东') {
    cities.value = ['广州', '深圳', '珠海'];
  } else if (newProvince === '北京') {
    cities.value = ['北京'];
  } else if (newProvince === '上海') {
    cities.value = ['上海'];
  } else {
    cities.value = [];
  }
};

function removeTags(input) {
  return input.replace(/<reasoning>.*?<\/reasoning>|<think>.*?<\/think>/gs, '').trim();
}

const conclude = async () => {
  try {
    const response = await axios.post(
      'https://dsai.cfworker.cfd/',
      {
        // 5个网站：deepseek, siliconFlow, nvidia, fireworks，huoshan
        site: 'huoshan',
        temperature: 0,
        top_p: 0.1,
        messages: [
          {
            role: 'system',
            content: `
你现在是一个中国传统八字命理的专业研究人员你熟读穷通宝典，三命通会，滴天髓，渊海子平这些书籍。你熟读千里命稿，协纪辨方书，果老星宗，子平真栓，神峰通考一系列书籍。
根据"排大运分阳年、阴年。阳年：甲丙戊庚王。阴年：乙丁已辛癸，阳年男，阴年女为顺排，阴年男，阳年女为逆排，具体排法以月干支为基准，进行顺逆，小孩交大运前，以月柱干支为大运十天干：甲乙丙丁戊已庚辛壬癸，十二地支：子丑寅卯辰已午未申酉戌亥。"
根据以上我所提到的书籍，及相关四柱八字的书籍和经验，对命盘的八字进行分析，内容越全面越好。
请输出：
1. 最终正确的结果。
2. 2025年蛇年这个命盘的整体运势如何，有什么机会和风险呀？
不要输出思考过程，不要输出额外的说明，不要用markdown格式，谢谢。
`
          },
          {
            role: 'user',
            content: `分析这个命盘：阳历：\`${prompt.value}\``
          },
        ],
      },
      { timeout: 600000 }
    );
    return removeTags(response.data.choices[0].message.content);
  } catch (error) {
    console.error('Error rewriting description:', error);
    return '推算失败，请稍后重试。';
  }
};

const generateConclusion = async () => {
  loading.value = true;
  elapsedTime.value = 0;
  timerInterval = setInterval(() => {
    elapsedTime.value++;
  }, 1000);
  conclusion.value = await conclude();
  loading.value = false;
  clearInterval(timerInterval);
};

const copyConclusion = () => {
  const textarea = document.createElement('textarea');
  textarea.value = conclusion.value;
  textarea.style.position = 'fixed';
  textarea.style.opacity = 0;
  document.body.appendChild(textarea);

  try {
    textarea.select();
    const successful = document.execCommand('copy');
    if (successful) {
      alert('结论已复制到剪贴板！');
    } else {
      alert('复制失败，请手动复制结论内容。');
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

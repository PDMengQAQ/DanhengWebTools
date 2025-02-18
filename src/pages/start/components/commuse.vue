<template>
  <div>
    <!-- 滚动公告 -->
    <div class="scrolling-notice" v-if="showNotice">
      <marquee behavior="scroll" direction="left">{{ t('main.notice') }}</marquee>
    </div>

    <!-- 原有的组件内容 -->
    <div class="commuse">

      <div class="commuse-item">
        <div class="text-neutral-400 dark:text-slate-100">{{ t('commuse.item') }}</div>
        <a-cascader allow-search v-model="value2" :options="options" placeholder="{{ t('main.item') }}" filterable />
      </div>

      <div class="commuse-item">
        <div class="text-stone-700 dark:text-slate-100">{{ t('commuse.number') }}</div>
        <a-input-number v-model="num" placeholder="{{ t('main.number') }}" mode="button" size="large" class="input-demo" />
      </div>

      <div class="generate">
        <a-input v-model="value" placeholder="" />
        <a-button type="primary" shape="round" size="large" @click="copyvalue">{{ t('main.copy') }}</a-button>
        <a-button type="primary" shape="round" size="large" @click="execute">执行</a-button>
      </div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { useClipboard } from '@vueuse/core';
import { Message } from '@arco-design/web-vue';
import { useAppStore } from '@/store/modules/app';
import { useI18n } from 'vue-i18n';
import axios from 'axios';
import JSEncrypt from 'jsencrypt';

const appStore = useAppStore();
const { text, isSupported, copy } = useClipboard();
const { t, locale } = useI18n();

const value2 = ref(1);
const num = ref(1000);

const value = computed(() => {
  return `give ${value2.value} x${num.value}`;
});

const message = Message;

const options = ref([] as { label: string; value: number }[]);

function copyvalue() {
  copy(value.value);
  if (isSupported) {
    message.success(t('main.success', { value: value.value }));
  }
}

const ADMIN_KEY = import.meta.env.VITE_DANHENG_ADMIN_KEY;
const API_BASE_URL = import.meta.env.VITE_DANHENG_DISPATCH_SERVER;

const execute = async () => {
  const uid = localStorage.getItem('lastSubmittedUid');

  if (!uid) {
    message.info('用户未登录,请先前往”远程“页面执行一次命令，然后重试');
    return;
  }

  const command = value.value;

  try {
    // Step 1: 获取授权
    const authRes = await axios.post(`${API_BASE_URL}/muip/auth_admin`, {
      admin_key: ADMIN_KEY,
      key_type: 'PEM'
    });

    if (authRes.data.code !== 0) {
      throw new Error('授权失败: ' + authRes.data.message);
    }

    const { rsaPublicKey, sessionId } = authRes.data.data;

    // Step 2: 使用rsaPublicKey加密命令
    const encrypt = new JSEncrypt();
    encrypt.setPublicKey(rsaPublicKey);
    const encryptedCommand = encrypt.encrypt(command);

    if (!encryptedCommand) {
      throw new Error('命令加密失败');
    }

    // Step 3: 提交命令
    const execCmdRes = await axios.post(`${API_BASE_URL}/muip/exec_cmd`, {
      SessionId: sessionId,
      Command: encryptedCommand,
      TargetUid: uid
    });

    if (execCmdRes.data.code !== 0) {
      throw new Error('命令提交失败: ' + execCmdRes.data.message);
    }

    const decodedMessage = base64Decode(execCmdRes.data.data.message);

    message.success(`命令提交成功：${decodedMessage}`);
  } catch (err: unknown) {
    const errorMessage = err instanceof Error ? err.message : '请求失败';
    message.error(errorMessage);
    console.error(err);
  }
};

const base64Decode = (str: string): string => {
  try {
    return decodeURIComponent(escape(window.atob(str)));
  } catch (e) {
    console.error('Base64解码失败:', e);
    return '解码失败';
  }
};

const showNotice = ref(true);

onMounted(() => {
  locale.value = navigator.language.includes('zh') ? 'zh' : 'en';

  import('./json/zh/commuse.json').then((commusezh) => {
    import('./json/en/commuse.json').then((commuseen) => {
      options.value = locale.value === 'zh' ? commusezh.default : commuseen.default;

      setTimeout(() => {
        showNotice.value = true;
      }, 1000);
    });
  });
});
</script>

<style lang="less" scoped>
/* 确保滚动公告的样式 */
.scrolling-notice {
  color: #BEBEBE;
  padding: 8px;
  font-size: 14px;
  text-align: center;
  white-space: nowrap;
  overflow: hidden;
  border-radius: 10px;
}

.commuse {
  width: 500px;
  margin: auto;
}

.commuse-item {
  display: flex;
  align-items: center;
  color: #000;
  margin: 18px 0;

  > div {
    &:nth-child(1) {
      width: 150px;
      text-align: right;
      padding-right: 10px;
      color: #6b6a6a !important;  /* 使用 !important 提高优先级 */
    }
  }
}

.generate {
  display: flex;
  align-items: center;
  margin-left: 100px;
}

@media screen and (max-width: 768px) {
  .commuse {
    width: 100%; 
    padding: 10px; 
  }

  .commuse-item {
    margin: 18px 0 10px; 
  }

  .commuse-item > div:nth-child(1) {
    width: auto; 
    text-align: left; 
    padding: 0; 
    margin-bottom: 5px; 
  }

  .generate {
    display: block; 
    margin-left: 0; 
    width: 100%; 
    margin-bottom: 80px; 
    margin-top: 10px; 
    text-align: center; 
  }

  .generate > .arco-input {
    margin-bottom: 10px; 
  }

  .generate button { 
    display: block; 
    width: 100%; 
    margin-top: 10px; 
  }
}
</style>

<template>
  <div class="app-container">
    <!-- 顶部标题栏 -->
    <div class="header">
      <div class="header-left">
        <h1 class="app-title">Qwen-MT翻译</h1>
      </div>
      <div class="header-right">
        <div
          class="status-indicator"
          @click="showConfigModal = true"
          :class="{ unconfigured: !apiKey }"
        >
          <span class="status-dot" :class="{ unconfigured: !apiKey }"></span>
          {{ apiKey ? "API已配置" : "API未配置" }}
        </div>
        <div class="lang-switch">
          <select v-model="sourceLang" class="lang-select">
            <option v-for="lang in languages" :key="lang.value" :value="lang.value">
              {{ lang.label }}
            </option>
          </select>
          <span class="arrow">→</span>
          <select v-model="targetLang" class="lang-select">
            <template v-for="lang in languages">
              <option :key="lang.value" :value="lang.value" v-if="lang.value !== 'auto'">
                {{ lang.label }}
              </option>
            </template>
          </select>
        </div>
      </div>
    </div>

    <!-- 对话区域 -->
    <div class="chat-area">
      <div v-if="conversations.length === 0" class="empty-state">
        <div class="empty-icon">💬</div>
        <p>开始您的翻译对话</p>
      </div>
      <div v-else class="conversation-list">
        <div
          v-for="(conv, index) in conversations"
          :key="index"
          class="conversation-item"
        >
          <!-- 用户输入 -->
          <div class="message user-message">
            <div class="message-content">{{ conv.input }}</div>
            <div class="message-time">
              {{ conv.time }}
              <button
                class="copy-btn"
                @click="copyToClipboard(conv.input)"
                :disabled="!conv.input"
              >
                复制
              </button>
            </div>
          </div>
          <!-- AI翻译结果 -->
          <div class="message ai-message">
            <div class="message-content">{{ conv.result || "正在翻译..." }}</div>
            <div class="message-time">
              {{ conv.time }}
              <button
                class="copy-btn"
                @click="copyToClipboard(conv.result)"
                :disabled="!conv.result"
              >
                复制
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 底部输入区域 -->
    <div class="input-area">
      <div class="input-container">
        <textarea
          v-model="inputText"
          @keydown.enter.exact.prevent="handleTranslate"
          @keydown.enter.shift="insertNewline"
          placeholder="输入要翻译的文本..."
          class="input-textarea"
          rows="1"
          :style="{ maxHeight: '96px', overflowY: 'auto' }"
          ref="textareaRef"
        ></textarea>
        <div class="input-actions">
          <button @click="handleTranslate" class="send-btn" :disabled="!inputText.trim()">
            <template v-if="isTranslating">⏳</template>
            <template v-else
              ><svg
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 24 24"
                fill="currentColor"
                aria-hidden="true"
                data-slot="icon"
                class="w-4 h-4"
              >
                <path
                  d="M3.478 2.404a.75.75 0 0 0-.926.941l2.432 7.905H13.5a.75.75 0 0 1 0 1.5H4.984l-2.432 7.905a.75.75 0 0 0 .926.94 60.519 60.519 0 0 0 18.445-8.986.75.75 0 0 0 0-1.218A60.517 60.517 0 0 0 3.478 2.404Z"
                ></path></svg
            >
            </template>
          </button>
        </div>
      </div>
      <div class="input-tip">按 Enter 发送，Shift + Enter 换行</div>
    </div>

    <!-- API配置模态框 -->
    <div v-if="showConfigModal" class="modal-overlay" @click="showConfigModal = false">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h3>API配置</h3>
          <button @click="showConfigModal = false" class="close-btn">×</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label for="apiKey">API Key</label>
            <input
              v-model="tempApiKey"
              type="password"
              id="apiKey"
              placeholder="请输入您的API Key"
              class="form-input"
            />
          </div>
          <div class="form-group">
            <label for="model">选择模型</label>
            <select v-model="tempModel" id="model" class="form-select">
              <option value="qwen-mt-turbo">qwen-mt-turbo</option>
              <option value="qwen-mt-plus">qwen-mt-plus</option>
            </select>
          </div>
        </div>
        <div class="modal-footer">
          <button @click="showConfigModal = false" class="btn-cancel">取消</button>
          <button @click="saveConfig" class="btn-save">保存</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, nextTick } from "vue";
import { useClipboard } from "@vueuse/core";

const { copy } = useClipboard({
  legacy: true,
});

interface Conversation {
  input: string;
  result: string;
  time: string;
}

const inputText = ref("");
const sourceLang = ref("auto");
const targetLang = ref("zh");
const isTranslating = ref(false);
const conversations = ref<Conversation[]>([]);

// API配置相关
const showConfigModal = ref(false);
const apiKey = ref("");
const selectedModel = ref("qwen-mt-turbo");
const tempApiKey = ref("");
const tempModel = ref("qwen-mt-turbo");

const languages = [
  { value: "auto", label: "auto" },
  { value: "zh", label: "Chinese" },
  { value: "en", label: "English" },
  { value: "ja", label: "日本語" },
  { value: "fr", label: "Français" },
  { value: "de", label: "Deutsch" },
  { value: "es", label: "Español" },
  { value: "ru", label: "Русский" },
  { value: "ko", label: "한국어" },
  { value: "it", label: "Italiano" },
  { value: "pt", label: "Português" },
];

// 保存API配置
const saveConfig = () => {
  apiKey.value = tempApiKey.value;
  selectedModel.value = tempModel.value;
  showConfigModal.value = false;

  // 保存到本地存储
  localStorage.setItem("qwen-translation-api-key", apiKey.value);
  localStorage.setItem("qwen-translation-model", selectedModel.value);
};

// 从本地存储加载配置
const loadConfig = () => {
  const savedApiKey = localStorage.getItem("qwen-translation-api-key");
  const savedModel = localStorage.getItem("qwen-translation-model");

  if (savedApiKey) {
    apiKey.value = savedApiKey;
    tempApiKey.value = savedApiKey;
  }

  if (savedModel) {
    selectedModel.value = savedModel;
    tempModel.value = savedModel;
  }
};

// 组件挂载时加载配置
loadConfig();

const handleTranslate = async () => {
  if (!inputText.value.trim() || isTranslating.value) return;

  // 检查API配置
  if (!apiKey.value) {
    alert("请先配置API Key");
    showConfigModal.value = true;
    return;
  }

  const currentTime = new Date().toLocaleTimeString("zh-CN", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });

  const newConversation: Conversation = {
    input: inputText.value,
    result: "",
    time: currentTime,
  };

  conversations.value.push(newConversation);
  const inputToTranslate = inputText.value;
  inputText.value = "";
  isTranslating.value = true;

  try {
    const response = await fetch(
      "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions",
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${apiKey.value}`,
        },
        body: JSON.stringify({
          model: selectedModel.value,
          messages: [
            // {
            //   role: "system",
            //   content: `你是一个专业的翻译助手。请将用户输入的文本从${getLanguageName(
            //     sourceLang.value
            //   )}翻译成${getLanguageName(
            //     targetLang.value
            //   )}。只需要返回翻译结果，不要添加任何解释或额外内容。`,
            // },
            {
              role: "user",
              content: inputToTranslate,
            },
          ],
          // stream: true, // 开启流式传输
          translation_options: {
            source_lang: sourceLang.value,
            target_lang: targetLang.value,
          },
        }),
      }
    );

    if (!response.ok) {
      throw new Error(`API请求失败: ${response.status} ${response.statusText}`);
    }

    const data = await response.json();

    if (data.choices && data.choices[0] && data.choices[0].message) {
      newConversation.result = data.choices[0].message.content.trim();
    } else {
      throw new Error("API返回格式错误");
    }
  } catch (error) {
    console.error("翻译错误:", error);
    newConversation.result = `翻译失败: ${
      error instanceof Error ? error.message : "未知错误"
    }`;
  } finally {
    isTranslating.value = false;
  }
};

// 复制到剪贴板
const copyToClipboard = (text: string | undefined) => {
  if (!text || typeof text !== "string") return;
  copy(text);
  // .then(() => {
  //   alert("已复制到剪贴板");
  // })
  // .catch((err) => {
  //   console.error("复制失败:", err);
  //   alert("复制失败，请手动复制");
  // });
};

// 获取语言显示名称
// const getLanguageName = (langCode: string) => {
//   const lang = languages.find((l) => l.value === langCode);
//   return lang ? lang.label : langCode;
// };

const textareaRef = ref<HTMLTextAreaElement | null>(null);

// 自动调整高度，最多4行
watch(inputText, async () => {
  await nextTick();
  const el = textareaRef.value;
  if (el) {
    el.style.height = "auto";
    el.style.height = Math.min(el.scrollHeight, 96) + "px"; // 4行高度
  }
});

// Shift+Enter换行
const insertNewline = (e: KeyboardEvent) => {
  const el = textareaRef.value;
  if (el) {
    const start = el.selectionStart;
    const end = el.selectionEnd;
    const value = inputText.value;
    inputText.value = value.slice(0, start) + "\n" + value.slice(end);
    nextTick(() => {
      el.selectionStart = el.selectionEnd = start + 1;
    });
  }
};
</script>

<style scoped>
* {
  box-sizing: border-box;
}

.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: #f5f5f5;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC",
    "Hiragino Sans GB", "Microsoft YaHei", Helvetica, Arial, sans-serif;
}

/* 顶部标题栏 */
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 24px;
  background: #fff;
  border-bottom: 1px solid #e1e5e9;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.app-icon {
  font-size: 24px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  width: 32px;
  height: 32px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.app-title {
  font-size: 18px;
  font-weight: 600;
  margin: 0;
  color: #1f2937;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.status-indicator {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  color: #16a34a;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
  transition: background 0.2s;
}

.status-indicator:hover {
  background: #f0f9ff;
}

.status-indicator.unconfigured {
  color: #ef4444;
}

.status-dot {
  width: 8px;
  height: 8px;
  background: #16a34a;
  border-radius: 50%;
}

.status-dot.unconfigured {
  background: #ef4444;
}

.lang-switch {
  display: flex;
  align-items: center;
  gap: 8px;
}

.lang-select {
  padding: 6px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  background: #fff;
  outline: none;
  cursor: pointer;
}

.arrow {
  color: #6b7280;
  font-weight: bold;
}

.settings-btn {
  padding: 8px 16px;
  background: #f3f4f6;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  color: #374151;
  transition: background 0.2s;
}

.settings-btn:hover {
  background: #e5e7eb;
}

/* 对话区域 */
.chat-area {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
  background: #f5f5f5;
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #9ca3af;
}

.empty-icon {
  font-size: 48px;
  margin-bottom: 16px;
}

.conversation-list {
  max-width: 800px;
  margin: 0 auto;
}

.conversation-item {
  margin-bottom: 32px;
}

.message {
  max-width: 70%;
  margin-bottom: 12px;
}

.user-message {
  margin-left: auto;
}

.user-message .message-content {
  background: #3b82f6;
  color: white;
  padding: 12px 16px;
  border-radius: 18px 18px 4px 18px;
  font-size: 15px;
  line-height: 1.4;
}

.ai-message .message-content {
  background: #fff;
  color: #1f2937;
  padding: 12px 16px;
  border-radius: 18px 18px 18px 4px;
  font-size: 15px;
  line-height: 1.4;
  border: 1px solid #e5e7eb;
}

.message-time {
  font-size: 12px;
  color: #9ca3af;
  margin-top: 4px;
  text-align: right;
}

.ai-message .message-time {
  text-align: left;
}

/* 底部输入区域 */
.input-area {
  padding: 16px 24px 24px;
  background: #fff;
  border-top: 1px solid #e1e5e9;
}

.input-container {
  max-width: 800px;
  margin: 0 auto;
  display: flex;
  align-items: flex-end;
  gap: 12px;
  background: #f8fafc;
  border: 1px solid #e1e5e9;
  border-radius: 12px;
  padding: 12px;
}

.input-textarea {
  flex: 1;
  border: none;
  background: transparent;
  resize: none;
  outline: none;
  font-size: 15px;
  line-height: 1.5;
  max-height: 120px;
  min-height: 20px;
}

.input-textarea::placeholder {
  color: #9ca3af;
}

.input-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.action-btn {
  width: 32px;
  height: 32px;
  border: none;
  background: transparent;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.action-btn:hover {
  background: #e5e7eb;
}

.send-btn {
  width: 32px;
  height: 32px;
  border: none;
  background: #3b82f6;
  color: white;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.send-btn:hover:not(:disabled) {
  background: #2563eb;
}

.send-btn:disabled {
  background: #d1d5db;
  cursor: not-allowed;
}

.input-tip {
  text-align: center;
  font-size: 12px;
  color: #9ca3af;
  margin-top: 8px;
}

/* 模态框样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: #fff;
  border-radius: 12px;
  width: 90%;
  max-width: 500px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 24px;
  border-bottom: 1px solid #e5e7eb;
}

.modal-header h3 {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
  color: #1f2937;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  color: #6b7280;
  cursor: pointer;
  padding: 0;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: background 0.2s;
}

.close-btn:hover {
  background: #f3f4f6;
}

.modal-body {
  padding: 24px;
}

.form-group {
  margin-bottom: 20px;
}

.form-group:last-child {
  margin-bottom: 0;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  font-size: 14px;
  font-weight: 500;
  color: #374151;
}

.form-input,
.form-select {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
}

.form-input:focus,
.form-select:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding: 20px 24px;
  border-top: 1px solid #e5e7eb;
  background: #f9fafb;
  border-radius: 0 0 12px 12px;
}

.btn-cancel,
.btn-save {
  padding: 8px 16px;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-cancel {
  background: #fff;
  border: 1px solid #d1d5db;
  color: #374151;
}

.btn-cancel:hover {
  background: #f9fafb;
}

.btn-save {
  background: #3b82f6;
  border: 1px solid #3b82f6;
  color: #fff;
}

.btn-save:hover {
  background: #2563eb;
  border-color: #2563eb;
}

.copy-btn {
  margin-left: 8px;
  padding: 2px 8px;
  font-size: 12px;
  border: none;
  border-radius: 4px;
  background: #f3f4f6;
  color: #3b82f6;
  cursor: pointer;
  transition: background 0.2s;
}

.copy-btn:hover {
  background: #e0e7ef;
}
</style>

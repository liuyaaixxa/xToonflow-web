<template>
  <t-card :bordered="false" class="ai-creation-card">
    <template #header>
      <div class="card-header">
        <div class="header-left">
          <i-magic :size="20" class="header-icon" />
          <span class="header-title">AI 创作助手</span>
        </div>
        <t-button v-if="isCreating" theme="danger" variant="outline" size="small" @click="handleStop">
          <template #icon><i-off-switch :size="16" /></template>
          停止
        </t-button>
      </div>
    </template>

    <div class="creation-content">
      <!-- 引导字段（可选） -->
      <div class="guided-fields">
        <t-row :gutter="16">
          <t-col :span="6">
            <t-form-item label="题材">
              <t-input v-model="formData.genre" placeholder="如：仙侠、都市、科幻" :disabled="isCreating" />
            </t-form-item>
          </t-col>
          <t-col :span="6">
            <t-form-item label="主角">
              <t-input v-model="formData.protagonist" placeholder="主角名称" :disabled="isCreating" />
            </t-form-item>
          </t-col>
          <t-col :span="6">
            <t-form-item label="背景">
              <t-input v-model="formData.setting" placeholder="故事背景" :disabled="isCreating" />
            </t-form-item>
          </t-col>
          <t-col :span="6">
            <t-form-item label="核心冲突">
              <t-input v-model="formData.conflict" placeholder="核心矛盾" :disabled="isCreating" />
            </t-form-item>
          </t-col>
        </t-row>
        <t-row :gutter="16">
          <t-col :span="6">
            <t-form-item label="生成章节数">
              <t-input-number v-model="formData.chapterCount" :min="1" :max="20" theme="normal" style="width: 100%" :disabled="isCreating" />
            </t-form-item>
          </t-col>
          <t-col :span="6">
            <t-form-item label="引用资产">
              <t-switch v-model="formData.useRefAssets" :disabled="isCreating" />
            </t-form-item>
          </t-col>
        </t-row>
      </div>

      <!-- 自由输入框 -->
      <div class="prompt-input">
        <t-textarea
          v-model="formData.prompt"
          placeholder="请描述你想创作的故事内容，AI 将根据你的描述生成小说章节..."
          :autosize="{ minRows: 3, maxRows: 6 }"
          :disabled="isCreating"
        />
      </div>

      <!-- 操作按钮 -->
      <div class="action-buttons">
        <t-button theme="primary" @click="handleCreate" :disabled="!canCreate" :loading="isCreating">
          <template #icon><i-magic :size="16" /></template>
          {{ isCreating ? '创作中...' : '开始创作' }}
        </t-button>
        <t-button
          v-if="lastChapterId"
          theme="default"
          variant="outline"
          @click="handleContinue"
          :disabled="!formData.prompt.trim() || isCreating"
        >
          <template #icon><i-arrow-right :size="16" /></template>
          续写
        </t-button>
      </div>

      <!-- 流式输出区域 -->
      <div v-if="streamContent || chapterList.length" class="output-area">
        <div class="output-header">
          <span class="output-title">创作输出</span>
          <t-button v-if="chapterList.length" theme="default" size="small" variant="text" @click="handleClearOutput">
            清空
          </t-button>
        </div>

        <!-- 已完成章节列表 -->
        <div v-if="chapterList.length" class="chapter-list">
          <t-tag
            v-for="chapter in chapterList"
            :key="chapter.chapterIndex"
            theme="success"
            variant="light"
            class="chapter-tag"
          >
            第{{ chapter.chapterIndex }}章: {{ chapter.chapter }}
          </t-tag>
        </div>

        <!-- 当前流式内容 -->
        <div v-if="streamContent" class="stream-content">
          <div class="stream-text">{{ streamContent }}</div>
          <div class="stream-cursor">
            <span class="cursor"></span>
          </div>
        </div>
      </div>
    </div>
  </t-card>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";
import { MessagePlugin } from "tdesign-vue-next";
import { storeToRefs } from "pinia";
import store from "@/stores";
import WsClient from "@/utils/wsClient";
import axios from "@/utils/axios";

const { projectId } = storeToRefs(store());

interface CreateParams {
  prompt: string;
  genre?: string;
  protagonist?: string;
  setting?: string;
  conflict?: string;
  chapterCount: number;
  useRefAssets: boolean;
}

interface ChapterData {
  chapterIndex: number;
  chapter: string;
  chapterData: string;
}

const emit = defineEmits<{
  (e: "refresh"): void;
}>();

// 表单数据
const formData = ref<CreateParams>({
  prompt: "",
  genre: "",
  protagonist: "",
  setting: "",
  conflict: "",
  chapterCount: 3,
  useRefAssets: false,
});

// 状态
const isCreating = ref(false);
const streamContent = ref("");
const chapterList = ref<ChapterData[]>([]);
const lastChapterId = ref<number | null>(null);

// WebSocket
let ws: WsClient | null = null;
let pendingMessage: any = null;
let wsInitialized = false;

// 计算属性
const canCreate = computed(() => {
  return formData.value.prompt.trim() && !isCreating.value;
});

// WebSocket 初始化
function initWsClient() {
  ws = new WsClient(`/novel/createNovel?projectId=${projectId.value}`, {
    onOpen: () => {
      console.log("[NovelCreation] WebSocket 连接已建立");
    },
    onMessage: handleWsMessage,
    onError: () => {
      isCreating.value = false;
      MessagePlugin.error("WebSocket 连接异常");
      ws = null;
    },
    onClose: () => {
      isCreating.value = false;
      wsInitialized = false;
      pendingMessage = null;
      ws = null;
    },
  });
}

// WebSocket 消息处理
function handleWsMessage(msg: string) {
  let msgData: any;
  try {
    msgData = JSON.parse(msg);
  } catch (e) {
    console.error("[NovelCreation] JSON 解析失败:", e);
    return;
  }

  switch (msgData.type) {
    case "init":
      wsInitialized = true;
      if (pendingMessage) {
        ws?.send(pendingMessage);
        pendingMessage = null;
      }
      break;

    case "stream":
      streamContent.value += msgData.data;
      break;

    case "chapter":
      chapterList.value.push(msgData.data);
      lastChapterId.value = msgData.data.id;
      streamContent.value = "";
      emit("refresh");
      break;

    case "complete":
      isCreating.value = false;
      streamContent.value = "";
      MessagePlugin.success(msgData.data?.message || "创作完成");
      emit("refresh");
      break;

    case "error":
      isCreating.value = false;
      MessagePlugin.error(msgData.data);
      break;

    case "notice":
      MessagePlugin.info(msgData.data);
      break;

    default:
      console.log("[NovelCreation] 未知消息类型:", msgData.type);
  }
}

// 发送 WebSocket 消息
function sendWs(data: any) {
  if (!ws || ws.ws?.readyState !== WebSocket.OPEN) {
    pendingMessage = data;
    wsInitialized = false;
    initWsClient();
  } else if (!wsInitialized) {
    pendingMessage = data;
  } else {
    ws.send(data);
  }
}

// 开始创作
async function handleCreate() {
  if (!canCreate.value) return;

  isCreating.value = true;
  streamContent.value = "";
  chapterList.value = [];

  sendWs({
    type: "create",
    data: formData.value,
  });
}

// 续写
async function handleContinue() {
  if (!lastChapterId.value || !formData.value.prompt.trim()) return;

  isCreating.value = true;
  streamContent.value = "";

  sendWs({
    type: "continue",
    data: {
      chapterId: lastChapterId.value,
      prompt: formData.value.prompt,
    },
  });
}

// 停止
function handleStop() {
  ws?.close();
  isCreating.value = false;
  MessagePlugin.warning("已停止创作");
}

// 清空输出
function handleClearOutput() {
  streamContent.value = "";
  chapterList.value = [];
}

// 获取最新章节ID
async function getLatestChapterId() {
  try {
    const res = await axios.post("/novel/getNovel", { projectId: projectId.value });
    const chapters = res.data || [];
    if (chapters.length > 0) {
      const latest = chapters.reduce((prev: any, curr: any) =>
        prev.chapterIndex > curr.chapterIndex ? prev : curr
      );
      lastChapterId.value = latest.id;
    }
  } catch (e) {
    console.error("[NovelCreation] 获取章节失败:", e);
  }
}

// 生命周期
onMounted(() => {
  getLatestChapterId();
});

onUnmounted(() => {
  ws?.close();
});
</script>

<style lang="scss" scoped>
.ai-creation-card {
  margin-bottom: 16px;
  border-radius: 0.75rem;

  .card-header {
    display: flex;
    justify-content: space-between;
    align-items: center;

    .header-left {
      display: flex;
      align-items: center;
      gap: 8px;

      .header-icon {
        color: var(--td-brand-color);
      }

      .header-title {
        font-weight: 600;
        font-size: 1rem;
      }
    }
  }
}

.creation-content {
  .guided-fields {
    margin-bottom: 16px;
    padding: 16px;
    background: var(--td-bg-color-secondarycontainer);
    border-radius: 8px;
  }

  .prompt-input {
    margin-bottom: 16px;
  }

  .action-buttons {
    display: flex;
    gap: 12px;
    margin-bottom: 16px;
  }

  .output-area {
    border: 1px solid var(--td-component-border);
    border-radius: 8px;
    padding: 16px;
    background: var(--td-bg-color-container);

    .output-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 12px;

      .output-title {
        font-weight: 500;
        color: var(--td-text-color-primary);
      }
    }

    .chapter-list {
      display: flex;
      flex-wrap: wrap;
      gap: 8px;
      margin-bottom: 12px;

      .chapter-tag {
        cursor: default;
      }
    }

    .stream-content {
      position: relative;

      .stream-text {
        white-space: pre-wrap;
        line-height: 1.8;
        color: var(--td-text-color-primary);
      }

      .stream-cursor {
        display: inline-block;

        .cursor {
          display: inline-block;
          width: 2px;
          height: 1em;
          background: var(--td-brand-color);
          animation: blink 1s infinite;
          vertical-align: middle;
          margin-left: 2px;
        }
      }
    }
  }
}

@keyframes blink {
  0%, 50% {
    opacity: 1;
  }
  51%, 100% {
    opacity: 0;
  }
}
</style>

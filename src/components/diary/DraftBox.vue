<script setup lang="ts">
import { ref, computed } from 'vue'
import { useDiaryStore } from '@/stores/diary'
import { useUserStore } from '@/stores/user'
import { pluginLoader } from '@/engine/PluginLoader'
import { globalTimeline } from '@/engine/Timeline'
import type { DiaryDraft } from '@/types'

const emit = defineEmits<{
  close: []
  editDraft: [draftId: string]
}>()

const diaryStore = useDiaryStore()
const userStore = useUserStore()

const deletingDraftId = ref<string | null>(null)
const publishingDraftId = ref<string | null>(null)

const drafts = computed(() => diaryStore.currentUserDrafts)

const diaryTypes = computed(() => {
  const types = pluginLoader.getDiaryTypes()
  return types
})

function getDiaryTypeName(typeId: string): string {
  return diaryTypes.value.get(typeId)?.name || typeId
}

function formatTime(timestamp: number): string {
  const now = globalTimeline.getTime()
  const diff = now - timestamp
  
  if (diff < 60) return '刚刚'
  if (diff < 3600) return `${Math.floor(diff / 60)} 分钟前`
  if (diff < 86400) return `${Math.floor(diff / 3600)} 小时前`
  return `${Math.floor(diff / 86400)} 天前`
}

function getContentPreview(content: string, maxLength: number = 80): string {
  if (content.length <= maxLength) return content
  return content.slice(0, maxLength) + '...'
}

function getTitlePreview(title: string): string {
  return title || '（无标题）'
}

function handleEditDraft(draft: DiaryDraft) {
  emit('editDraft', draft.id)
}

async function handleDeleteDraft(draftId: string) {
  if (!confirm('确定要删除这篇草稿吗？删除后无法恢复。')) return
  
  deletingDraftId.value = draftId
  setTimeout(() => {
    diaryStore.deleteDraft(draftId)
    deletingDraftId.value = null
  }, 300)
}

async function handlePublishDraft(draftId: string) {
  const draft = diaryStore.getDraftById(draftId)
  if (!draft) return
  
  if (!draft.title.trim() || !draft.content.trim()) {
    alert('草稿内容不完整，需要标题和正文才能发布')
    return
  }
  
  if (!confirm('确定要发布这篇草稿吗？发布后将从草稿箱移除。')) return
  
  publishingDraftId.value = draftId
  
  try {
    await diaryStore.publishDraft(draftId)
    publishingDraftId.value = null
    emit('close')
  } catch (e) {
    publishingDraftId.value = null
  }
}

function getScheduleSummary(draft: DiaryDraft): string[] {
  const items: string[] = []
  if (draft.schedule.enablePublishAt) items.push('📅 定时发布')
  if (draft.schedule.enableDecayStartAt) items.push('⏳ 定时腐烂')
  if (draft.schedule.enableAutoArchiveAt) items.push('📦 定时撤回')
  return items
}
</script>

<template>
  <div class="fixed inset-0 bg-black/80 z-50 flex items-center justify-center p-4">
    <div class="bg-gray-900 rounded-lg border-2 border-yellow-500 w-full max-w-3xl max-h-[90vh] overflow-hidden flex flex-col">
      <div class="p-6 border-b border-gray-700">
        <div class="flex items-center justify-between">
          <h2 class="font-vt323 text-2xl text-yellow-400 glow-text">
            📝 草稿箱
          </h2>
          <button
            class="text-gray-400 hover:text-white text-2xl"
            @click="emit('close')"
          >
            ✕
          </button>
        </div>
        <p class="text-gray-400 font-vt323 text-sm mt-1">
          这里保存着你还没写完的日记，随时可以继续编辑或发布
        </p>
      </div>
      
      <div class="flex-1 overflow-y-auto p-6">
        <div v-if="drafts.length === 0" class="text-center py-16">
          <div class="text-6xl mb-4">📭</div>
          <p class="text-gray-500 font-vt323 text-xl">
            草稿箱是空的
          </p>
          <p class="text-gray-600 font-vt323 text-sm mt-2">
            写日记时点击「保存草稿」可以存到这里
          </p>
        </div>
        
        <div v-else class="space-y-4">
          <div
            v-for="draft in drafts"
            :key="draft.id"
            class="bg-gray-800/50 border border-gray-700 rounded-lg p-4 hover:border-yellow-500/50 transition-colors"
          >
            <div class="flex items-start justify-between gap-4">
              <div class="flex-1 min-w-0">
                <div class="flex items-center gap-2 mb-2">
                  <h3 class="font-vt323 text-lg text-white truncate">
                    {{ getTitlePreview(draft.title) }}
                  </h3>
                  <span class="px-2 py-0.5 bg-yellow-500/20 text-yellow-400 text-xs font-vt323 rounded border border-yellow-500/30">
                    草稿
                  </span>
                  <span class="px-2 py-0.5 bg-gray-700 text-gray-300 text-xs font-vt323 rounded">
                    {{ getDiaryTypeName(draft.type) }}
                  </span>
                </div>
                
                <p class="text-gray-400 text-sm font-jetbrains mb-3 line-clamp-2">
                  {{ getContentPreview(draft.content) }}
                </p>
                
                <div class="flex items-center gap-4 flex-wrap">
                  <span class="text-gray-500 text-xs font-vt323">
                    更新于 {{ formatTime(draft.updatedAt) }}
                  </span>
                  
                  <div class="flex items-center gap-2">
                    <span
                      v-for="item in getScheduleSummary(draft)"
                      :key="item"
                      class="text-xs text-gray-500 font-vt323"
                    >
                      {{ item }}
                    </span>
                  </div>
                  
                  <div v-if="draft.selectedMethods.length > 0" class="flex items-center gap-1">
                    <span class="text-gray-500 text-xs font-vt323">烂法:</span>
                    <span
                      v-for="methodId in draft.selectedMethods.slice(0, 3)"
                      :key="methodId"
                      class="px-1.5 py-0.5 bg-gray-700/50 text-gray-400 text-xs font-vt323 rounded"
                    >
                      {{ methodId }}
                    </span>
                    <span
                      v-if="draft.selectedMethods.length > 3"
                      class="text-gray-500 text-xs font-vt323"
                    >
                      +{{ draft.selectedMethods.length - 3 }}
                    </span>
                  </div>
                </div>
              </div>
              
              <div class="flex flex-col gap-2 shrink-0">
                <button
                  class="btn-pixel text-sm px-3 py-1 text-yellow-400 border-yellow-400"
                  @click="handleEditDraft(draft)"
                >
                  ✏️ 继续编辑
                </button>
                <button
                  class="btn-pixel text-sm px-3 py-1 text-diary-fresh border-diary-fresh"
                  :disabled="!!publishingDraftId"
                  @click="handlePublishDraft(draft.id)"
                >
                  {{ publishingDraftId === draft.id ? '发布中...' : '🚀 发布' }}
                </button>
                <button
                  class="btn-pixel text-sm px-3 py-1 text-red-400 border-red-400"
                  :disabled="!!deletingDraftId"
                  @click="handleDeleteDraft(draft.id)"
                >
                  {{ deletingDraftId === draft.id ? '删除中...' : '🗑️ 删除' }}
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
      
      <div class="p-4 border-t border-gray-700 bg-gray-800/30">
        <div class="flex items-center justify-between">
          <span class="text-gray-500 font-vt323 text-sm">
            共 {{ drafts.length }} 篇草稿
          </span>
          <button
            class="btn-pixel text-gray-400 border-gray-600"
            @click="emit('close')"
          >
            关闭
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

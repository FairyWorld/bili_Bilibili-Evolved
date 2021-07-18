<template>
  <div class="manage-panel">
    <div class="manage-panel-title sub-page-row">
      <VIcon :icon="config.icon" />
      <div class="title-text">
        {{ config.title }}
      </div>
      <VIcon icon="search" :size="18" />
      <TextBox
        v-model="search"
        class="list-search"
        :placeholder="`在 ${filteredList.length} 个${config.title}中搜索`"
      />
    </div>
    <div v-if="config.description" class="sub-page-row">
      <div class="description-text">
        {{ config.description }}
      </div>
    </div>
    <div v-if="config.description" class="sub-page-row separator"></div>
    <div class="sub-page-row add-item-row">
      <div class="title-text">
        添加{{ config.title }}:
      </div>
      <div class="item-actions">
        <VButton ref="batchAddButton" @click="showBatchAddPopup()">
          <VIcon :size="18" icon="mdi-download-multiple" />
          批量
        </VButton>
        <VButton @click="browse()">
          <VIcon :size="18" icon="mdi-folder-open-outline" />
          浏览
        </VButton>
      </div>
      <VPopup
        v-model="batchAddShow"
        :trigger-element="$refs.batchAddButton"
        class="batch-add-popup"
      >
        <TextArea
          ref="batchAddTextArea"
          v-model="batchUrl"
          class="batch-add-textarea"
          :placeholder="'批量粘贴' + config.title + '链接'"
        />
        <div class="batch-add-actions">
          <VButton @click="batchAddShow = false">
            <VIcon :size="12" icon="close" />
            取消
          </VButton>
          <VButton type="primary" :disabled="!batchUrl" @click="batchAddItem()">
            <VIcon :size="18" icon="mdi-plus" />
            添加
          </VButton>
        </div>
      </VPopup>
    </div>
    <div class="sub-page-row">
      <TextBox
        v-model="url"
        class="item-url"
        :placeholder="'粘贴' + config.title + '链接'"
        @keydown.enter="addItem()"
      />
      <VButton :disabled="!url" @click="addItem()">
        <VIcon :size="18" icon="mdi-plus" />
        添加
      </VButton>
    </div>
    <div class="sub-page-row separator"></div>
    <div class="sub-page-row">
      <div class="title-text">
        已安装的{{ config.title }}:
      </div>
      <div class="exclude-built-in">
        隐藏内置{{ config.title }}
        <SwitchBox v-model="excludeBuiltIn" />
      </div>
    </div>
    <div v-if="!loaded" class="sub-page-row">
      <VLoading key="loading" />
    </div>
    <div v-if="loaded" class="manage-item-list">
      <VEmpty v-if="debouncedList.length === 0" key="empty" />
      <ManageItem
        v-for="item of debouncedList"
        :key="item.name"
      >
        <slot name="item" :item="item">
          {{ item.displayName }}
        </slot>
      </ManageItem>
    </div>
  </div>
</template>
<script lang="ts">
import { monkey } from '@/core/ajax'
import { pickFile } from '@/core/file-picker'
import { Toast, ToastType } from '@/core/toast'
import { logError } from '@/core/utils/log'
import {
  VIcon,
  VButton,
  TextBox,
  VEmpty,
  VLoading,
  VPopup,
  TextArea,
  SwitchBox,
} from '@/ui'
import ManageItem from './ManageItem.vue'

export default Vue.extend({
  components: {
    VIcon,
    VButton,
    TextBox,
    VEmpty,
    VLoading,
    VPopup,
    TextArea,
    SwitchBox,
    ManageItem,
  },
  props: {
    config: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      search: '',
      url: '',
      loaded: false,
      batchAddShow: false,
      batchUrl: '',
      excludeBuiltIn: true,
      debouncedList: [],
    }
  },
  computed: {
    filteredList() {
      return this.config.list.filter(it => this.config.listFilter(
        it, this.search, this.excludeBuiltIn,
      ))
    },
  },
  watch: {
    filteredList() {
      this.loaded = false
      window.setTimeout(() => {
        this.debouncedList = this.filteredList
        this.loaded = true
      }, 200)
    },
  },
  mounted() {
    window.setTimeout(() => {
      this.debouncedList = this.filteredList
      this.loaded = true
    })
  },
  methods: {
    async browse() {
      const codes = await pickFile({ accept: '*.js' })
      if (codes.length === 0) {
        return
      }
      const [codeFile] = codes
      const code = await codeFile.text()
      try {
        Toast.info(await this.config.onItemAdd?.(code, ''), `添加${this.config.title}`)
      } catch (error) {
        logError(error)
      }
    },
    async showBatchAddPopup() {
      this.batchAddShow = !this.batchAddShow
      if (this.batchAddShow) {
        await this.$nextTick()
        this.$refs.batchAddTextArea?.focus()
      }
    },
    async addItem() {
      if (!this.url) {
        return
      }
      const toast = Toast.info('获取中...', `添加${this.config.title}`)
      try {
        const code = await monkey({
          url: this.url,
          method: 'GET',
        })
        toast.message = await this.config.onItemAdd?.(code, this.url)
        this.url = ''
      } catch (error) {
        console.error(error)
        toast.type = ToastType.Error
        toast.message = error
      }
    },
    async batchAddItem() {
      if (!this.batchUrl) {
        return
      }
      const urls = (this.batchUrl as string).split('\n')
        .map(it => it.trim())
        .filter(it => it !== '')
      const toast = Toast.info('获取中...', '批量添加')
      const results = await Promise.allSettled(urls.map(async url => {
        const code = await monkey({
          url,
          method: 'GET',
        })
        return this.config.onItemAdd?.(code, url)
      }))
      const resultsText = results.map((r, index) => {
        const suffix = urls[index]
        if (r.status === 'fulfilled') {
          return `${r.value} ${suffix}`
        }
        console.error(r.reason, suffix)
        return `${r.reason} ${suffix}`
      }).join('\n')
      toast.message = resultsText
      this.batchUrl = ''
    },
  },
})
</script>
<style lang="scss">
@import "common";

.manage-panel {
  height: calc(var(--panel-height) - 52px - 48px);
  display: flex;
  flex-direction: column;

  > :not(:last-child) {
    margin-bottom: 12px;
  }
  .be-button .be-icon {
    margin-right: 6px;
  }
  .exclude-built-in .be-switch-box {
    margin-left: 6px;
  }
  .title-text {
    font-size: 14px;
    font-weight: bold;
  }
  .item-url-result {
    color: var(--theme-color);
  }
  .item-url {
    margin-right: 12px;
  }
  .manage-item-list {
    @include v-center();
    @include no-scrollbar();
    flex-shrink: 1;
  }
  .item-actions {
    @include h-center();
    gap: 12px;
  }
  .exclude-built-in {
    @include h-center();
  }
  .be-loading {
    width: 100%;
    text-align: center;
  }
  .description-text {
    opacity: .75;
  }
  .add-item-row {
    position: relative;
  }
  .batch-add-popup {
    top: calc(100% + 8px);
    left: 50%;
    transition: .2s ease-out;
    transform: translateX(-50%) translateY(-8px);
    padding: 8px;
    width: 100%;
    min-height: calc(var(--panel-height) / 2);
    @include popup();
    @include v-stretch(8px);
    &.open {
      transform: translateX(-50%) translateY(0px);
    }
    .be-text-area {
      flex: 1 0 auto;
    }
    .batch-add-actions {
      @include h-center(8px);
      .be-button {
        flex: 1 0 0;
      }
      .be-icon {
        margin-right: 6px;
      }
    }
  }
  &-title {
    .be-icon {
      margin-right: 6px;
    }
    .title-text {
      font-size: 16px;
      font-weight: bold;
      flex: 1 0 auto;
    }
  }
}
</style>
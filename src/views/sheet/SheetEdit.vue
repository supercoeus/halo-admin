<template>
  <div>
    <a-row :gutter="12">
      <a-col :span="24">
        <div style="margin-bottom: 16px">
          <a-input
            v-model="sheetToStage.title"
            size="large"
            placeholder="请输入页面标题"
          />
        </div>
        <div id="editor">
          <halo-editor
            ref="md"
            v-model="sheetToStage.originalContent"
            :boxShadow="false"
            :toolbars="toolbars"
            :ishljs="true"
            :autofocus="false"
            @imgAdd="handleAttachmentUpload"
            @save="handleSaveDraft"
          />
        </div>
      </a-col>
    </a-row>

    <SheetSettingDrawer
      :sheet="sheetToStage"
      :sheetMetas="selectedSheetMetas"
      :visible="sheetSettingVisible"
      @close="onSheetSettingsClose"
      @onRefreshSheet="onRefreshSheetFromSetting"
      @onRefreshSheetMetas="onRefreshSheetMetasFromSetting"
      @onSaved="onSaved"
    />

    <AttachmentDrawer v-model="attachmentDrawerVisible" />
    <footer-tool-bar :style="{ width: isSideMenu() && isDesktop() ? `calc(100% - ${sidebarOpened ? 256 : 80}px)` : '100%'}">
      <a-button
        type="danger"
        @click="handleSaveDraft"
        :disabled="saving"
      >保存草稿</a-button>
      <a-button
        @click="handlePreview"
        style="margin-left: 8px;"
        :disabled="saving"
      >预览</a-button>
      <a-button
        type="primary"
        style="margin-left: 8px;"
        @click="handleShowSheetSetting"
      >发布</a-button>
      <a-button
        type="dashed"
        style="margin-left: 8px;"
        @click="()=>this.attachmentDrawerVisible = true"
      >附件库</a-button>
    </footer-tool-bar>
  </div>
</template>

<script>
import { mixin, mixinDevice } from '@/utils/mixin.js'
import { mapGetters } from 'vuex'
import moment from 'moment'
import { toolbars } from '@/core/const'
import SheetSettingDrawer from './components/SheetSettingDrawer'
import AttachmentDrawer from '../attachment/components/AttachmentDrawer'
import FooterToolBar from '@/components/FooterToolbar'
import { haloEditor } from 'halo-editor'
import 'halo-editor/dist/css/index.css'
import sheetApi from '@/api/sheet'
import attachmentApi from '@/api/attachment'
export default {
  components: {
    haloEditor,
    FooterToolBar,
    AttachmentDrawer,
    SheetSettingDrawer
  },
  mixins: [mixin, mixinDevice],
  data() {
    return {
      toolbars,
      attachmentDrawerVisible: false,
      sheetSettingVisible: false,
      sheetToStage: {},
      selectedSheetMetas: [],
      isSaved: false,
      contentChanges: 0,
      saving: false
    }
  },
  beforeRouteEnter(to, from, next) {
    // Get sheetId id from query
    const sheetId = to.query.sheetId

    next(vm => {
      if (sheetId) {
        sheetApi.get(sheetId).then(response => {
          const sheet = response.data.data
          vm.sheetToStage = sheet
          vm.selectedSheetMetas = sheet.sheetMetas
        })
      }
    })
  },
  destroyed: function() {
    if (this.sheetSettingVisible) {
      this.sheetSettingVisible = false
    }
    if (this.attachmentDrawerVisible) {
      this.attachmentDrawerVisible = false
    }
    if (window.onbeforeunload) {
      window.onbeforeunload = null
    }
  },
  beforeRouteLeave(to, from, next) {
    if (this.sheetSettingVisible) {
      this.sheetSettingVisible = false
    }
    if (this.attachmentDrawerVisible) {
      this.attachmentDrawerVisible = false
    }
    if (this.contentChanges <= 1) {
      next()
    } else if (this.isSaved) {
      next()
    } else {
      this.$confirm({
        title: '当前页面数据未保存，确定要离开吗？',
        content: h => <div style="color:red;">如果离开当面页面，你的数据很可能会丢失！</div>,
        onOk() {
          next()
        },
        onCancel() {
          next(false)
        }
      })
    }
  },
  mounted() {
    window.onbeforeunload = function(e) {
      e = e || window.event
      if (e) {
        e.returnValue = '当前页面数据未保存，确定要离开吗？'
      }
      return '当前页面数据未保存，确定要离开吗？'
    }
  },
  watch: {
    temporaryContent: function(newValue, oldValue) {
      if (newValue) {
        this.contentChanges++
      }
    }
  },
  computed: {
    temporaryContent() {
      return this.sheetToStage.originalContent
    },
    ...mapGetters(['options'])
  },
  methods: {
    handleSaveDraft() {
      this.sheetToStage.status = 'DRAFT'
      this.saving = true
      if (!this.sheetToStage.title) {
        this.sheetToStage.title = moment(new Date()).format('YYYY-MM-DD-HH-mm-ss')
      }
      if (this.sheetToStage.id) {
        sheetApi.update(this.sheetToStage.id, this.sheetToStage, false).then(response => {
          this.$log.debug('Updated sheet', response.data.data)
          this.$message.success('保存草稿成功！')
          this.saving = false
        })
      } else {
        sheetApi.create(this.sheetToStage, false).then(response => {
          this.$log.debug('Created sheet', response.data.data)
          this.$message.success('保存草稿成功！')
          this.sheetToStage = response.data.data
          this.saving = false
        })
      }
    },
    handleAttachmentUpload(pos, $file) {
      var formdata = new FormData()
      formdata.append('file', $file)
      attachmentApi.upload(formdata).then(response => {
        var responseObject = response.data

        if (responseObject.status === 200) {
          var HaloEditor = this.$refs.md
          HaloEditor.$img2Url(pos, encodeURI(responseObject.data.path))
          this.$message.success('图片上传成功！')
        } else {
          this.$message.error('图片上传失败：' + responseObject.message)
        }
      })
    },
    handleShowSheetSetting() {
      this.sheetSettingVisible = true
    },
    handlePreview() {
      this.sheetToStage.status = 'DRAFT'
      if (!this.sheetToStage.title) {
        this.sheetToStage.title = moment(new Date()).format('YYYY-MM-DD-HH-mm-ss')
      }
      this.saving = true
      if (this.sheetToStage.id) {
        sheetApi.update(this.sheetToStage.id, this.sheetToStage, false).then(response => {
          this.$log.debug('Updated sheet', response.data.data)
          sheetApi.preview(this.sheetToStage.id).then(response => {
            window.open(response.data, '_blank')
            this.saving = false
          })
        })
      } else {
        sheetApi.create(this.sheetToStage, false).then(response => {
          this.$log.debug('Created sheet', response.data.data)
          this.sheetToStage = response.data.data
          sheetApi.preview(this.sheetToStage.id).then(response => {
            window.open(response.data, '_blank')
            this.saving = false
          })
        })
      }
    },
    onSheetSettingsClose() {
      this.sheetSettingVisible = false
    },
    onRefreshSheetFromSetting(sheet) {
      this.sheetToStage = sheet
    },
    onRefreshSheetMetasFromSetting(sheetMetas) {
      this.selectedSheetMetas = sheetMetas
    },
    onSaved(isSaved) {
      this.isSaved = isSaved
    }
  }
}
</script>

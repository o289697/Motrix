<template>
  <el-container class="content panel" direction="vertical">
    <el-header class="panel-header" height="84">
      <h4>{{ title }}</h4>
    </el-header>
    <el-main class="panel-content">
      <el-form
        class="form-preference"
        ref="basicForm"
        label-position="right"
        size="mini"
        :model="form"
        :rules="rules">
        <el-form-item :label="`${$t('preferences.startup')}: `" :label-width="formLabelWidth">
          <el-col
            class="form-item-sub"
            :span="24"
            v-if="!isLinux()"
          >
            <el-checkbox v-model="form.openAtLogin">
              {{ $t('preferences.open-at-login') }}
            </el-checkbox>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            <el-checkbox v-model="form.keepWindowState">
              {{ $t('preferences.keep-window-state') }}
            </el-checkbox>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            <el-checkbox v-model="form.resumeAllWhenAppLaunched">
              {{ $t('preferences.auto-resume-all') }}
            </el-checkbox>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            <el-checkbox v-model="form.autoCheckUpdate">
              {{ $t('preferences.auto-check-update') }}
            </el-checkbox>
            <div class="el-form-item__info" style="margin-top: 8px;" v-if="form.lastCheckUpdateTime !== 0">
              {{ $t('preferences.last-check-update-time') + ': ' + (form.lastCheckUpdateTime !== 0 ?  new
              Date(form.lastCheckUpdateTime).toLocaleString() : new Date().toLocaleString()) }}
            </div>
          </el-col>
        </el-form-item>
        <el-form-item :label="`${$t('preferences.default-dir')}: `" :label-width="formLabelWidth">
          <el-input placeholder="" v-model="downloadDir" :readonly="isMas()">
            <mo-select-directory
              v-if="isRenderer()"
              slot="append"
              @selected="onDirectorySelected"
            />
          </el-input>
          <div class="el-form-item__info" v-if="isMas()" style="margin-top: 8px;">
            {{ $t('preferences.mas-default-dir-tips') }}
          </div>
        </el-form-item>
        <el-form-item :label="`${$t('preferences.transfer-setting')}: `" :label-width="formLabelWidth">
          <el-col class="form-item-sub" :span="24">
            {{ $t('preferences.transfer-speed-upload') }}
            <el-radio-group class="radio-group-margin-left" v-model="maxOverallUploadLimitRadio">
              <el-radio :label="true">{{ $t('preferences.transfer-speed-no-set') }}</el-radio>
              <el-radio :label="false">
                {{ $t('preferences.transfer-speed-set') }}
                <el-input class="download-speed-input" type="number" v-model="maxOverallUploadLimitValue">
                  <el-select v-model="maxOverallUploadLimitUnit" slot="append">
                    <el-option label="K" value="K"></el-option>
                    <el-option label="M" value="M"></el-option>
                  </el-select>
                </el-input>
              </el-radio>
            </el-radio-group>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            {{ $t('preferences.transfer-speed-download') }}
            <el-radio-group class="radio-group-margin-left" v-model="maxOverallDownloadLimitRadio">
              <el-radio :label="true">{{ $t('preferences.transfer-speed-no-set') }}</el-radio>
              <el-radio :label="false">
                {{ $t('preferences.transfer-speed-set') }}
                <el-input class="download-speed-input" type="number" v-model="maxOverallDownloadLimitValue">
                  <el-select v-model="maxOverallDownloadLimitUnit" slot="append">
                    <el-option label="K" value="K"></el-option>
                    <el-option label="M" value="M"></el-option>
                  </el-select>
                </el-input>
              </el-radio>
            </el-radio-group>
          </el-col>
        </el-form-item>
        <el-form-item :label="`${$t('preferences.task-manage')}: `" :label-width="formLabelWidth">
          <el-col class="form-item-sub" :span="24">
            {{ $t('preferences.max-concurrent-downloads') }}
            <el-input-number
              v-model="form.maxConcurrentDownloads"
              controls-position="right"
              :min="1"
              :max="10"
              :label="$t('preferences.max-concurrent-downloads')">
            </el-input-number>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            {{ $t('preferences.max-connection-per-server') }}
            <el-input-number
              v-model="form.split"
              controls-position="right"
              :min="1"
              :max="form.maxConnectionPerServer"
              :label="$t('preferences.max-connection-per-server')">
            </el-input-number>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            <el-checkbox v-model="form.continue">
              {{ $t('preferences.continue') }}
            </el-checkbox>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            <el-checkbox v-model="form.newTaskShowDownloading">
              {{ $t('preferences.new-task-show-downloading') }}
            </el-checkbox>
          </el-col>
          <el-col class="form-item-sub" :span="24">
            <el-checkbox v-model="form.taskNotification">
              {{ $t('preferences.task-completed-notify') }}
            </el-checkbox>
          </el-col>
        </el-form-item>
      </el-form>
      <div class="form-actions">
        <el-button type="primary" @click="submitForm('basicForm')">{{ $t('preferences.save') }}</el-button>
        <el-button @click="resetForm('basicForm')">{{ $t('preferences.discard') }}</el-button>
      </div>
    </el-main>
  </el-container>
</template>
<style>
  .radio-group-margin-left {
    margin-left: 6px;
  }
  .el-input-group.download-speed-input {
    width: 50%;
  }
  .el-input-group .el-select .el-input {
    width: 60px;
  }
 .download-speed-input input::-webkit-outer-spin-button, .download-speed-input input::-webkit-inner-spin-button {
    -webkit-appearance: none; 
  }
</style>

<script>
  import is from 'electron-is'
  import { mapState } from 'vuex'
  import SelectDirectory from '@/components/Native/SelectDirectory'
  import { prettifyDir } from '@/components/Native/utils'

  const initialForm = (config) => {
    const {
      autoCheckUpdate,
      dir,
      keepWindowState,
      lastCheckUpdateTime,
      maxConcurrentDownloads,
      maxConnectionPerServer,
      maxOverallUploadLimit,
      maxOverallDownloadLimit,
      newTaskShowDownloading,
      openAtLogin,
      resumeAllWhenAppLaunched,
      split,
      taskNotification
    } = config
    const result = {
      autoCheckUpdate,
      continue: config.continue,
      dir,
      keepWindowState,
      lastCheckUpdateTime,
      maxConcurrentDownloads,
      maxConnectionPerServer,
      maxOverallUploadLimit,
      maxOverallDownloadLimit,
      newTaskShowDownloading,
      openAtLogin,
      resumeAllWhenAppLaunched,
      split,
      taskNotification
    }
    return result
  }

  export default {
    name: 'mo-preference-basic',
    components: {
      [SelectDirectory.name]: SelectDirectory
    },
    data: function () {
      return {
        formLabelWidth: '23%',
        form: initialForm(this.$store.state.preference.config),
        rules: {}
      }
    },
    computed: {
      title: function () {
        return this.$t('preferences.basic')
      },
      downloadDir: function () {
        return prettifyDir(this.form.dir)
      },
      maxOverallUploadLimitValue: {
        get: function () {
          let value = this.form.maxOverallUploadLimit
          if (typeof value === 'string') {
            return value.substring(0, value.length - 1)
          }
          return this.form.maxOverallUploadLimit
        },
        set: function (value) {
          value = this.fixSpeedInputValue(value)
          this.form.maxOverallUploadLimit = value + this.maxOverallUploadLimitUnit
        }
      },
      maxOverallUploadLimitUnit: {
        get: function () {
          let value = this.form.maxOverallUploadLimit
          if (typeof value === 'string') {
            return value.substring(value.length - 1, value.length)
          }
          return 'K'
        },
        set: function (value) {
          this.form.maxOverallUploadLimit = this.maxOverallUploadLimitValue + value
        }
      },
      maxOverallUploadLimitRadio: {
        get: function () {
          let value = this.form.maxOverallUploadLimit
          if (typeof value === 'string') {
            return value.startsWith('0')
          }
          return value === 0
        },
        set: function (value) {
          if (value) {
            this.form.maxOverallUploadLimit = 0
          } else {
            this.form.maxOverallUploadLimit = 128
          }
        }
      },
      maxOverallDownloadLimitValue: {
        get: function () {
          let value = this.form.maxOverallDownloadLimit
          if (typeof value === 'string') {
            return value.substring(0, value.length - 1)
          }
          return this.form.maxOverallDownloadLimit
        },
        set: function (value) {
          value = this.fixSpeedInputValue(value)
          this.form.maxOverallDownloadLimit = value + this.maxOverallDownloadLimitUnit
        }
      },
      maxOverallDownloadLimitUnit: {
        get: function () {
          let value = this.form.maxOverallDownloadLimit
          if (typeof value === 'string') {
            return value.substring(value.length - 1, value.length)
          }
          return 'K'
        },
        set: function (value) {
          this.form.maxOverallDownloadLimit = this.maxOverallDownloadLimitValue + value
        }
      },
      maxOverallDownloadLimitRadio: {
        get: function () {
          let value = this.form.maxOverallDownloadLimit
          if (typeof value === 'string') {
            return value.startsWith('0')
          }
          return value === 0
        },
        set: function (value) {
          if (value) {
            this.form.maxOverallDownloadLimit = 0
          } else {
            this.form.maxOverallDownloadLimit = 128
          }
        }
      },
      ...mapState('preference', {
        config: state => state.config
      })
    },
    methods: {
      isRenderer: is.renderer,
      isMas: is.mas,
      isLinux: is.linux,
      onDirectorySelected (dir) {
        this.form.dir = dir
      },
      submitForm (formName) {
        this.$refs[formName].validate((valid) => {
          if (!valid) {
            console.log('error submit!!')
            return false
          }

          const data = {
            ...this.form
          }
          const { openAtLogin } = data

          this.$store.dispatch('preference/save', data)
          if (this.isRenderer()) {
            this.$electron.ipcRenderer.send('command', 'application:open-at-login', openAtLogin)
            this.$electron.ipcRenderer.send('command', 'application:relaunch')
          }
        })
      },
      resetForm (formName) {
        this.form = initialForm(this.$store.state.preference.config)
      },
      fixSpeedInputValue (value) {
        if (!value) {
          return 0
        }
        if (typeof value === 'string' && value.startsWith('0')) {
          return this.fixSpeedInputValue(value.substring(1, value.length))
        }
        return value
      }
    }
  }
</script>

<style lang="scss">
</style>

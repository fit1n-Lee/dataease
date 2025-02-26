<script lang="ts" setup>
import { ref, reactive, computed, toRefs, nextTick, watch } from 'vue'
import { useI18n } from '@/hooks/web/useI18n'
import type { FormInstance, FormRules } from 'element-plus-secondary'
import EmptyBackground from '@/components/empty-background/src/EmptyBackground.vue'
import { cloneDeep } from 'lodash-es'
import ApiHttpRequestDraw from './ApiHttpRequestDraw.vue'
import { Configuration, ApiConfiguration, SyncSetting } from './index.vue'
import { getSchema } from '@/api/datasource'
import { Calendar } from '@element-plus/icons-vue'
import { Base64 } from 'js-base64'
import { CustomPassword } from '@/components/custom-password'
import { ElForm, ElMessage } from 'element-plus-secondary'
import Cron from '@/components/cron/src/Cron.vue'
import { ComponentPublicInstance } from 'vue'
const { t } = useI18n()
const prop = defineProps({
  form: {
    required: false,
    default() {
      return reactive<{
        id: number
        name: string
        desc: string
        type: string
        syncSetting?: SyncSetting
        configuration?: Configuration
        apiConfiguration?: ApiConfiguration[]
      }>({
        id: 0,
        name: '',
        desc: '',
        type: 'API',
        apiConfiguration: []
      })
    },
    type: Object
  },

  activeStep: {
    required: false,
    default: 1,
    type: Number
  }
})

const { form, activeStep } = toRefs(prop)

const state = reactive({
  itemRef: []
})

const schemas = ref([])
const dsForm = ref<FormInstance>()

const cronEdit = ref(true)

const defaultRule = {
  name: [
    {
      required: true,
      message: t('datasource.input_name'),
      trigger: 'blur'
    },
    {
      min: 2,
      max: 64,
      message: t('datasource.input_limit_2_25', [2, 64]),
      trigger: 'blur'
    }
  ]
}

const rule = ref<FormRules>(cloneDeep(defaultRule))
const api_table_title = ref('')
const editApiItem = ref()
const defaultApiItem = {
  name: '',
  deTableName: '',
  url: '',
  serialNumber: 0,
  method: 'GET',
  request: {
    headers: [{}],
    arguments: [],
    body: {
      type: '',
      raw: '',
      kvs: []
    },
    authManager: {
      verification: '',
      username: '',
      password: ''
    }
  },
  fields: []
}

const initForm = type => {
  if (type !== 'API') {
    form.value.configuration = {
      dataBase: '',
      extraParams: '',
      username: '',
      password: '',
      host: '',
      authMethod: '',
      port: '',
      initialPoolSize: 5,
      minPoolSize: 5,
      maxPoolSize: 5,
      queryTimeout: 30
    }
    schemas.value = []
    rule.value = cloneDeep(defaultRule)
    setRules()
  }
  if (type === 'API') {
    form.value.syncSetting = {
      updateType: 'all_scope',
      syncRate: 'SIMPLE_CRON',
      simpleCronValue: '1',
      simpleCronType: 'minute',
      startTime: '',
      endTime: '',
      endLimit: '0',
      cron: '0 0/1 * * * ? *'
    }
  }
  if (type === 'oracle') {
    form.value.configuration.connectionType = 'sid'
  }

  form.value.type = type
  setTimeout(() => {
    dsForm.value.clearValidate()
  }, 0)
}

const notapiexcelconfig = computed(() => form.value.type !== 'API')

const authMethodList = [
  {
    id: 'passwd',
    label: t('datasource.passwd')
  },
  {
    id: 'kerberos',
    label: 'Kerberos'
  }
]
const setRules = () => {
  const configRules = {
    'configuration.dataBase': [
      {
        required: true,
        message: t('datasource.please_input_data_base'),
        trigger: 'blur'
      }
    ],
    'configuration.authMethod': [
      {
        required: true,
        message: t('datasource.please_select_oracle_type'),
        trigger: 'blur'
      }
    ],
    'configuration.username': [
      {
        required: true,
        message: t('datasource.please_input_user_name'),
        trigger: 'blur'
      }
    ],
    'configuration.password': [
      {
        required: true,
        message: t('datasource.please_input_password'),
        trigger: 'blur'
      }
    ],
    'configuration.host': [
      {
        required: true,
        message: t('datasource._ip_address'),
        trigger: 'blur'
      }
    ],
    'configuration.extraParams': [
      {
        required: false,
        message: t('datasource.please_input_url'),
        trigger: 'blur'
      }
    ],
    'configuration.port': [
      {
        required: true,
        message: t('datasource.please_input_port'),
        trigger: 'blur'
      }
    ],
    'configuration.initialPoolSize': [
      {
        required: true,
        message: t('common.inputText') + t('datasource.initial_pool_size'),
        trigger: 'blur'
      }
    ],
    'configuration.minPoolSize': [
      {
        required: true,
        message: t('common.inputText') + t('datasource.min_pool_size'),
        trigger: 'blur'
      }
    ],
    'configuration.maxPoolSize': [
      {
        required: true,
        message: t('common.inputText') + t('datasource.max_pool_size'),
        trigger: 'blur'
      }
    ],
    'configuration.queryTimeout': [
      {
        required: true,
        message: t('common.inputText') + t('datasource.query_timeout'),
        trigger: 'blur'
      }
    ]
  }
  if (['oracle', 'sqlServer', 'pg', 'redshift', 'db2'].includes(form.value.type)) {
    configRules['configuration.schema'] = [
      {
        required: true,
        message: t('datasource.please_choose_schema'),
        trigger: 'blur'
      }
    ]
  }

  if (form.value.type === 'oracle') {
    configRules['configuration.connectionType'] = [
      {
        required: true,
        message: t('datasource.connection_mode'),
        trigger: 'change'
      }
    ]
  }
  rule.value = { ...cloneDeep(configRules), ...cloneDeep(defaultRule) }
}

watch(
  () => form.value.type,
  val => {
    if (val !== 'API') {
      rule.value = cloneDeep(defaultRule)
      setRules()
    }
  },
  {
    immediate: true
  }
)

watch(
  () => activeStep.value,
  () => {
    showCron.value = form.value.syncSetting?.syncRate === 'CRON'
  }
)

const setItemRef = (ele: ComponentPublicInstance | null | Element) => {
  state.itemRef.push(ele)
}

const copyItem = (item?: ApiConfiguration) => {
  var newItem = JSON.parse(JSON.stringify(item))
  newItem.deTableName = ''
  newItem.serialNumber =
    form.value.apiConfiguration[form.value.apiConfiguration.length - 1].serialNumber + 1
  var reg = new RegExp(item.name + '_copy_' + '([0-9]*)', 'gim')
  var number = 0
  for (var i = 1; i < form.value.apiConfiguration.length; i++) {
    var match = form.value.apiConfiguration[i].name.match(reg)
    if (match !== null) {
      var num = match[0].substring(
        form.value.apiConfiguration[i].name.length + 5,
        match[0].length - 1
      )
      if (!isNaN(parseInt(num)) && parseInt(num) > number) {
        number = parseInt(num)
      }
    }
  }
  number = number + 1
  newItem.name = item.name + '_copy_' + number
  state.itemRef = []
  form.value.apiConfiguration.push(newItem)
  ElMessage.success(t('datasource.success_copy'))
}
const addApiItem = item => {
  let apiItem = null
  api_table_title.value = t('datasource.data_table')
  if (item) {
    apiItem = cloneDeep(item)
  } else {
    apiItem = cloneDeep(defaultApiItem)
    apiItem.serialNumber =
      form.value.apiConfiguration.length > 0
        ? form.value.apiConfiguration[form.value.apiConfiguration.length - 1].serialNumber + 1
        : 0
  }
  nextTick(() => {
    editApiItem.value.initApiItem(apiItem, form.value.apiConfiguration)
  })
}
const showPriority = ref(false)

const deleteItem = (item, idx) => {
  form.value.apiConfiguration.splice(form.value.apiConfiguration.indexOf(item), 1)
  cancelItem(idx)
}
const cancelItem = (index: number) => {
  state.itemRef[index].hide()
}
const submitForm = () => {
  dsForm.value.clearValidate()
  return dsForm.value.validate
}

const clearForm = () => {
  return dsForm.value.clearValidate()
}

const resetForm = () => {
  dsForm.value.resetFields()
}

const returnItem = apiItem => {
  var find = false
  for (let i = 0; i < form.value.apiConfiguration.length; i++) {
    if (form.value.apiConfiguration[i].serialNumber === apiItem.serialNumber) {
      find = true
      form.value.apiConfiguration[i] = apiItem
    }
  }
  if (!find) {
    state.itemRef = []
    form.value.apiConfiguration.push(apiItem)
  }
}

const showCron = ref(false)

const onRateChange = () => {
  if (form.value.syncSetting.syncRate === 'SIMPLE') {
    form.value.syncSetting.endLimit = 0
    form.value.syncSetting.endTime = 0
    form.value.syncSetting.cron = ''
  }
  if (form.value.syncSetting.syncRate === 'SIMPLE_CRON') {
    form.value.syncSetting.cron = '0 0/1 * * * ? *'
    form.value.syncSetting.simpleCronType = 'minute'
  }
  if (form.value.syncSetting.syncRate === 'CRON') {
    form.value.syncSetting.cron = '00 00 * ? * * *'
  }
  nextTick(() => {
    showCron.value = form.value.syncSetting.syncRate === 'CRON'
  })
}

const onSimpleCronChange = () => {
  if (form.value.syncSetting.simpleCronType === 'minute') {
    if (form.value.syncSetting.simpleCronValue < 1 || form.value.syncSetting.simpleCronValue > 59) {
      ElMessage.warning(t('cron.minute_limit'))
      form.value.syncSetting.simpleCronValue = 59
    }
    form.value.syncSetting.cron = '0 0/' + form.value.syncSetting.simpleCronValue + ' * * * ? *'
    return
  }
  if (form.value.syncSetting.simpleCronType === 'hour') {
    if (form.value.syncSetting.simpleCronValue < 1 || form.value.syncSetting.simpleCronValue > 23) {
      ElMessage.warning(t('cron.hour_limit'))
      form.value.syncSetting.simpleCronValue = 23
    }
    form.value.syncSetting.cron = '0 0 0/' + form.value.syncSetting.simpleCronValue + ' * * ? *'
    return
  }
  if (form.value.syncSetting.simpleCronType === 'day') {
    if (form.value.syncSetting.simpleCronValue < 1 || form.value.syncSetting.simpleCronValue > 31) {
      ElMessage.warning(t('cron.day_limit'))
      form.value.syncSetting.simpleCronValue = 31
    }
    form.value.syncSetting.cron = '0 0 0 1/' + form.value.syncSetting.simpleCronValue + ' * ? *'
    return
  }
}

const showSchema = ref(false)

const getDsSchema = () => {
  showSchema.value = true
  const validateFrom = dsForm.value.validate
  validateFrom(val => {
    showSchema.value = false
    if (val) {
      const request = JSON.parse(JSON.stringify(form.value))
      request.configuration = Base64.encode(JSON.stringify(request.configuration))
      getSchema(request).then(res => {
        schemas.value = res.data
        ElMessage.success(t('commons.success'))
      })
    }
  })
}

const validatorSchema = () => {
  dsForm.value.validateField('configuration.schema')
}

const dsApiForm = ref()
const apiRule = {
  'syncSetting.updateType': [
    {
      required: true,
      message: t('datasource.update_type'),
      trigger: 'change'
    }
  ],
  'syncSetting.syncRate': [
    {
      required: true,
      message: t('datasource.sync_rate'),
      trigger: 'change'
    }
  ],
  'syncSetting.simpleCronValue': [
    {
      required: true,
      message: t('auth.set_rules'),
      trigger: 'change'
    }
  ],
  'syncSetting.cron': [
    {
      required: true,
      message: t('common.cron_exp'),
      trigger: 'change'
    }
  ],
  'syncSetting.startTime': [
    {
      required: true,
      message: t('datasource.start_time'),
      trigger: 'change'
    }
  ],
  'syncSetting.endLimit': [
    {
      required: true,
      message: t('datasource.end_time'),
      trigger: 'change'
    }
  ]
}

defineExpose({
  submitForm,
  resetForm,
  initForm,
  clearForm
})
</script>

<template>
  <div class="editor-detail">
    <div class="detail-inner create-dialog">
      <div v-show="form.type === 'API'" class="info-update">
        <div :class="activeStep === 1 && 'active'" class="info-text">数据源配置信息</div>
        <div class="update-info-line"></div>
        <div :class="activeStep === 2 && 'active'" class="update-text">数据更新设置</div>
      </div>
      <div class="title-form_primary base-info" v-show="activeStep !== 2 && form.type === 'API'">
        {{ t('datasource.basic_info') }}
      </div>
      <el-form
        ref="dsForm"
        :model="form"
        :rules="rule"
        label-width="180px"
        label-position="top"
        require-asterisk-position="right"
      >
        <el-form-item
          :label="t('auth.datasource') + t('chart.name')"
          prop="name"
          v-show="activeStep !== 2"
        >
          <el-input
            v-model="form.name"
            autocomplete="off"
            :placeholder="t('datasource.input_name')"
          />
        </el-form-item>
        <el-form-item :label="t('common.description')" v-show="activeStep !== 2">
          <el-input
            class="description-text"
            type="textarea"
            :placeholder="t('common.inputText')"
            v-model="form.description"
            :row="10"
            :maxlength="50"
            show-word-limit
          />
        </el-form-item>
        <template v-if="form.type === 'API'">
          <div class="title-form_primary flex-space table-info-mr" v-show="activeStep !== 2">
            <span>{{ t('datasource.data_table') }}</span>
            <el-button type="primary" style="margin-left: auto" @click="() => addApiItem(null)">
              <template #icon>
                <Icon name="icon_add_outlined"></Icon>
              </template>
              {{ t('common.add') }}
            </el-button>
          </div>
          <empty-background
            v-show="activeStep !== 2"
            v-if="!form.apiConfiguration.length"
            :description="t('datasource.no_data_table')"
            img-type="noneWhite"
          />
          <template v-if="form.type === 'API' && activeStep === 1">
            <div class="api-card-content">
              <div
                v-for="(api, idx) in form.apiConfiguration"
                :key="api.id"
                class="api-card"
                @click="addApiItem(api)"
              >
                <el-row>
                  <el-col style="display: flex" :span="19">
                    <span class="name ellipsis">{{ api.name }}</span>
                    <span v-if="api.status === 'Error'" class="de-tag invalid">{{
                      t('datasource.invalid')
                    }}</span>
                    <span v-if="api.status === 'Success'" class="de-tag valid">{{
                      t('datasource.valid')
                    }}</span>
                  </el-col>
                  <el-col style="text-align: right" :span="5">
                    <el-icon class="de-copy-icon hover-icon" @click.stop="copyItem(api)">
                      <Icon name="de-copy"></Icon>
                    </el-icon>

                    <span @click.stop>
                      <el-popover
                        placement="top"
                        width="200"
                        :ref="setItemRef"
                        show-arrow
                        popper-class="api-table-delete"
                        trigger="click"
                      >
                        <template #reference>
                          <el-icon class="de-delete-icon hover-icon">
                            <Icon name="de-delete"></Icon>
                          </el-icon>
                        </template>
                        <template #default>
                          <el-icon class="de-copy-icon icon-warning">
                            <Icon name="icon_warning_filled"></Icon>
                          </el-icon>
                          <div class="tips">
                            {{ t('datasource.delete_this_item') }}
                          </div>
                          <div class="foot">
                            <el-button style="min-width: 48px" secondary @click="cancelItem(idx)">{{
                              t('common.cancel')
                            }}</el-button>
                            <el-button
                              style="min-width: 48px"
                              type="primary"
                              @click="deleteItem(api, idx)"
                              >{{ t('common.sure') }}</el-button
                            >
                          </div>
                        </template>
                      </el-popover>
                    </span>
                  </el-col>
                </el-row>
                <div class="req-title">
                  <span>{{ t('datasource.method') }}</span>
                  <span>{{ t('datasource.url') }}</span>
                </div>
                <div class="req-value">
                  <span>{{ api.method }}</span>
                  <el-tooltip effect="dark" :content="api.url" placement="top">
                    <span>{{ api.url }}</span>
                  </el-tooltip>
                </div>
              </div>
            </div>
          </template>
        </template>
        <template v-if="notapiexcelconfig">
          <el-form-item :label="t('datasource.host')" prop="configuration.host">
            <el-input
              v-model="form.configuration.host"
              :placeholder="t('datasource._ip_address')"
              autocomplete="off"
            />
          </el-form-item>
          <el-form-item :label="t('datasource.port')" prop="configuration.port">
            <el-input-number
              v-model="form.configuration.port"
              autocomplete="off"
              step-strictly
              class="text-left"
              :min="0"
              :placeholder="t('common.inputText') + t('datasource.port')"
              controls-position="right"
              type="number"
            />
          </el-form-item>
          <el-form-item :label="t('datasource.data_base')" prop="configuration.dataBase">
            <el-input
              v-model="form.configuration.dataBase"
              :placeholder="t('datasource.please_input_data_base')"
              autocomplete="off"
            />
          </el-form-item>
          <el-form-item
            :label="t('datasource.auth_method')"
            prop="configuration.authMethod"
            v-if="form.type === 'presto'"
          >
            <el-select
              :placeholder="t('common.inputText') + t('datasource.auth_method')"
              v-model="form.configuration.authMethod"
              class="de-select"
            >
              <el-option
                v-for="item in authMethodList"
                :key="item.id"
                :label="item.label"
                :value="item.id"
              />
            </el-select>
          </el-form-item>
          <el-form-item
            :label="t('datasource.client_principal')"
            prop="configuration.username"
            v-if="form.type === 'presto'"
          >
            <el-input
              :placeholder="t('common.inputText') + t('datasource.client_principal')"
              v-model="form.configuration.username"
              autocomplete="off"
            />
          </el-form-item>
          <el-form-item
            :label="t('datasource.keytab_Key_path')"
            prop="configuration.password"
            v-if="form.type === 'presto'"
          >
            <CustomPassword
              :placeholder="t('common.inputText') + t('datasource.keytab_Key_path')"
              show-password
              type="password"
              v-model="form.configuration.password"
            />
            <p>
              {{ t('datasource.kerbers_info') }}
            </p>
          </el-form-item>
          <el-form-item :label="t('datasource.user_name')" v-if="form.type !== 'presto'">
            <el-input
              :placeholder="t('common.inputText') + t('datasource.user_name')"
              v-model="form.configuration.username"
              autocomplete="off"
            />
          </el-form-item>
          <el-form-item :label="t('datasource.password')" v-if="form.type !== 'presto'">
            <CustomPassword
              :placeholder="t('common.inputText') + t('datasource.password')"
              show-password
              type="password"
              v-model="form.configuration.password"
            />
          </el-form-item>
          <el-form-item :label="t('datasource.extra_params')">
            <el-input
              :placeholder="t('common.inputText') + t('datasource.extra_params')"
              v-model="form.configuration.extraParams"
              autocomplete="off"
            />
          </el-form-item>
          <el-form-item
            v-if="form.type == 'oracle'"
            :label="t('datasource.connection_mode')"
            prop="configuration.connectionType"
          >
            <el-radio v-model="form.configuration.connectionType" label="sid"
              >{{ t('datasource.oracle_sid') }}
            </el-radio>
            <el-radio v-model="form.configuration.connectionType" label="serviceName">
              {{ t('datasource.oracle_service_name') }}
            </el-radio>
          </el-form-item>
          <el-form-item
            v-if="['oracle', 'sqlServer', 'pg', 'redshift', 'db2'].includes(form.type)"
            class="schema-label"
            :prop="showSchema ? '' : 'configuration.schema'"
          >
            <template v-slot:label>
              <span class="name">{{ t('datasource.schema') }}<i class="required" /></span>
              <el-button text size="small" @click="getDsSchema()">
                <template #icon>
                  <Icon name="icon_add_outlined"></Icon>
                </template>
                {{ t('datasource.get_schema') }}
              </el-button>
            </template>
            <el-select
              v-model="form.configuration.schema"
              filterable
              :placeholder="t('common.please_select')"
              class="de-select"
              @change="validatorSchema"
              @blur="validatorSchema"
            >
              <el-option v-for="item in schemas" :key="item" :label="item" :value="item" />
            </el-select>
          </el-form-item>
          <span
            v-if="!['es', 'api'].includes(form.type)"
            class="de-expand"
            @click="showPriority = !showPriority"
            >{{ t('datasource.priority') }}
            <el-icon>
              <Icon :name="showPriority ? 'icon_down_outlined' : 'icon_down_outlined-1'"></Icon>
            </el-icon>
          </span>
          <template v-if="showPriority">
            <el-row :gutter="24" class="mb16">
              <el-col :span="12">
                <el-form-item
                  :label="t('datasource.initial_pool_size')"
                  prop="configuration.initialPoolSize"
                >
                  <el-input-number
                    v-model="form.configuration.initialPoolSize"
                    controls-position="right"
                    autocomplete="off"
                    :placeholder="t('common.inputText') + t('datasource.initial_pool_size')"
                    type="number"
                    :min="0"
                  />
                </el-form-item>
              </el-col>
              <el-col :span="12">
                <el-form-item
                  :label="t('datasource.min_pool_size')"
                  prop="configuration.minPoolSize"
                >
                  <el-input-number
                    v-model="form.configuration.minPoolSize"
                    controls-position="right"
                    autocomplete="off"
                    :placeholder="t('common.inputText') + t('datasource.min_pool_size')"
                    type="number"
                    :min="0"
                  />
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="24">
              <el-col :span="12">
                <el-form-item
                  :label="t('datasource.max_pool_size')"
                  prop="configuration.maxPoolSize"
                >
                  <el-input-number
                    v-model="form.configuration.maxPoolSize"
                    controls-position="right"
                    autocomplete="off"
                    :placeholder="t('common.inputText') + t('datasource.max_pool_size')"
                    type="number"
                    :min="0"
                  />
                </el-form-item>
              </el-col>
              <el-col :span="12">
                <el-form-item
                  :label="`${t('datasource.query_timeout')}(${t('common.second')})`"
                  prop="configuration.queryTimeout"
                >
                  <el-input-number
                    v-model="form.configuration.queryTimeout"
                    controls-position="right"
                    autocomplete="off"
                    :placeholder="t('common.inputText') + t('datasource.query_timeout')"
                    type="number"
                    :min="0"
                  />
                </el-form-item>
              </el-col>
            </el-row>
          </template>
        </template>
      </el-form>
      <el-form
        ref="dsApiForm"
        :model="form"
        :rules="apiRule"
        label-width="180px"
        label-position="top"
        require-asterisk-position="right"
      >
        <!--        API update setting -->
        <el-form-item
          :label="t('datasource.update_type')"
          prop="syncSetting.updateType"
          v-if="activeStep === 2 && form.type === 'API'"
        >
          <el-radio-group v-model="form.syncSetting.updateType">
            <el-radio label="all_scope">{{ t('datasource.all_scope') }}</el-radio>
            <el-radio label="add_scope"> {{ t('datasource.add_scope') }}</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item
          :label="t('datasource.sync_rate')"
          prop="syncSetting.syncRate"
          v-if="activeStep === 2 && form.type === 'API'"
        >
          <el-radio-group v-model="form.syncSetting.syncRate" @change="onRateChange">
            <el-radio label="RIGHTNOW">立即更新</el-radio>
            <el-radio label="CRON">{{ t('datasource.cron_config') }}</el-radio>
            <el-radio label="SIMPLE_CRON">{{ t('datasource.simple_cron') }}</el-radio>
          </el-radio-group>
        </el-form-item>
        <div
          v-if="activeStep === 2 && form.type === 'API' && form.syncSetting.syncRate !== 'RIGHTNOW'"
          class="execute-rate-cont"
        >
          <el-form-item
            :label="t('auth.set_rules')"
            v-if="form.syncSetting.syncRate === 'SIMPLE_CRON'"
            prop="syncSetting.simpleCronValue"
          >
            <div class="simple-cron">
              {{ t('common.every') }}
              <el-input-number
                v-model="form.syncSetting.simpleCronValue"
                controls-position="right"
                :min="1"
                @change="onSimpleCronChange()"
              />
              <el-select
                v-model="form.syncSetting.simpleCronType"
                filterable
                @change="onSimpleCronChange()"
              >
                <el-option :label="t('common.minute')" value="minute" />
                <el-option :label="t('common.hour')" value="hour" />
                <el-option :label="t('common.day')" value="day" />
              </el-select>
              更新一次
            </div>
          </el-form-item>
          <el-form-item
            v-if="form.syncSetting.syncRate === 'CRON'"
            prop="syncSetting.cron"
            :label="t('common.cron_exp')"
          >
            <el-popover :width="834" v-model="cronEdit" trigger="click">
              <template #default>
                <div style="width: 814px; height: 400px; overflow-y: auto">
                  <cron
                    v-if="showCron"
                    v-model="form.syncSetting.cron"
                    :is-rate="form.syncRate === 'CRON'"
                    @close="cronEdit = false"
                  />
                </div>
              </template>
              <template #reference>
                <el-input
                  v-model="form.syncSetting.cron"
                  style="width: 50%"
                  @click="cronEdit = true"
                />
              </template>
            </el-popover>
          </el-form-item>
          <el-form-item
            v-if="form.syncSetting.syncRate !== 'RIGHTNOW'"
            :label="t('datasource.start_time')"
            prop="syncSetting.startTime"
          >
            <el-date-picker
              v-model="form.syncSetting.startTime"
              class="de-date-picker"
              :prefix-icon="Calendar"
              type="datetime"
              :placeholder="t('datasource.start_time')"
            />
          </el-form-item>
          <el-form-item
            v-if="form.syncSetting.syncRate !== 'RIGHTNOW'"
            :label="t('datasource.end_time')"
            prop="syncSetting.endLimit"
          >
            <el-radio-group v-model="form.syncSetting.endLimit">
              <el-radio label="0">{{ t('datasource.no_limit') }}</el-radio>
              <el-radio label="1"> {{ t('datasource.set_end_time') }}</el-radio>
            </el-radio-group>
            <div style="width: 100%">
              <el-date-picker
                v-if="form.syncSetting.endLimit === '1'"
                v-model="form.syncSetting.endTime"
                class="de-date-picker"
                :prefix-icon="Calendar"
                type="datetime"
                :placeholder="t('datasource.end_time')"
              />
            </div>
          </el-form-item>
        </div>
      </el-form>

      <api-http-request-draw @return-item="returnItem" ref="editApiItem"></api-http-request-draw>
    </div>
  </div>
</template>

<style lang="less" scoped>
.editor-detail {
  width: 100%;
  display: flex;
  justify-content: center;
  .ed-radio {
    height: 22px;
  }

  .mb16 {
    :deep(.ed-form-item) {
      margin-bottom: 16px;
    }
  }

  .execute-rate-cont {
    background: #f5f6f7;
    border-radius: 4px;
    padding: 16px;
    margin-top: -8px;
  }

  .de-select {
    width: 100%;
  }
  .ed-input-number {
    width: 100%;
  }

  :deep(.is-controls-right > span) {
    background: #fff;
  }

  .de-expand {
    font-family: '阿里巴巴普惠体 3.0 55 Regular L3';
    font-size: 14px;
    font-weight: 400;
    line-height: 22px;
    color: #3370ff;
    cursor: pointer;
    display: inline-flex;
    align-items: center;

    .ed-icon {
      margin-left: 4px;
    }
  }

  :deep(.ed-date-editor.ed-input) {
    .ed-input__wrapper {
      width: 100%;
    }
    width: 100%;
  }
  .simple-cron {
    height: 32px;
    .ed-select,
    .ed-input-number {
      width: 140px;
      margin: 0 8px;
    }
  }
  .detail-inner {
    width: 800px;
    padding-top: 8px;

    .description-text {
      :deep(.ed-textarea__inner) {
        height: 92px;
      }
    }

    .base-info {
      margin: 24px 0 16px 0;
    }

    .table-info-mr {
      margin: 28px 0 12px 0;
    }

    .info-update {
      height: 22px;
      width: 100%;
      display: flex;
      align-items: center;
      font-family: '阿里巴巴普惠体 3.0 55 Regular L3';
      font-size: 14px;
      font-style: normal;
      font-weight: 400;
      line-height: 22px;
      justify-content: center;

      .update-info-line {
        width: 208px;
        height: 1px;
        background: #bcbdbf;
        margin: 0 8px;
      }

      .info-text,
      .update-text {
        padding-left: 16px;
        position: relative;
        color: #1f2329;
        font-weight: 400;
        font-family: '阿里巴巴普惠体 3.0 55 Regular L3';
        font-size: 14px;
        font-style: normal;
        line-height: 22px;
        &::before {
          width: 8px;
          height: 8px;
          content: '';
          position: absolute;
          left: 0;
          top: 50%;
          transform: translateY(-50%);
          border: 1px solid #3370ff;
          border-radius: 50%;
        }

        &.active {
          font-weight: 500;
        }

        &.active::before {
          border: none;
          background: #3370ff;
        }
      }
    }

    .detail-operate {
      text-align: right;
      padding: 8px 0;
    }

    .flex-space {
      display: flex;
      align-items: center;
    }
  }
}

.api-card-content {
  display: flex;
  flex-wrap: wrap;
  margin-left: -16px;
}
.api-card {
  height: 120px;
  width: 392px;
  border-radius: 4px;
  border: 1px solid var(--deCardStrokeColor, #dee0e3);
  border-radius: 4px;
  margin: 0 0 16px 16px;
  padding: 16px;
  font-family: '阿里巴巴普惠体 3.0 55 Regular L3';
  cursor: pointer;

  &:hover {
    border-color: #3370ff;
  }
  .name {
    font-size: 16px;
    font-weight: 500;
    margin-right: 8px;
    max-width: 80%;
  }
  .req-title,
  .req-value {
    display: flex;
    font-size: 14px;
    font-weight: 400;
    :nth-child(1) {
      width: 100px;
    }
    :nth-child(2) {
      margin-left: 24px;
      max-width: 230px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
  }
  .req-title {
    color: var(--deTextSecondary, #646a73);
    margin: 16px 0 4px 0;
  }
  .req-value {
    color: var(--deTextPrimary, #1f2329);
  }
  .de-copy-icon {
    margin-right: 16px;
    color: var(--deTextSecondary, #646a73);
  }
  .de-delete-icon {
    cursor: pointer;
  }
  .de-tag {
    display: inline-flex;
    justify-content: center;
    align-items: center;
    border-radius: 2px;
    padding: 1px 6px;
    height: 24px;
    &.invalid {
      color: #646a73;
      background: rgba(31, 35, 41, 0.1);
    }

    &.valid {
      color: green;
      background: rgba(52, 199, 36, 0.2);
    }
  }
}
</style>

<style lang="less">
.api-table-delete {
  padding: 20px 24px !important;
  display: flex;
  flex-wrap: wrap;
  .small {
    height: 28px;
    min-width: 48px !important;
  }

  .icon-warning {
    transform: translateY(3px);
  }

  .tips {
    font-family: '阿里巴巴普惠体 3.0 55 Regular L3';
    font-size: 14px;
    font-weight: 500;
    line-height: 22px;
    margin-left: 8.67px;
    color: var(--deTextPrimary, #1f2329);
  }
  i {
    font-size: 14.666666030883789px;
    color: var(--deWarning, #ff8800);
    line-height: 22px;
  }
  .foot {
    text-align: right;
    width: 100%;
    margin-top: 16px;
  }
}

.schema-label {
  .ed-form-item__label {
    display: flex !important;
    justify-content: space-between;
    padding-right: 0;
    &::after {
      display: none;
    }
    .name {
      .required::after {
        content: '*';
        color: #f54a45;
        margin-left: 2px;
      }
    }
  }
}
</style>

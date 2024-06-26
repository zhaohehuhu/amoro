
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
/-->

<template>
  <div class="tables-wrap">
    <div v-if="!isSecondaryNav" class="tables-content">
      <div class="g-flex-jsb">
        <div class="g-flex-col">
          <div class="g-flex">
            <span :title="baseInfo.tableName" class="table-name g-text-nowrap">{{baseInfo.tableName}}</span>
            <span v-if="!isIceberg" class="create-time">{{ `${$t('createTime')}: ${baseInfo.createTime}` }}</span>
          </div>
          <div class="table-info g-flex-ac">
            <p>{{`${$t('table')}${$t('size')}`}}: <span class="text-color">{{baseInfo.size}}</span></p>
            <a-divider type="vertical" />
            <p>{{$t('file')}}:  <span class="text-color">{{baseInfo.file}}</span></p>
            <a-divider type="vertical" />
            <p>{{$t('averageFileSize')}}: <span class="text-color">{{baseInfo.averageFile}}</span></p>
            <a-divider type="vertical" />
            <p>{{$t('tableFormat')}}: <span class="text-color">{{baseInfo.tableFormat}}</span></p>
          </div>
        </div>
      </div>
      <div class="content">
        <a-tabs v-model:activeKey="activeKey" destroyInactiveTabPane @change="onChangeTab">
          <a-tab-pane key="Details" tab="Details" forceRender>
            <u-details @setBaseDetailInfo="setBaseDetailInfo" ref="detailRef"/>
          </a-tab-pane>
          <a-tab-pane v-if="detailLoaded" key="Files" tab="Files">
            <u-files :hasPartition="baseInfo.hasPartition"/>
          </a-tab-pane>
          <a-tab-pane v-for="tab in tabConfigs" :key="tab.key" :tab="`${tab.label}`">
            <component :is="`U${tab.key}`"></component>
          </a-tab-pane>
        </a-tabs>
      </div>
    </div>
    <!-- Create table secondary page -->
    <router-view v-else @goBack="goBack"></router-view>
  </div>
</template>

<script lang="ts">
// TODO: replace to antv-4. After all replacements are completed, switch to automatic import.
import { Tabs, TabPane, Divider } from 'ant-design-vue'

import { defineComponent, reactive, toRefs, watch, shallowReactive, computed, onMounted, ref, nextTick } from 'vue'
import UDetails from './components/Details.vue'
import UFiles from './components/Files.vue'
import UOperations from './components/Operations.vue'
import USnapshots from './components/Snapshots.vue'
import UOptimizing from './components/Optimizing.vue'
import { useRoute, useRouter } from 'vue-router'
import useStore from '@/store/index'
import { IBaseDetailInfo } from '@/types/common.type'

export default defineComponent({
  name: 'Tables',
  components: {
    UDetails,
    UFiles,
    UOperations,
    USnapshots,
    UOptimizing,

    ATabs: Tabs,
    ATabPane: TabPane,
    ADivider: Divider
  },
  setup() {
    const router = useRouter()
    const route = useRoute()
    const store = useStore()

    const detailRef = ref()

    const tabConfigs = shallowReactive([
      { key: 'Snapshots', label: 'Snapshots' },
      { key: 'Optimizing', label: 'Optimizing' },
      { key: 'Operations', label: 'Operations' }
    ])

    const state = reactive({
      activeKey: 'Details',
      isSecondaryNav: false,
      baseInfo: {
        tableType: '',
        tableName: '',
        createTime: '',
        size: '',
        file: '',
        averageFile: '',
        tableFormat: '',
        hasPartition: false
      } as IBaseDetailInfo,
      detailLoaded: false
    })

    const isIceberg = computed(() => {
      return state.baseInfo.tableType === 'ICEBERG'
    })

    const setBaseDetailInfo = (baseInfo: IBaseDetailInfo) => {
      state.detailLoaded = true
      state.baseInfo = { ...baseInfo }
    }

    const onChangeTab = (key: string | number) => {
      const query = { ...route.query }
      query.tab = key.toString()
      router.replace({ query: { ...query } })
    }

    const hideTablesMenu = () => {
      store.updateTablesMenu(false)
    }

    const goBack = () => {
      state.isSecondaryNav = false
      router.back()
    }

    watch(
      () => route.path,
      () => {
        state.isSecondaryNav = !!(route.path.indexOf('create') > -1)
      }, { immediate: true }
    )

    watch(
      () => route.query,
      (value, oldVal) => {
        const { catalog, db, table } = value
        const { catalog: oldCatalog, db: oldDb, table: oldTable } = oldVal
        if (`${catalog}${db}${table}` !== `${oldCatalog}${oldDb}${oldTable}`) {
          state.activeKey = 'Details'
          return
        }
        state.activeKey = value.tab as string
      }
    )

    onMounted(() => {
      state.activeKey = (route.query?.tab as string) || 'Details'
      nextTick(() => {
        if (detailRef.value) {
          detailRef.value.getTableDetails()
        }
      })
    })

    return {
      ...toRefs(state),
      tabConfigs,
      store,
      isIceberg,
      setBaseDetailInfo,
      hideTablesMenu,
      goBack,
      onChangeTab
    }
  }
})

</script>

<style lang="less" scoped>
.tables-wrap {
  font-size: 14px;
  border: 1px solid #e8e8f0;
  padding: 12px 0;
  min-height: 100%;
  .create-time {
    margin-top: 12px;
  }
  .tables-menu-wrap {
    position: fixed;
    width: 100%;
    height: 100%;
    top: 0;
    left: 200px;
    z-index: 100;
  }
  .table-name {
    font-size: 24px;
    line-height: 1.5;
    margin-right: 16px;
    max-width: 400px;
    padding-left: 24px;
  }
  .table-info {
    padding: 12px 24px 0 24px;
    .text-color {
      color: #7CB305;
    }
  }
  .table-edit {
    font-size: 18px;
    padding-right: 12px;
  }
  :deep(.ant-tabs-nav) {
    padding-left: 12px;
    margin-bottom: 0;
  }
}
</style>

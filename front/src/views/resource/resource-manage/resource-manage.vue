<script setup lang="ts">
import { ref, watch, computed, provide } from 'vue';

import HostManage from './children/manage/host-manage.vue';
import VpcManage from './children/manage/vpc-manage.vue';
import SubnetManage from './children/manage/subnet-manage.vue';
import SecurityManage from './children/manage/security-manage.vue';
import DriveManage from './children/manage/drive-manage.vue';
import IpManage from './children/manage/ip-manage.vue';
import RoutingManage from './children/manage/routing-manage.vue';
import ImageManage from './children/manage/image-manage.vue';
import NetworkInterfaceManage from './children/manage/network-interface-manage.vue';
// import AccountSelector from '@/components/account-selector/index.vue';
import { DISTRIBUTE_STATUS_LIST } from '@/constants';
import { useDistributionStore } from '@/store/distribution';
import EipForm from '@/views/business/forms/eip/index.vue';
import subnetForm from '@/views/business/forms/subnet/index.vue';
import securityForm from '@/views/business/forms/security/index.vue';
import firewallForm from '@/views/business/forms/firewall';
import BkTab, { BkTabPanel } from 'bkui-vue/lib/tab';
import { RouterView, useRouter, useRoute } from 'vue-router';

import { RESOURCE_TYPES, RESOURCE_TABS } from '@/common/constant';

import { useI18n } from 'vue-i18n';
import useSteps from './hooks/use-steps';

import type { FilterType } from '@/typings/resource';

import { useAccountStore } from '@/store';

import { useVerify } from '@/hooks';
import { useResourceAccountStore } from '@/store/useResourceAccountStore';
import { InfoBox } from 'bkui-vue';

// use hooks
const { t } = useI18n();
const router = useRouter();
const route = useRoute();
const accountStore = useAccountStore();
const { isShowDistribution, ResourceDistribution } = useSteps();

const isResourcePage = computed(() => {
  // 资源下没有业务ID
  return !accountStore.bizs;
});

// 权限hook
const {
  showPermissionDialog,
  handlePermissionConfirm,
  handlePermissionDialog,
  handleAuth,
  permissionParams,
  authVerifyData,
} = useVerify();

const resourceAccountStore = useResourceAccountStore();

// 搜索过滤相关数据
const filter = ref({ op: 'and', rules: [] });
const accountId = ref('');
const status = ref('all');
const op = ref('eq');
const accountFilter = ref<FilterType>({
  op: 'and',
  rules: [{ field: 'type', op: 'eq', value: 'resource' }],
});
const isShowSideSlider = ref(false);
const componentRef = ref();
const securityType = ref('group');
const isEdit = ref(false);
const formDetail = ref({});
const activeResourceTab = ref(RESOURCE_TABS[0].key);

provide('securityType', securityType);

const formMap = {
  ip: EipForm,
  subnet: subnetForm,
  security: securityForm,
};

const renderForm = computed(() => {
  return Object.keys(formMap).reduce((acc, cur) => {
    if (route.query.type === cur) {
      if (cur === 'security' && securityType.value === 'gcp') acc = firewallForm;
      else acc = formMap[cur];
    }
    return acc;
  }, {});
});

// 组件map
const componentMap = {
  host: HostManage,
  vpc: VpcManage,
  subnet: SubnetManage,
  security: SecurityManage,
  drive: DriveManage,
  ip: IpManage,
  routing: RoutingManage,
  image: ImageManage,
  'network-interface': NetworkInterfaceManage,
};

// 标签相关数据
const tabs = RESOURCE_TYPES.map((type) => {
  return {
    name: type.type,
    type: t(type.name),
    component: componentMap[type.type],
  };
});
const activeTab = ref((route.query.type as string) || tabs[0].type);

const filterData = (key: string, val: string | number) => {
  if (!filter.value.rules.length) {
    if (val === 1) {
      // 已分配标志
      op.value = 'neq';
    }
    filter.value.rules.push({
      field: key,
      op: op.value,
      value: -1,
    });
  } else {
    filter.value.rules.forEach((e: any) => {
      console.log(e.field, key, e.field === key);
      if (e.field === key) {
        e.op = val === 1 ? 'neq' : 'eq';
        return;
      }
      if (filter.value.rules.length === 2) return;
      if (val === 1) {
        // 已分配标志
        op.value = 'neq';
      }
      filter.value.rules.push({
        field: key,
        op: op.value,
        value: -1,
      });
    });
  }
};

const handleAdd = () => {
  // ['host', 'vpc', 'drive', ||| 'security', 'subnet', 'ip']
  switch (activeTab.value) {
    case 'host':
      router.push({
        path: '/resource/service-apply/cvm',
        query: route.query,
      });
      break;
    case 'vpc':
      router.push({
        path: '/resource/service-apply/vpc',
        query: route.query,
      });
      break;
    case 'drive':
      router.push({
        path: '/resource/service-apply/disk',
        query: route.query,
      });
      break;
    case 'subnet':
      router.push({
        path: '/resource/service-apply/subnet',
        query: route.query,
      });
      break;
    default:
      isShowSideSlider.value = true;
  }
};

const handleTabChange = (val: 'group' | 'gcp') => {
  securityType.value = val;
};

watch(
  () => route.path,
  (path) => {
    RESOURCE_TABS.forEach(({ key }) => {
      const reg = new RegExp(key);
      if (reg.test(path)) {
        activeResourceTab.value = key;
      }
    });
  },
  {
    immediate: true,
  },
);

// 搜索数据
watch(
  () => accountId.value,
  (val) => {
    if (val) {
      if (!filter.value.rules.length) {
        filter.value.rules.push({
          field: 'account_id',
          op: 'eq',
          value: val,
        });
      } else {
        filter.value.rules.forEach((e: any) => {
          if (e.field === 'account_id') {
            e.value = val;
          } else {
            if (filter.value.rules.length === 2) return;
            filter.value.rules.push({
              field: 'account_id',
              op: 'eq',
              value: val,
            });
          }
        });
      }
    } else {
      filter.value.rules = filter.value.rules.filter((e: any) => e.field !== 'account_id');
    }
    useDistributionStore().setCloudAccountId(val);
  },
  {
    immediate: true,
  },
);

watch(
  () => status.value,
  (val) => {
    if (val === 'all' || !val) {
      filter.value.rules = filter.value.rules.filter((e: any) => e.field !== 'bk_biz_id');
    } else {
      filterData('bk_biz_id', val);
    }
  },
);

// 状态保持
watch(activeTab, () => {
  router.replace({
    query: {
      ...route.query,
      type: activeTab.value,
    },
  });
});

watch(
  () => resourceAccountStore.resourceAccount,
  (resourceAccount: any) => {
    if (resourceAccount?.id) accountId.value = resourceAccount.id;
    else accountId.value = '';
  },
  {
    deep: true,
    immediate: true,
  },
);

watch(
  () => activeResourceTab.value,
  (val) => {
    router.push({
      path: val,
      query: route.query,
    });
  },
  {
    immediate: true,
  },
);

const getResourceAccountList = async () => {
  try {
    const params = {
      filter: accountFilter.value,
      page: {
        start: 0,
        limit: 100,
      },
    };
    const res = await accountStore.getAccountList(params);
    accountStore.updateAccountList(res?.data?.details); // 账号数据   用于筛选
  } catch (error) {}
};

const handleCancel = () => {
  isShowSideSlider.value = false;
  isEdit.value = false;
};

// 新增成功 刷新列表
const handleSuccess = () => {
  handleCancel();
  if (Array.isArray(componentRef.value)) componentRef.value[0].fetchComponentsData();
  else componentRef.value.fetchComponentsData();
};

const handleEdit = (detail: any) => {
  formDetail.value = detail;
  isEdit.value = true;
  isShowSideSlider.value = true;
};

const handleBeforeClose = () => {
  InfoBox({
    title: '请确认是否关闭侧栏？',
    subTitle: '关闭后，内容需要重新填写！',
    theme: 'warning',
    onConfirm() {
      handleCancel();
    },
  });
};

getResourceAccountList();
</script>

<template>
  <div>
    <!-- <section class="flex-center resource-header">
      <section class="flex-center" v-if="activeTab !== 'image'">
        <div class="mr10">{{t('云账号')}}</div>
        <div class="mr20">
          <account-selector
            :is-resource-page="isResourcePage"
            :filter="accountFilter"
            v-model="accountId"
          />
        </div>
      </section>
      <section class="flex-center" v-if="activeTab !== 'image'">
        <div class="mr10">{{t('分配状态')}}</div>
        <div class="mr20">
          <bk-select
            v-model="status"
          >
            <bk-option
              v-for="(item, index) in DISTRIBUTE_STATUS_LIST"
              :key="index"
              :value="item.value"
              :label="item.label"
            />
          </bk-select>
        </div>
      </section>
      <section class="flex-center">
        <bk-button
          theme="primary"
          class="ml10"
          @click="handleDistribution"
        >
          {{ t('快速分配') }}
        </bk-button>
      </section>
      <section class="flex-center">
        <bk-checkbox
          v-model="isAccurate"
        >
          {{ t('精确') }}
        </bk-checkbox>
        <bk-search-select
          class="search-filter ml10"
          clearable
          :data="searchData"
          v-model="searchValue"
        />
      </section>
    </section> -->

    <div class="navigation-resource">
      <div class="card-layout">
        <p class="resource-title">
          <span class="main-account-name">
            {{ resourceAccountStore?.resourceAccount?.name || "全部账号" }}
          </span>
          <span v-if="resourceAccountStore?.resourceAccount?.id" class="main-account-id pl20">
            主账号ID：{{ resourceAccountStore.resourceAccount.id }}
          </span>
        </p>
        <BkTab
          class="resource-tab-wrap ml15"
          type="unborder-card"
          v-model:active="activeResourceTab"
        >
          <BkTabPanel
            v-for="item of RESOURCE_TABS"
            :label="item.label"
            :key="item.key"
            :name="item.key"
            :disabled="
              item.key === '/resource/resource/account' &&
                !resourceAccountStore?.resourceAccount?.id
            "
          />
        </BkTab>
      </div>
    </div>

    <div v-if="activeResourceTab === '/resource/resource/'">
      <bk-alert
        theme="error"
        closable
        class="error-message-alert"
        v-if="resourceAccountStore?.resourceAccount?.sync_failed_reason?.length"
      >
        <template #title>
          {{ resourceAccountStore?.resourceAccount?.sync_failed_reason }}
        </template>
      </bk-alert>
      <bk-tab
        v-model:active="activeTab"
        type="card-grid"
        class="resource-main g-scroller"
      >
        <template #setting>
          <div style="margin: 0 10px">
            <bk-select v-model="status" :clearable="false" class="w80">
              <bk-option
                v-for="(item, index) in DISTRIBUTE_STATUS_LIST"
                :key="index"
                :value="item.value"
                :label="item.label"
              />
            </bk-select>
          </div>
        </template>
        <bk-tab-panel
          v-for="item in tabs"
          :key="item.name"
          :name="item.name"
          :label="item.type"
        >
          <component
            v-if="item.name === activeTab"
            :is="item.component"
            :filter="filter"
            :where-am-i="activeTab"
            :is-resource-page="isResourcePage"
            :auth-verify-data="authVerifyData"
            @auth="(val: string) => {
              handleAuth(val)
            }"
            @tabchange="handleTabChange"
            ref="componentRef"
            @edit="handleEdit"
          >
            <span
              v-if="
                ['host', 'vpc', 'drive', 'security', 'subnet', 'ip'].includes(
                  activeTab,
                )
              "
            >
              <bk-button
                theme="primary"
                class="new-button"
                :class="{ 'hcm-no-permision-btn': !authVerifyData?.permissionAction?.iaas_resource_create }"
                @click="() => {
                  if (!authVerifyData?.permissionAction?.iaas_resource_create) {
                    handleAuth('iaas_resource_create');
                  } else {
                    handleAdd();
                  }
                }"
              >
                {{ activeTab === 'host' ? '购买' : '新建' }}
              </bk-button>
            </span>
          </component>
        </bk-tab-panel>
      </bk-tab>

      <bk-sideslider
        v-model:isShow="isShowSideSlider"
        width="800"
        title="新增"
        quick-close
        :before-close="handleBeforeClose"
      >
        <template #default>
          <component
            :is="renderForm"
            :filter="filter"
            @cancel="handleCancel"
            @success="handleSuccess"
            :is-edit="isEdit"
            :detail="formDetail"
          ></component>
        </template>
      </bk-sideslider>

      <resource-distribution
        v-model:is-show="isShowDistribution"
        :choose-resource-type="true"
        :title="t('快速分配')"
        :data="[]"
      />

      <permission-dialog
        v-model:is-show="showPermissionDialog"
        :params="permissionParams"
        @cancel="handlePermissionDialog"
        @confirm="handlePermissionConfirm"
      ></permission-dialog>
    </div>

    <RouterView v-else></RouterView>
  </div>
</template>

<style lang="scss" scoped>
.flex-center {
  display: flex;
  align-items: center;
}
.resource-header {
  background: #fff;
  box-shadow: 1px 2px 3px 0 rgb(0 0 0 / 5%);
  padding: 20px;
}
.resource-main {
  // margin-top: 20px;
  box-shadow: 1px 2px 3px 0 rgb(0 0 0 / 5%);
  height: calc(100vh - 200px);

  :deep(.bk-tab-header) {
    line-height: normal !important;

    .bk-tab-header-item {
      padding: 0 24px;
    }
  }

  :deep(.bk-tab-content) {
    // border-left: 1px solid #dcdee5;
    // border-right: 1px solid #dcdee5;
    border-bottom: 1px solid #dcdee5;
    padding: 20px;
  }
}
.search-filter {
  width: 500px;
}
.new-button {
  width: 64px;
}
.w80 {
  width: 80px;
}
.navigation-resource {
  min-height: 88px;
  margin: -24px -24px 24px -24px;
}
.card-layout {
  background: #fff;
  border-bottom: 1px solid #dcdee5;
}
.resource-title {
  font-family: MicrosoftYaHei;
  font-size: 16px;
  color: #313238;
  letter-spacing: 0;
  line-height: 24px;
  padding: 14px 0 9px 24px;
  display: flex;
  align-items: center;

  .main-account-id {
    font-size: 14px;
    color: #313238;
  }
}
.bk-tab-content {
  padding: 0 !important;
}
.error-message-alert {
  margin: -8px 0 16px 0;
}
</style>

<style lang="scss">
.delete-resource-infobox, .recycle-resource-infobox {
  .bk-info-sub-title {
    word-break: break-all;
  }
}
</style>

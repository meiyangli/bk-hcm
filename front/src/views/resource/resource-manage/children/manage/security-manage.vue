<script setup lang="ts">
import type {
  // PlainObject,
  FilterType,
} from '@/typings/resource';
import { GcpTypeEnum, CloudType } from '@/typings';
import { Button, InfoBox, Message, Tag } from 'bkui-vue';
import { useResourceStore, useAccountStore } from '@/store';
import { ref, h, PropType, watch, reactive, defineExpose, computed } from 'vue';

import { useI18n } from 'vue-i18n';
import { useRouter, useRoute } from 'vue-router';
import useQueryCommonList from '@/views/resource/resource-manage/hooks/use-query-list-common';
import useColumns from '@/views/resource/resource-manage/hooks/use-columns';
import useFilter from '@/views/resource/resource-manage/hooks/use-filter';
import { useRegionsStore } from '@/store/useRegionsStore';
import { VendorEnum } from '@/common/constant';
import { cloneDeep } from 'lodash-es';
import { useBusinessMapStore } from '@/store/useBusinessMap';
import useSelection from '../../hooks/use-selection';
import {
  BatchDistribution,
  DResourceType,
} from '@/views/resource/resource-manage/children/dialog/batch-distribution';

const props = defineProps({
  filter: {
    type: Object as PropType<FilterType>,
  },
  isResourcePage: {
    type: Boolean,
  },
  authVerifyData: {
    type: Object as PropType<any>,
  },
  whereAmI: {
    type: String,
  },
});

// use hooks
const { t } = useI18n();

const { getRegionName } = useRegionsStore();
const router = useRouter();

const route = useRoute();

const activeType = ref('group');
const fetchUrl = ref<string>('security_groups/list');
const resourceStore = useResourceStore();
const accountStore = useAccountStore();

const emit = defineEmits(['auth', 'handleSecrityType', 'edit', 'tabchange']);
const { columns, generateColumnsSettings } = useColumns('group');
const businessMapStore = useBusinessMapStore();

const state = reactive<any>({
  datas: [],
  pagination: {
    current: 1,
    limit: 10,
    count: 0,
  },
  isLoading: false,
  handlePageChange: () => {},
  handlePageSizeChange: () => {},
  columns,
  params: {
    fetchUrl: 'security_groups',
    columns: 'group',
  },
});

const { searchData, searchValue, filter } = useFilter(props);

const {
  datas,
  pagination,
  isLoading,
  handlePageChange,
  handlePageSizeChange,
  getList,
} = useQueryCommonList(
  {
    ...props,
    filter: filter.value,
  },
  fetchUrl,
);

const selectSearchData = computed(() => {
  return [
    {
      name: activeType.value === 'group' ? '安全组 ID' : '防火墙 ID',
      id: 'cloud_id',
    },
    ...searchData.value,
    ...[
      {
        name: '云地域',
        id: 'region',
      },
    ],
  ];
});

// eslint-disable-next-line max-len
state.datas = datas;
state.isLoading = isLoading;
state.pagination = pagination;
state.handlePageChange = handlePageChange;
state.handlePageSizeChange = handlePageSizeChange;

// 状态保持
watch(
  () => activeType.value,
  (v) => {
    console.log(1);
    state.isLoading = true;
    state.pagination.current = 1;
    state.pagination.limit = 10;
    handleSwtichType(v);
  },
);

const handleSwtichType = async (type: string) => {
  if (type === 'gcp') {
    fetchUrl.value = 'vendors/gcp/firewalls/rules/list';
    state.params.fetchUrl = 'vendors/gcp/firewalls/rules';
    state.params.columns = 'gcp';
  } else {
    fetchUrl.value = 'security_groups/list';
    state.params.fetchUrl = 'security_groups';
    state.params.columns = 'group';
  }
  emit('handleSecrityType', type);
};

// 抛出请求数据的方法，新增成功使用
const fetchComponentsData = () => {
  getList();
};

// 初始化
getList();

defineExpose({ fetchComponentsData });
const isRowSelectEnable = ({ row, isCheckAll }: DoublePlainObject) => {
  if (isCheckAll) return true;
  return isCurRowSelectEnable(row);
};
const isCurRowSelectEnable = (row: any) => {
  if (!props.isResourcePage) return true;
  if (row.id) {
    return row.bk_biz_id === -1;
  }
};
const { selections, handleSelectionChange, resetSelections } = useSelection();

const groupColumns = [
  {
    type: 'selection',
    width: '100',
    onlyShowOnList: true,
  },
  {
    label: '安全组 ID',
    field: 'cloud_id',
    width: '120',
    sort: true,
    isDefaultShow: true,
    render({ data }: any) {
      return h(
        Button,
        {
          text: true,
          theme: 'primary',
          disabled: data.bk_biz_id !== -1 && props.isResourcePage,
          onClick() {
            const routeInfo: any = {
              query: {
                ...route.query,
                id: data.id,
                vendor: data.vendor,
              },
            };
            // 业务下
            if (route.path.includes('business')) {
              routeInfo.query.bizs = accountStore.bizs;
              Object.assign(routeInfo, {
                name: 'securityBusinessDetail',
              });
            } else {
              Object.assign(routeInfo, {
                name: 'resourceDetail',
                params: {
                  type: 'security',
                },
              });
            }
            router.push(routeInfo);
          },
        },
        [data.cloud_id || '--'],
      );
    },
  },
  {
    label: '安全组名称',
    field: 'name',
    sort: true,
    isDefaultShow: true,
  },
  {
    label: t('云厂商'),
    field: 'vendor',
    sort: true,
    isDefaultShow: true,
    render({ data }: any) {
      return h('span', {}, [CloudType[data.vendor]]);
    },
  },
  {
    label: t('地域'),
    field: 'region',
    sort: true,
    isDefaultShow: true,
    render: ({ data }: { data: { vendor: VendorEnum; region: string } }) => {
      return getRegionName(data.vendor, data.region);
    },
  },
  {
    label: t('描述'),
    field: 'memo',
    isDefaultShow: true,
    render: ({ ceil }: any) => (ceil ? ceil : '--'),
  },
  {
    label: '是否分配',
    field: 'bk_biz_id',
    sort: true,
    isOnlyShowInResource: true,
    isDefaultShow: true,
    render: ({ data }: { data: { bk_biz_id: number }; cell: number }) => {
      return h(
        Tag,
        {
          theme: data.bk_biz_id === -1 ? false : 'success',
        },
        [data.bk_biz_id === -1 ? '未分配' : '已分配'],
      );
    },
  },
  {
    label: '所属业务',
    field: 'bk_biz_id2',
    isOnlyShowInResource: true,
    render({ data }: any) {
      return h('span', {}, [
        data.bk_biz_id === -1
          ? t('未分配')
          : businessMapStore.businessMap.get(data.bk_biz_id),
      ]);
    },
  },
  {
    label: t('账号 ID'),
    field: 'account_id',
    sort: true,
  },
  // {
  //   label: t('资源 ID'),
  //   field: 'cloud_id',
  //   sort: true,
  // },
  // {
  //   label: t('关联模板'),
  //   field: '',
  // },
  {
    label: t('创建时间'),
    field: 'created_at',
    sort: true,
  },
  {
    label: t('修改时间'),
    field: 'updated_at',
    sort: true,
  },
  {
    label: t('操作'),
    field: 'operate',
    isDefaultShow: true,
    render({ data }: any) {
      return h('span', {}, [
        h(
          'span',
          {
            onClick() {
              emit(
                'auth',
                props.isResourcePage
                  ? 'iaas_resource_operate'
                  : 'biz_iaas_resource_operate',
              );
            },
          },
          [
            h(
              Button,
              {
                text: true,
                disabled:
                  !props.authVerifyData?.permissionAction[
                    props.isResourcePage
                      ? 'iaas_resource_operate'
                      : 'biz_iaas_resource_operate'
                  ]
                  || (data.bk_biz_id !== -1 && props.isResourcePage),
                theme: 'primary',
                onClick() {
                  const routeInfo: any = {
                    query: {
                      activeTab: 'rule',
                      id: data.id,
                      vendor: data.vendor,
                    },
                  };
                  // 业务下
                  if (route.path.includes('business')) {
                    Object.assign(routeInfo, {
                      name: 'securityBusinessDetail',
                    });
                  } else {
                    Object.assign(routeInfo, {
                      name: 'resourceDetail',
                      params: {
                        type: 'security',
                      },
                    });
                  }
                  router.push(routeInfo);
                },
              },
              [t('配置规则')],
            ),
          ],
        ),
        h(
          'span',
          {
            onClick() {
              emit(
                'auth',
                props.isResourcePage
                  ? 'iaas_resource_delete'
                  : 'biz_iaas_resource_delete',
              );
            },
          },
          [
            h(
              Button,
              {
                class: 'ml10',
                disabled:
                  !props.authVerifyData?.permissionAction[
                    props.isResourcePage
                      ? 'iaas_resource_delete'
                      : 'biz_iaas_resource_delete'
                  ]
                  || (data.bk_biz_id !== -1 && props.isResourcePage),
                text: true,
                theme: 'primary',
                onClick() {
                  securityHandleShowDelete(data);
                },
              },
              [t('删除')],
            ),
          ],
        ),
      ]);
    },
  },
];

const groupSettings = generateColumnsSettings(groupColumns);

const gcpColumns = [
  {
    type: 'selection',
    width: '100',
    onlyShowOnList: true,
  },
  {
    label: '防火墙 ID	',
    field: 'cloud_id',
    width: '120',
    sort: true,
    isDefaultShow: true,
    render({ data }: any) {
      return h(
        Button,
        {
          text: true,
          theme: 'primary',
          disabled: data.bk_biz_id !== -1,
          onClick() {
            const routeInfo: any = {
              query: {
                ...route.query,
                id: data.id,
              },
            };
            // 业务下
            if (route.path.includes('business')) {
              Object.assign(routeInfo, {
                name: 'gcpBusinessDetail',
              });
            } else {
              Object.assign(routeInfo, {
                name: 'resourceDetail',
                params: {
                  type: 'gcp',
                },
              });
            }
            router.push(routeInfo);
          },
        },
        [data.cloud_id || '--'],
      );
    },
  },
  // {
  //   label: t('资源 ID'),
  //   field: 'account_id',
  //   sort: true,
  // },
  {
    label: '防火墙名称',
    field: 'name',
    sort: true,
    isDefaultShow: true,
  },
  {
    label: t('云厂商'),
    field: 'vendor',
    sort: true,
    isDefaultShow: true,
    render() {
      return h('span', {}, [t('谷歌云')]);
    },
  },
  {
    label: '所属VPC',
    field: 'vpc_id',
    sort: true,
    isDefaultShow: true,
  },
  {
    label: t('优先级'),
    field: 'priority',
    sort: true,
    isDefaultShow: true,
  },
  {
    label: '流量方向',
    field: 'type',
    sort: true,
    isDefaultShow: true,
    render({ data }: any) {
      return h('span', {}, [GcpTypeEnum[data.type]]);
    },
  },
  {
    label: t('目标'),
    field: 'target_tags',
    sort: true,
    isDefaultShow: true,
    render({ data }: any) {
      return h('span', {}, [
        data.target_tags || data.target_service_accounts || '--',
      ]);
    },
  },
  // {
  //   label: t('过滤条件'),
  //   field: '',
  // },
  {
    label: t('协议/端口'),
    field: 'allowed_denied',
    sort: true,
    isDefaultShow: true,
    render({ data }: any) {
      return h(
        'span',
        {},
        data?.allowed || data?.denied
          ? (data?.allowed || data?.denied).map((e: any) => {
            return h('div', {}, `${e.protocol}:${e.port}`);
          })
          : '--',
      );
    },
  },
  {
    label: '是否分配',
    field: 'bk_biz_id',
    sort: true,
    isOnlyShowInResource: true,
    isDefaultShow: true,
    render: ({ data }: { data: { bk_biz_id: number }; cell: number }) => {
      return h(
        Tag,
        {
          theme: data.bk_biz_id === -1 ? false : 'success',
        },
        [data.bk_biz_id === -1 ? '未分配' : '已分配'],
      );
    },
  },
  {
    label: '所属业务',
    field: 'bk_biz_id2',
    isOnlyShowInResource: true,
    render({ data }: any) {
      return h('span', {}, [
        data.bk_biz_id === -1
          ? t('未分配')
          : businessMapStore.businessMap.get(data.bk_biz_id),
      ]);
    },
  },
  {
    label: t('创建时间'),
    field: 'created_at',
    sort: true,
  },
  {
    label: t('修改时间'),
    field: 'updated_at',
    sort: true,
  },
  {
    label: t('操作'),
    field: 'operator',
    isDefaultShow: true,
    render({ data }: any) {
      return h('span', {}, [
        h(
          'span',
          {
            onClick() {
              emit(
                'auth',
                props.isResourcePage
                  ? 'iaas_resource_operate'
                  : 'biz_iaas_resource_operate',
              );
            },
          },
          [
            h(
              Button,
              {
                text: true,
                theme: 'primary',
                disabled:
                  !props.authVerifyData?.permissionAction[
                    props.isResourcePage
                      ? 'iaas_resource_operate'
                      : 'biz_iaas_resource_operate'
                  ]
                  || (data.bk_biz_id !== -1 && props.isResourcePage),
                onClick() {
                  emit('edit', cloneDeep(data));
                },
              },
              [t('编辑')],
            ),
          ],
        ),
        h(
          'span',
          {
            onClick() {
              emit(
                'auth',
                props.isResourcePage
                  ? 'iaas_resource_operate'
                  : 'biz_iaas_resource_operate',
              );
            },
          },
          [
            h(
              Button,
              {
                class: 'ml10',
                text: true,
                disabled:
                  !props.authVerifyData?.permissionAction[
                    props.isResourcePage
                      ? 'iaas_resource_delete'
                      : 'biz_iaas_resource_delete'
                  ]
                  || (data.bk_biz_id !== -1 && props.isResourcePage),
                theme: 'primary',
                onClick() {
                  securityHandleShowDelete(data);
                },
              },
              [t('删除')],
            ),
          ],
        ),
      ]);
    },
  },
];

const gcpSettings = generateColumnsSettings(gcpColumns);

const types = [
  { name: 'group', label: t('安全组') },
  { name: 'gcp', label: t('GCP防火墙规则') },
];

const securityType = ref('group');

watch(
  () => securityType.value,
  (val) => {
    emit('tabchange', val);
  },
  {
    immediate: true,
  },
);

// const computedSettings = computed(() => {
//   const fields = [];
//   const columns = securityType.value === 'group' ? groupColumns : gcpColumns;
//   for (const column of columns) {
//     if (column.field && column.label) {
//       fields.push({
//         label: column.label,
//         field: column.field,
//         disabled: column.field === 'id',
//       });
//     }
//   }
//   return {
//     fields,
//     checked: fields.map(field => field.field),
//   };
// });

const securityHandleShowDelete = (data: any) => {
  InfoBox({
    title: '请确认是否删除',
    subTitle: `将删除【${data.cloud_id}${data.name ? ` - ${data.name}` : ''}】`,
    theme: 'danger',
    headerAlign: 'center',
    footerAlign: 'center',
    contentAlign: 'center',
    extCls: 'delete-resource-infobox',
    async onConfirm() {
      try {
        await resourceStore.deleteBatch(
          activeType.value === 'group'
            ? 'security_groups'
            : 'vendors/gcp/firewalls/rules',
          { ids: [data.id] },
        );
        getList();
        Message({
          message: t('删除成功'),
          theme: 'success',
        });
      } catch (error) {
        console.log(error);
      }
    },
  });
};
</script>

<template>
  <div>
    <section>
      <slot></slot>
      <BatchDistribution
        :selections="selections"
        :type="
          activeType === 'group'
            ? DResourceType.security_groups
            : DResourceType.firewall
        "
        :get-data="
          () => {
            getList();
            resetSelections();
          }
        "
      />
      <section class="flex-row align-items-center mt20">
        <bk-radio-group v-model="activeType" :disabled="state.isLoading">
          <bk-radio-button
            v-for="item in types"
            :key="item.name"
            :label="item.name"
            v-model="securityType"
          >
            {{ item.label }}
          </bk-radio-button>
        </bk-radio-group>
        <bk-search-select
          class="search-filter search-selector-container"
          clearable
          :conditions="[]"
          :data="selectSearchData"
          v-model="searchValue"
        />
      </section>
    </section>

    <bk-loading :loading="state.isLoading">
      <bk-table
        v-if="activeType === 'group'"
        :settings="groupSettings"
        class="mt20"
        row-hover="auto"
        remote-pagination
        :pagination="state.pagination"
        :columns="groupColumns"
        :data="state.datas"
        show-overflow-tooltip
        :is-row-select-enable="isRowSelectEnable"
        @selection-change="(selections: any) => handleSelectionChange(selections, isCurRowSelectEnable)"
        @page-limit-change="state.handlePageSizeChange"
        @page-value-change="state.handlePageChange"
        @column-sort="state.handleSort"
      />

      <bk-table
        v-if="activeType === 'gcp'"
        :settings="gcpSettings"
        class="mt20"
        row-hover="auto"
        remote-pagination
        :pagination="state.pagination"
        :columns="gcpColumns"
        :data="state.datas"
        show-overflow-tooltip
        :is-row-select-enable="isRowSelectEnable"
        @selection-change="(selections: any) => handleSelectionChange(selections, isCurRowSelectEnable)"
        @page-limit-change="state.handlePageSizeChange"
        @page-value-change="state.handlePageChange"
        @column-sort="state.handleSort"
      />
    </bk-loading>
  </div>
</template>

<style lang="scss" scoped>
.w100 {
  width: 100px;
}
.w60 {
  width: 60px;
}
.mt20 {
  margin-top: 20px;
}
.search-filter {
  width: 500px;
}
.search-selector-container {
  margin-left: auto;
}
.ml10 {
  margin-left: 10px;
}
</style>

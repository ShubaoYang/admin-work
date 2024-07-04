<template>
  <div class="main-container">
    <TableBody>
      <template #header>
        <TableHeader :show-filter="false">
          <template #top-right>
            <AddButton @add="onAddItem" />
          </template>
        </TableHeader>
      </template>
      <template #default>
        <n-data-table
          :loading="tableLoading"
          :data="dataList"
          :columns="tableColumns"
          :row-key="rowKey"
        />
      </template>
    </TableBody>
    <ModalDialog ref="modalDialogRef" @confirm="onDataFormConfirm">
      <template #content>
        <DataForm ref="dataFormRef" :options="formItems" />
      </template>
    </ModalDialog>
    <ModalDialog ref="menuModalDialogRef" title="菜单权限" contentHeight="40vh">
      <template #content>
        <n-tree
          :data="menuData"
          checkable
          block-line
          cascade
          :default-expanded-keys="defaultExpandedKeys"
          :default-checked-keys="defaultCheckedKeys"
        />
      </template>
    </ModalDialog>
  </div>
</template>

<script lang="ts">
  import { post } from '@/api/http'
  import { get } from '@/api/http'
  import { getDatasourcePage,getDatasourceById,saveDatasourceById } from '@/api/url'
  import {
    TableActionModel,
    useRenderAction,
    useRowKey,
    useTable,
    usePagination,
    useTableColumn,
  } from '@/hooks/table'
  import { DataFormType, ModalDialogType, FormItem } from '@/types/components'
import { number } from 'echarts'
  import { DataTableColumn, NInput, TreeOption, useDialog, useMessage } from 'naive-ui'
  import { defineComponent, h, nextTick, onMounted, ref, shallowReactive } from 'vue'
  interface DataSourceModeType {
    id: number
    name: string
    host: string
    port: number
    databaseName: string
    username: string
    password: string
  }
  const formItems = [
    {
      label: 'ID',
      type: 'number',
      key: 'id',
      value: ref(null),
      disabled: true,
      ss: 123,
      validator: (formItem, message) => {
        if (!formItem.value.value) {
          message.error('请输入ID')
          return false
        }
        return true
      },
      render: (formItem) => {
        return h(NInput, {
          value: formItem.value.value,
          onUpdateValue: (val) => {
            formItem.value.value = val
          },
          placeholder: '请输入ID',
        })
      },
    },
    {
      label: '数据源名称',
      type: 'input',
      key: 'name',
      value: ref(null),
      validator: (formItem, message) => {
        if (!formItem.value.value) {
          message.error('请输入数据源名称')
          return false
        }
        return true
      },
      render: (formItem) => {
        return h(NInput, {
          value: formItem.value.value,
          onUpdateValue: (val) => {
            formItem.value.value = val
          },
          placeholder: '请输入数据源名称',
        })
      },
    },
    {
      label: '地址',
      key: 'host',
      value: ref(null),
      maxLength: 100,
      validator: (formItem, message) => {
        if (!formItem.value.value) {
          message.error('请输入地址')
          return false
        }
        return true
      },
      render: (formItem) => {
        return h(
          NInput,
          {
            value: formItem.value.value,
            onUpdateValue: (val) => {
              formItem.value.value = val
            },
            placeholder: '请输入地址',
          },
        )
      },
    },
    {
      label: '端口',
      key: 'port',
      value: ref(null),
      type: 'number',
      validator: (formItem, message) => {
        if (!formItem.value.value) {
          message.error('请输入端口')
          return false
        }
        return true
      },
      render: (formItem) => {
        return h(NInput, {
          value: formItem.value.value,
          onUpdateValue: (val) => {
            formItem.value.value = val
          },
          placeholder: '请输入端口',
        })
      },
    },
    {
      label: '数据库类型',
      key: 'databaseName',
      value: ref(null),
      maxLength: 50,
      inputType: 'text',
      validator: (formItem, message) => {
        if (!formItem.value.value) {
          message.error('请输入数据库类型')
          return false
        }
        return true
      },
      render: (formItem) => {
        return h(NInput, {
          value: formItem.value.value,
          onUpdateValue: (val) => {
            formItem.value.value = val
          },
          placeholder: '请输入数据库类型',
        })
      },
    },
    {
      label: '用户名',
      key: 'username',
      value: ref(null),
      maxLength: 50,
      inputType: 'text',
      validator: (formItem, message) => {
        if (!formItem.value.value) {
          message.error('请输入用户名')
          return false
        }
        return true
      },
      render: (formItem) => {
        return h(NInput, {
          value: formItem.value.value,
          onUpdateValue: (val) => {
            formItem.value.value = val
          },
          placeholder: '请输入用户名',
        })
      },
    },
    {
      label: '密码',
      key: 'password',
      value: ref(null),
      maxLength: 50,
      inputType: 'text',
      validator: (formItem, message) => {
        if (!formItem.value.value) {
          message.error('请输入密码')
          return false
        }
        return true
      },
      render: (formItem) => {
        return h(NInput, {
          value: formItem.value.value,
          onUpdateValue: (val) => {
            formItem.value.value = val
          },
          placeholder: '请输入密码',
        })
      },
    },
  ] as FormItem[]
  function handleMenuData(
    menuData: Array<any>,
    defaultCheckedKeys: Array<string>,
    defaultExpandedKeys: Array<string>
  ) {
    const tempMenus = [] as Array<TreeOption>
    menuData.forEach((it) => {
      const tempMenu = {} as TreeOption
      tempMenu.key = it.menuUrl
      tempMenu.label = it.menuName
      defaultCheckedKeys.push(tempMenu.key as string)
      if (it.children) {
        defaultExpandedKeys.push(tempMenu.key as string)
        tempMenu.children = handleMenuData(it.children, defaultCheckedKeys, defaultExpandedKeys)
      }
      tempMenus.push(tempMenu)
    })
    return tempMenus
  }
  export default defineComponent({
    name: 'Datasource',
    setup() {
      const modalDialogRef = ref<ModalDialogType | null>(null)
      const dataFormRef = ref<DataFormType | null>(null)
      const menuModalDialogRef = ref<ModalDialogType | null>(null)
      const table = useTable<DataSourceModeType>()
      const rowKey = useRowKey('id')
      const naiveDialog = useDialog()
      const message = useMessage()
      const pagination = usePagination(doRefresh)
      const menuData = shallowReactive([] as Array<TreeOption>)
      const tableColumns = useTableColumn(
        [
          // table.selectionColumn,
          // table.indexColumn,
          {
            title: 'id',
            key: 'id',
          },
          {
            title: '名称',
            key: 'name',
          },
          {
            title: '地址',
            key: 'host',
          },
          {
            title: '端口',
            key: 'port',
          },
          {
            title: '数据库类型',
            key: 'databaseName',
          },
          {
            title: '账号',
            key: 'username',
          },
          {
            title: '密码',
            key: 'password',
          },
          {
            title: '操作',
            key: 'actions',
            render: (rowData) => {
              return useRenderAction([
                {
                  label: '编辑',
                  onClick: onUpdateItem.bind(null, rowData),
                },
                // {
                //   label: '删除',
                //   type: 'error',
                //   onClick: onDeleteItem.bind(null, rowData),
                // },
              ] as TableActionModel[])
            },
          },
        ],
        {
          align: 'center',
        } as DataTableColumn
      )
      const defaultCheckedKeys = shallowReactive([] as Array<string>)
      const defaultExpandedKeys = shallowReactive([] as Array<string>)
      function doRefresh() {
        get<Array<DataSourceModeType>>({
          url: getDatasourcePage,
          data: () => {
            return {
              current: pagination.page,
              size: pagination.pageSize,
            }
          },
        })
          .then((res) => {
            // console.log(res)
            table.handleSuccess(res)
            pagination.setTotalSize((res as any).data.total)
          })
          .catch(console.log)
      }
      function onAddItem() {
        modalDialogRef.value?.toggle()
        nextTick(() => {
          dataFormRef.value?.reset()
        })
      }
      function onUpdateItem(item: any) {
        modalDialogRef.value?.toggle()
        nextTick(() => {
          formItems.forEach((it) => {
            const key = it.key
            const propName = item[key]
            if (propName) {
                it.value.value = propName
            }
          })
        })
      }
      function onDeleteItem(data: any) {
        naiveDialog.warning({
          title: '提示',
          content: '是否要删除此菜单？',
          positiveText: '删除',
          onPositiveClick: () => {
            message.success('模拟菜单删除成功，参数为' + JSON.stringify(data))
          },
        })
      }
      function onDataFormConfirm() {
        if (dataFormRef.value?.validator()) {
          modalDialogRef.value?.toggle()
          console.log(dataFormRef.value.generatorParams())
          post<Array<DataSourceModeType>>({
            url: saveDatasourceById,
            data: () => {
              return {
                id : dataFormRef.value.generatorParams().id,
                name: dataFormRef.value.generatorParams().name,
                host: dataFormRef.value.generatorParams().host,
                port: dataFormRef.value.generatorParams().port,
                databaseName: dataFormRef.value.generatorParams().databaseName,
                username: dataFormRef.value.generatorParams().username,
                password: dataFormRef.value.generatorParams().password,
              }
            },
          })
          .then((res) => {
            naiveDialog.success({
              title: '提示',
              positiveText: '确定',
              content:
                '模拟菜单添加成功，参数为：' + JSON.stringify(dataFormRef.value.generatorParams()),
            })
            doRefresh()
          })
          .catch(console.log)
        }
      }
      function onShowMenu(item: any) {
        get({
          url: getDatasourceById,
          data: {
            id: item.id,
          },
        })
          .then((res) => {
            menuData.length = 0
            menuData.push(...handleMenuData(res.data, defaultCheckedKeys, defaultExpandedKeys))
            menuModalDialogRef.value?.toggle()
          })
          .catch(console.log)
      }
      onMounted(doRefresh)
      return {
        modalDialogRef,
        menuModalDialogRef,
        dataFormRef,
        rowKey,
        menuData,
        tableColumns,
        formItems,
        defaultCheckedKeys,
        defaultExpandedKeys,
        ...table,
        onAddItem,
        onDataFormConfirm,
      }
    },
  })
</script>

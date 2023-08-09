<template>
  <BasicDrawer
    v-bind="$attrs"
    width="70%"
    showFooter
    @register="registerDrawer"
    :title="getTitle"
    @ok="handleSubmit"
  >
    <a-form
      ref="formRef"
      name="dynamic_form_nest_item"
      :model="dynamicValidateForm"
      @finish="onFinish"
    >
      <a-space
        v-for="(sight, index) in dynamicValidateForm.sights"
        :key="sight.id"
        style="display: flex; margin-bottom: 8px"
        align="baseline"
      >
        <a-form-item
          name="area"
          label="Area"
          :rules="[{ required: true, message: 'Missing area' }]"
        >
          <a-select v-model:value="sight.area" :options="areas" />
        </a-form-item>
        <a-space
          v-if="sight.area === 'Beijing'"
          style="display: flex; margin-bottom: 8px"
          align="baseline"
        >
          <a-form-item
            :name="['sights', index, 'value']"
            label="Sight"
            :rules="{
              required: true,
              message: 'Missing sight',
            }"
          >
            <a-select
              v-model:value="sight.value"
              :disabled="!dynamicValidateForm.area"
              :options="(sights[dynamicValidateForm.area] || []).map((a) => ({ value: a }))"
              style="width: 130px"
            ></a-select>
          </a-form-item>
          <a-form-item
            label="Price"
            :name="['sights', index, 'price']"
            :rules="{
              required: true,
              message: 'Missing price',
            }"
          >
            <a-input v-model:value="sight.price" />
          </a-form-item>
        </a-space>
        <a-space
          v-else-if="sight.area === 'HangZhou'"
          style="display: flex; margin-bottom: 8px"
          align="baseline"
        >
          <a-form-item
            :name="['sights', index, 'value']"
            label="Sight"
            :rules="{
              required: true,
              message: 'Missing sight',
            }"
          >
            <a-select
              v-model:value="sight.value"
              :disabled="!dynamicValidateForm.area"
              :options="(sights[dynamicValidateForm.area] || []).map((a) => ({ value: a }))"
              style="width: 130px"
            ></a-select>
          </a-form-item>
        </a-space>
        <a-space v-else style="display: flex; margin-bottom: 8px" align="baseline">
          <a-form-item
            label="Price"
            :name="['sights', index, 'price']"
            :rules="{
              required: true,
              message: 'Missing price',
            }"
          >
            <a-input v-model:value="sight.price" />
          </a-form-item>
        </a-space>
        <MinusCircleOutlined v-if="index !== 0" @click="removeSight(sight)" />
        <PlusOutlined @click="addSight" />
      </a-space>
      <a-form-item>
        <a-button type="primary" html-type="submit">Submit</a-button>
      </a-form-item>
    </a-form>
  </BasicDrawer>
</template>
<script lang="ts">
  import { defineComponent, ref, reactive, watch, computed, unref } from 'vue';
  import { BasicDrawer, useDrawerInner } from '/@/components/Drawer';
  import { BasicForm, useForm } from '/@/components/Form/index';
  import { formSchema } from './dept.data';
  import { createOrUpdate, getDeptList } from './dept.api';
  import { useI18n } from '/@/hooks/web/useI18n';

  import { MinusCircleOutlined, PlusOutlined } from '@ant-design/icons-vue';
  import type { FormInstance } from 'ant-design-vue';

  interface Sights {
    area: string;
    value: string;
    price: string;
    id: number;
  }

  export default defineComponent({
    name: 'DeptDrawer',
    components: { MinusCircleOutlined, PlusOutlined, BasicDrawer, BasicForm },
    emits: ['success', 'register'],
    setup(_, { emit }) {
      const areas = [
        { label: 'Beijing', value: 'Beijing' },
        { label: 'Shanghai', value: 'Shanghai' },
        { label: 'HangZhou', value: 'HangZhou' },
      ];

      const sights = {
        Beijing: ['Tiananmen', 'Great Wall'],
        Shanghai: ['Oriental Pearl', 'The Bund'],
        HangZhou: [' HangZhou', 'xihu'],
      };

      const formRef = ref<FormInstance>();
      const dynamicValidateForm = reactive<{ sights: Sights[]; area: string }>({
        sights: [
          {
            area: 'Beijing',
            value: undefined,
            price: undefined,
            id: Date.now(),
          },
        ],
      });
      watch(
        () => dynamicValidateForm.area,
        () => {
          dynamicValidateForm.sights = [];
        },
      );
      const removeSight = (item: Sights) => {
        let index = dynamicValidateForm.sights.indexOf(item);
        if (index !== -1) {
          dynamicValidateForm.sights.splice(index, 1);
        }
      };
      const addSight = () => {
        dynamicValidateForm.sights.push({
          area: 'Beijing',
          value: undefined,
          price: undefined,
          id: Date.now(),
        });
      };
      const onFinish = (values) => {
        console.log('Received values of form:', values);
        console.log('dynamicValidateForm:', dynamicValidateForm);
      };

      const { t } = useI18n();

      const isUpdate = ref(true);

      const [registerForm, { resetFields, setFieldsValue, updateSchema, validate }] = useForm({
        labelWidth: 100,
        schemas: formSchema,
        showActionButtonGroup: false,
        baseColProps: { lg: 12, md: 24 },
      });

      const [registerDrawer, { setDrawerProps, closeDrawer }] = useDrawerInner(async (data) => {
        await resetFields();
        setDrawerProps({ confirmLoading: false });
        isUpdate.value = !!data?.isUpdate;

        if (unref(isUpdate)) {
          await setFieldsValue({
            ...data.record,
          });
        }
        const treeData = await getDeptList({});
        await updateSchema({
          field: 'parent_id',
          componentProps: { treeData },
        });
      });

      const getTitle = computed(() =>
        !unref(isUpdate) ? t('common.addText') : t('common.updateText'),
      );

      async function handleSubmit() {
        try {
          const values = await validate();
          setDrawerProps({ confirmLoading: true });
          await createOrUpdate(values, unref(isUpdate));
          closeDrawer();
          emit('success');
        } finally {
          setDrawerProps({ confirmLoading: false });
        }
      }
      return {
        formRef,
        dynamicValidateForm,
        onFinish,
        removeSight,
        addSight,
        areas,
        sights,
        registerDrawer,
        registerForm,
        getTitle,
        handleSubmit,
      };
    },
  });
</script>

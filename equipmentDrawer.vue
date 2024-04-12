<template>
  <BasicDrawer
    v-bind="$attrs"
    @register="registerDrawer"
    :title="getTitle"
    :width="700"
    @ok="handleSubmit"
    :showFooter="showFooter"
    destroyOnClose
  >
    <a-form
      ref="formRef"
      :model="formModal"
      :label-col="{ span: 4 }"
      :wrapper-col="{ span: 20 }"
    >
      <a-form-item
        label="通信方式"
        name="msgChannel"
        :rules="[{ required: true, message: '请选择通信方式'}]"
      >
        <a-radio-group v-model:value="formModal.msgChannel" @change="onChange">
          <a-radio :value="1">短报文</a-radio>
          <a-radio :value="2">移动通信</a-radio>
        </a-radio-group>
      </a-form-item>
      <a-form-item
        v-if="ifShow"
        label="通播设备"
        name="receiverList"
        :rules="[{ required: true, message: '请选择通播设备' }]"
      >
        <a-tree-select
          v-model:value="formModal.receiverList"
          style="width: 100%"
          tree-checkable
          :show-checked-strategy="SHOW_CHILD"
          :dropdown-style="{ maxHeight: '400px', overflow: 'auto' }"
          :tree-data="treeData"
          :fieldNames="{
            children:'children', label:'name', value: 'id'
          }"
          placeholder="请选择通播设备"
        />
      </a-form-item>
      <a-form-item
        label="消息类型"
        name="msgType"
        :rules="[{ required: true, message: '请选择消息类型'}]"
      >
        <a-radio-group v-model:value="formModal.msgType" @change="onChangeType">
          <a-radio :value="1">文字</a-radio>
          <a-radio :value="2">图片</a-radio>
          <a-radio :value="3">语音</a-radio>
        </a-radio-group>
      </a-form-item>
      <a-form-item
        label="消息内容"
        name="msgContent"
        :rules="[{ required: true, message: '请输入消息内容' }]"
      >
        <div v-show="formModal.msgType===1">
          <a-textarea :rows="4" v-model:value="formModal.msgContent" placeholder="请输入消息内容"/>
        </div>
        <div v-show="formModal.msgType===2">
          <a-upload
            name="file"
            list-type="picture-card"
            class="avatar-uploader"
            :showUploadList="false"
            accept=".jpg,.jpeg,.gif,.png,.webp"
            :action="uploadUrl"
            :headers="getheader()"
            :before-upload="beforeUpload"
            @change="handleChange"
          >
            <img v-if="imageUrl" :src="imageUrl" alt="avatar"/>
            <div v-else>
              <loading-outlined v-if="loading"></loading-outlined>
              <plus-outlined v-else></plus-outlined>
            </div>
          </a-upload>
        </div>
        <div v-show="formModal.msgType===3">
          <a-textarea v-model:value="formModal.msgContent" :rows="4" placeholder="请输入消息内容"/>
          <a-button @click="start">开始识别</a-button>
          <a-button @click="finish">结束识别</a-button>
        </div>
      </a-form-item>
    </a-form>
  </BasicDrawer>
</template>
<script lang="ts" setup>
import {ref, reactive} from 'vue';
import {BasicDrawer, useDrawerInner} from '/@/components/Drawer';
import {useDrawerAdaptiveWidth} from '/@/hooks/jeecg/useAdaptiveWidth';
import {addSendMsg, terminalList} from './equipmentApi'
import {cloneDeep} from 'lodash-es';
import {PlusOutlined, LoadingOutlined} from '@ant-design/icons-vue';
import {useGlobSetting} from '/@/hooks/setting';
import {getFileAccessHttpUrl, getHeaders} from '/@/utils/common/compUtils';
import {TreeSelect} from 'ant-design-vue';
import {useMessage} from "@/hooks/web/useMessage";
/**
 * 图片上传
 */
const imageUrl = ref()
const loading = ref(false)
const {domainUrl} = useGlobSetting();
const uploadUrl = domainUrl + '/sys/common/upload';
const getheader = () => {
  return getHeaders();
}

const getBizData = () => {
  return {
    biz: 'jeditor',
    jeditor: '1',
  };
}
const {createMessage: $message} = useMessage();
const beforeUpload = (file) => {
  const isJpgOrPng = file.type === 'image/jpeg' || file.type === 'image/png';
  if (!isJpgOrPng) {
    $message.error('请上传正确的格式');
  }
  return isJpgOrPng;
};

const handleChange=(info: Recordable)=> {
  const file = info.file;
  const status = file?.status;

  if (status === 'uploading') {
    if (!loading.value) {
      loading.value = true;
    }
  } else if (status === 'done') {
    let realUrl = getFileAccessHttpUrl(file.response.message);
    imageUrl.value = realUrl
    formModal.msgContent = realUrl
    loading.value = false;
  } else if (status === 'error') {
    loading.value = false;
  }
}

const onChangeType = ()=>{
  formModal.msgContent = ''
}


/**
 * 语音转文字
 */

import IatRecorder from "@/utils/xunfei/iatRecorder"; // 科大讯飞
let iatRecorder = new IatRecorder();



// 状态改变时处罚
iatRecorder.onWillStatusChange = function (oldStatus, status) {

};

// 监听识别结果的变化
iatRecorder.onTextChange = function (text) {
  formModal.msgContent = text
};

const start = () => {
  iatRecorder.start();
};

const finish=()=>{
  iatRecorder.stop();
}

const getTitle = '通播发送'

const showFooter = ref(true);

const formModal = reactive({
  msgChannel: 1,
  msgContent: "",
  msgType: 1,
  receiverList: []
})

const ifShow = ref(true)
const record = ref()
// 声明Emits
const emit = defineEmits(['success', 'register']);
const [registerDrawer, {setDrawerProps, closeDrawer}] = useDrawerInner(async (data) => {
  ifShow.value = data.ifShow
  record.value = data.record
  getTerminalList()
})

const SHOW_CHILD = TreeSelect.SHOW_CHILD;

const treeData = ref()
//重置表单
const resetForm = () => {
  const newData = {
    msgChannel: 1,
    msgContent: "",
    msgType: "",
    receiverList: []
  }
  Object.assign(formModal, newData);
}

const onChange = () => {
  formModal.receiverList = []
  getTerminalList()
}

const getTerminalList = () => {
  terminalList({msgChannel: formModal.msgChannel}).then((res) => {
    if (res) {
      treeData.value = res
    }
  })
}
const {adaptiveWidth} = useDrawerAdaptiveWidth();
const formRef = ref()

async function handleSubmit() {
  const params = cloneDeep(formModal)
  console.log(ifShow.value)
  if (!ifShow.value) {
    if (record.value.bdCode && record.value.bdCode !== '') {
      params.receiverList.push(record.value.bdCode)
    } else {
      params.receiverList.push(record.value.terminalId)
    }
  }
  try {
    await formRef.value.validate().then(() => {
      addSendMsg(params).then(() => {
        resetForm()
        closeDrawer();
        //刷新列表
        emit('success');
      })

    })
  } catch (error) {
  }
}
</script>

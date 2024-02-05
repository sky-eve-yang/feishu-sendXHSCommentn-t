<template>
  <el-form ref="form" class="form" label-position="top">
    
    <div style="width: 100%;padding-left: 10px;border-left: 5px solid #2598f8;margin-bottom: 20px;padding-top: 5px;">{{ $t('title') }}</div>
    <el-alert style="margin: 20px 0;color: #606266;" type="info" >
      <div class="helper-doc">
        <div style="margin-bottom: 6px;">{{$t('alerts.desc')}}</div>
        <span>{{ $t('helpTip') }}</span>
        <span style="height: 16px;width: 16px;margin-left: 12px;">
          <a :href="t('helpDocLink')" target="_blank"><span>{{ $t('helpDoc') }}</span></a>
        </span>  
      </div>
    </el-alert>
    

    <el-form-item :label="$t('labels.noteLink')" size="large" required>
      <el-select v-model="noteLinkFieldId" :placeholder="$t('placeholder.noteLink')" style="width: 100%">
        <el-option v-for="meta in fieldMetaListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('labels.comment')" size="large" required>
      <el-select v-model="commentContentFieldId" :placeholder="$t('placeholder.comment')" style="width: 100%">
        <el-option v-for="meta in fieldMetaListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('labels.cookie')" size="large" required>
      <el-input v-model="cookie" type="text" :placeholder="$t('placeholder.cookie')"></el-input>
    </el-form-item>
    <el-form-item :label="$t('labels.secret')" size="large" required>
      <el-input v-model="secret" type="password" :placeholder="$t('placeholder.secret')"></el-input>
    </el-form-item>

    
    <el-button v-loading="isWritingData" @click="subbmitAndWriteData" :disabled="!issubmitAbled" color="#3370ff" type="primary" plain size="large">{{ $t('submit') }}</el-button>
  </el-form>
</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import qs from 'qs';


// -- 数据区域
const { t } = useI18n();
const fieldNameListToBeWritten = ref(['releaseTime', 'uploader', 'errorTip'])
const i18nFieldListBasePath = ref('fieldListToWritten')
const i18nCheckFieldTypeBasePath = ref('fieldListToWritten')

const noteLinkFieldId = ref("")
const commentContentFieldId = ref("")
const baseUrl = ref("https://send-xh-scomment-backend-1326906378.replit.app")
const path = ref("send_comment")
const cookie = ref()
const secret = ref()
const fieldMetaListSeView = ref([])


const isWritingData = ref(false)
const issubmitAbled = computed(() => {
  return noteLinkFieldId.value && commentContentFieldId.value.length && cookie.value && secret.value

})  // 是否允许提交，及必选字段是否都填写

// -- 核心算法区域
const subbmitAndWriteData = async () => {
  isWritingData.value = true
  const {table, view, tableId, viewId} = await getBaseTableAndView()
  const recordList = await getRecordListByBase(view, tableId, viewId)

  // 记录缓存
  setParamsStorage()
  
  const fieldIdObjectToWritten = await getfieldIdObjectToWritten(table)

  for (let recordId of recordList) {
    // 读取笔记链接和评论内容
    console.log(4444)

    const noteLinkField = await table.getFieldById(noteLinkFieldId.value)
    const commentContentField = await table.getFieldById(commentContentFieldId.value)
    const noteLink = await getCellValueByRFT(recordId, noteLinkField, 'link')
    const commentContent = await getCellValueByRFT(recordId, commentContentField, 'text')

    // TODO: 报错审查，审查笔记链接和评论内容 - 字段格式 链接内容格式
    console.log(85, recordId, noteLink)
    const errorTipFieldId = fieldIdObjectToWritten['errorTip']
    const errorRes = await handleNoteLinkError(noteLink, errorTipFieldId, recordId, table)
    if (errorRes.isError) continue

    // 请求后端，获取返回数据
    const requestData = await getCommentRequestData(baseUrl.value, path.value, noteLink, commentContent, cookie.value, secret.value)
    if (!requestData.isSuccess) {
      await table.setCellValue(
        fieldIdObjectToWritten['errorTip'], recordId, 
        [{ type: 'text', text: requestData.result.response.data.error }]
      )
      continue
    }
    // 获取字段id和值
    const fieldValueObjectToWritten = getfieldValueObjectToWritten(fieldNameListToBeWritten.value, requestData)
    const fieldIdAndValueObejctToWritten = getFieldIdAndValueObject(fieldIdObjectToWritten, fieldValueObjectToWritten)
    // 写入数据
    await table.setRecord(recordId, {
      fields: fieldIdAndValueObejctToWritten
    })
  }
  isWritingData.value = false

  
  // UI 提示：数据处理完毕
  await bitable.ui.showToast({
    toastType: 'success',
    message: `${t('finishTip')}`
  })

}

const handleNoteLinkError = async (noteLink, errorTipFieldId, recordId, table) => {
  

  if (noteLink.isEmpty) 
    return {"isError": true}
  

  if (noteLink.isError) {
      await table.setCellValue(
        errorTipFieldId, recordId, 
        [{ type: 'text', text: t('errorTip.emptyNoteLink') }]
      )

      return {"isError": true}
    }

  if (!noteLink.includes("https://www.xiaohongshu.com/explore")) {
    console.log(errorTipFieldId, recordId, t('errorTip.errorLink'))
    await table.setCellValue(
      errorTipFieldId, recordId, 
      [{ type: 'text', text: t('errorTip.errorLink') }]
    )
    return {"isError": true}
  }
  return {"isError": false}
}



const setParamsStorage = () => {
  localStorage.setItem('noteLinkFieldId', noteLinkFieldId.value)   // string 类型
  localStorage.setItem('commentContentFieldId', commentContentFieldId.value)   // string 类型
  localStorage.setItem('cookie', cookie.value)   // string 类型
  localStorage.setItem('secret', secret.value)   // string 类型
}

/**
 * 获取当前多维表格的 table 和 view 信息
 * @return {object} table
 * @return {object} view
 * @return {string} tableId
 * @return {string} tableId
 */
const getBaseTableAndView = async () => {
  // 加载bitable实例
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();
  const view = await table.getViewById(viewId);

  return { table, view, tableId, viewId}
}

/**
 * @{query} 获取需要处理的记录Id列表
 * @param {object} view 多维表格视图信息
 * @param {string} tableId 多维表格表格Id
 * @param {string} viewId 多维表格视图Id
 * @return {array} recordList 需要处理的记录Id列表
 */
const getRecordListByBase = async (view, tableId, viewId) => {
  // ## mode1: 全部记录
  // const recordList = await view.getVisibleRecordIdList()

  // ## model2: 交互式选择记录 
  const recordList = await bitable.ui.selectRecordIdList(tableId, viewId);
  return recordList
}

/**
 * @{query} 获取单元格的内容 
 * @param {string} recordId 记录Id
 * @param {object} field 字段信息
 * @param {string} type 要获取的字段内容类型
 */
const getCellValueByRFT = async (recordId, field, type='text') => {
  const cellValue = await field.getValue(recordId)
  console.log(cellValue)
  console.log(type)
  console.log(typeof cellValue)
  if (!cellValue) 
    return {"isEmpty": true}
  

  if (typeof cellValue == 'object') {
    for (let item of cellValue) {
      if (item.type == (type == 'link' ? 'url' : 'text')   ) 
        return item[type]
    }
    // 没有找到合适的值
    return {"isError": true}
  }
    
  return cellValue
}

/**
 * @{query} 发送评论，获取返回数据
 * @param {string} baseUrl 接口请求的基地址
 * @param {string} path 接口请求路由
 * @param {string} noteLink 小红书文章链接
 * @param {string} commentContent 评论内容
 * @param {string} cookie 小红书 Cookie
 * @param {string} secret 授权码，开通插件授权服务
 */
const getCommentRequestData = async (baseUrl, path, noteLink, commentContent, cookie, secret) => {
  var url = `${baseUrl}/${path}`
  let return_res;

  if (path === 'send_comment') { // 进行接口归一化处理
    var data = qs.stringify({
      'url': noteLink,
      'content': commentContent,
      'cookie': cookie,
      'secret': secret
    });

    var config = {
      method: 'post',
      url: url,
      data : data
    };
    await axios(config)
      .then(function (response) {
        const res = response.data
        console.log("getCommentRequestData() >> response.data", res);
        const commentInfo = res.info.data.comment
        return_res = {
          "status": 200,
          "data": {
            "releaseTime": commentInfo.create_time,
            "uploader": commentInfo.user_info.nickname,
            "isSuccess": Boolean(res.info.success) 
            
          }
        }
      })
      .catch(function (error) {
        console.log("请求错误：", error);
        return_res = error
      });
  } 

  if (return_res.status === 200)
    return return_res.data
  else 
    return {"status": "-100", "result": return_res}
}


const getfieldIdObjectToWritten = async (table) => {

  // 匹配已有的字段
  const existedFieldIdObjectToWritten = getExistedFieldIdObjectToWritten(fieldMetaListSeView.value, fieldNameListToBeWritten.value)

  if (typeof existedFieldIdObjectToWritten == 'string') {// 错误处理，提示格式错误 
    await bitable.ui.showToast({
      toastType: 'warning',
      message: mappedFields
    })
    isWritingData.value = false
    return
  }

  // 创建缺少的字段
  await createFields(existedFieldIdObjectToWritten, table)
  
  console.log("completeMappedFieldIdsValue() => existedFieldIdObjectToWritten", existedFieldIdObjectToWritten)
  return existedFieldIdObjectToWritten
}

/**
 * @{query} 获取要写入数据的字段的 name-id 对象，若需创建，则id值设置为-1
 * @param {array} existedFieldMetaList 表格中已经存在的字段元数据列表
 * @param {array} fieldNameListToBeWritten 需要写入数据的字段名称列表
 */
const getExistedFieldIdObjectToWritten = (existedFieldMetaList, fieldNameListToBeWritten) => {
  const existedFieldIdObject = {};
  for (let field of fieldNameListToBeWritten) {
    // 查找与fieldNameListToBeWritten相匹配的existedFieldMetaList项目
    const foundField = existedFieldMetaList.find(f => f.name ===  t(`${i18nFieldListBasePath.value}.${field}`));

    const type = getFieldToCreateType(field)
    const checkTip = `「t${t(`${i18nFieldListBasePath.value}.${field}`)}」${t(`${i18nCheckFieldTypeBasePath}.${type}`)}`

    if (field.endsWith('Count') && foundField && foundField.type !== 2)
      return checkTip
    else if (field == t('selectGroup.videoInfo.uploader') && foundField && foundField.type !== 1)
      return checkTip
    else if (field == t('selectGroup.videoInfo.releaseTime') && foundField && foundField.type !== 5)
      return checkTip
    else if (field == t('selectGroup.videoInfo.lastUpdateTime') && foundField && foundField.type !== 5)
      return checkTip
    

    
    // 如果找到了相应的项目，就使用其id，否则设置为-1
    existedFieldIdObject[field] = foundField ? foundField.id : -1;
  }
    


  return existedFieldIdObject;
} 

/**
 * @{imperative} 给fieldIdObjectToWritten中所有id标记为-1的field创建字段，并返回其id
 * @param {object} fieldIdObjectToWritten 需要写入数据的字段 name-id 集合
 * @param {object} table 表格实例
 */
const createFields = async (fieldIdObjectToWritten, table) => {
  for (let key in fieldIdObjectToWritten) {
    if (fieldIdObjectToWritten[key] == -1) {
      const name = t(`${i18nFieldListBasePath.value}.${key}`)
      const type = getFieldToCreateType(key)

      fieldIdObjectToWritten[key] = await table.addField({
        type: FieldType[type],
        name,
      })
    }
  }
}

/**
 * @{query} 依据name返回field对应的字段类型
 * @param {string} name fieldName
 */
const getFieldToCreateType = (name) => {
  const textFieldList = ['uploader', 'errorTip']
  const dateTimeFieldList = ['releaseTime']

  if (textFieldList.includes(name)) {
    return "Text"
  } else if (dateTimeFieldList.includes(name)) {
    return "DateTime"
  }

  return "Text"
}

/**
 * @{query} 获取要写入数据的字段 name-value 对象
 * @param {array} fieldNameListToBeWritten 需要写入数据的字段名称数组
 * @param {object} requestData 要写入的数据
 */
const getfieldValueObjectToWritten = (fieldNameListToBeWritten, requestData) => {
  let fieldValueObjectToWritten = {}
  for (let name of fieldNameListToBeWritten) {
    fieldValueObjectToWritten[name] = requestData[name] || ""
  }

  return fieldValueObjectToWritten
}

/**
 * @{query} 获得 id-value 字段object
 * @param {object} fieldIdObject name-id 字段object
 * @param {object} fieldValueObject name-value 字段object
 */
const getFieldIdAndValueObject = (fieldIdObject, fieldValueObject) => {
  console.log(320, fieldIdObject)
  console.log(321, fieldValueObject)
  let fieldIdAndValueObject = {}
  for (let name in fieldIdObject) {
    fieldIdAndValueObject[fieldIdObject[name]] = fieldValueObject[name]
  }
  console.log(326, fieldIdAndValueObject)

  return fieldIdAndValueObject

}



onMounted(async () => {
  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  fieldMetaListSeView.value = await view.getFieldMetaList()
  // 获取字段列表 -- end
  
  // 从缓存中获取数据 -- start
  if (localStorage.getItem('noteLinkFieldId') !== null) {
    noteLinkFieldId.value = localStorage.getItem('noteLinkFieldId')
  }
  if (localStorage.getItem('commentContentFieldId') !== null) {
    commentContentFieldId.value = localStorage.getItem('commentContentFieldId')
  }
  if (localStorage.getItem('cookie') !== null) {
    cookie.value = localStorage.getItem('cookie')
  }
  if (localStorage.getItem('secret') !== null) {
    secret.value = localStorage.getItem('secret')
  }
  // 从缓存中获取数据 -- end
});
    
</script>



<style scoped>
.helper-doc {
  
  margin: 10px 0;
  font-size: 14px;
}
.helper-doc a {
  color: #409eff;
}
.helper-doc a:hover {
  color: #7abcff;
}
</style>

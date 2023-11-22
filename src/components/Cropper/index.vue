<!--
 * @Author: nicho-UJN nicholas9698@outlook.com
 * @Date: 2023-09-08 10:43:30
 * @LastEditors: nicho-UJN nicholas9698@outlook.com
 * @LastEditTime: 2023-10-27 16:42:41
 * @FilePath: /mini-mwp/src/pages/index/index.vue
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
<template>
  <view class="index">
    <nut-noticebar
      text="沐沐解题(小学数学题自动解答) 是基于人工智能模型和文心一言开发的小程序应用，旨在帮助学生理解问题求解思路，增强逻辑推理能力。"
      wrapable style="text-align: left;"
    ></nut-noticebar>
    <view>
      <nut-textarea v-model:model-value="problem" placeholder="请输入待求解的问题。" autofocus="true" limit-show max-length="512" style="height: 14em;" />
    </view>
    <view class="btn">
      <nut-button type='info' color="#7232dd" plain style="margin-right: 2em;" @click="takePhoto()">拍照识别</nut-button>
      <nut-button type="info" :loading="loading" style="margin-left: 2em;" @click="handleClick('text', msg, true)">开始解答</nut-button>
    </view>
    <view>
      <nut-textarea readonly v-model:model-value="answer" />
    </view>
    <nut-toast :msg="msg" v-model:visible="show" :type="type" :cover="cover"/>
    <nut-popup round="true" position="bottom" closeable :style="{ height: '15%', paddingTop: '15%'}" v-model:visible="showPop" @click-close-icon="onCloseIcon" @click-overlay="onCloseIcon">网络连接失败！</nut-popup>
    <nut-dialog title="温馨提示" content="亲，授权小程序登录后才能正常使用小程序功能" v-model:visible="visible" @cancel="onCancel" @ok="onOk" />
    <nut-overlay v-model:visible="overshow" :close-on-click-overlay="false">
      <div class="display: flex; height: 100%; align-items: center;justify-content: center;">
        <image-cropper />
      </div>
    </nut-overlay>
    <view>
      <my-tabbar />
    </view>
  </view>
</template>

<script>
import { reactive, toRefs } from 'vue';
import Taro from '@tarojs/taro';
import ImageCropper from '../../components/Cropper';
import MyTabbar from '../../components/Tabbar';

export default {
  name: 'Index',
  components: {
    MyTabbar,
    ImageCropper
  },
  setup() {
    Taro.eventCenter.on('cancelOcr', (data) => {
      state.overshow = false;
    });
    Taro.eventCenter.on('ocrResult', (data) => {
      state.problem = data;
    })
    const state = reactive({
      openid: '',
      msg: '问题提交成功，请耐心等待～',
      loading: false,
      type: 'text',
      problem: '',
      answer: '',
      show: false,
      showPop: false,
      cover: false,
      overshow: false,
      visible: true
    });
    try {
      var hasUserInfo = Taro.getStorageSync('hasUserInfo');
      if (hasUserInfo) {
        state.visible = false;
        state.openid = Taro.getStorageSync('openid');
      }
    } catch (e) {
      Taro.setStorageSync('hasUserInfo', false);
      state.visible = true;
    };
    const onOk = () => {
      Taro.login({
        success: function(res) {
          if (res.code) {
            Taro.request({
              method: 'POST',
              url: 'https://api.evosimy.top/minilogin',
              header: {
                'content-type': 'application/json'
              },
              data: {
                code: res.code
              },
              success: function (res) {
                Taro.setStorageSync('hasUserInfo', true);
                Taro.setStorageSync('openid', res.data['openid']);
                state.openid = res.data['openid'];
              },
              fail: function (res) {
                state.showPop = true;
              }
            })
          } else {
            // console.log('登录失败！' + res.errMsg)
            state.showPop = true;
          }
        }
      })
    };
    const onCloseIcon = () => {
      Taro.exitMiniProgram();
    }
    const onCancel = () => {
      Taro.exitMiniProgram();
    };

    const takePhoto = () => {
      Taro.navigateTo({
        url: '/pages/camera/camera',
        success: function () {
          Taro.eventCenter.on('acceptDataFromOpenedPage', (data) => {
            state.overshow = true;
          });

        }
      })
    };

    const handleClick = (type, msg, cover = false) => {
      state.show = true;
      state.loading = true;
      state.msg = msg;
      state.type = type;
      state.cover = cover;
      Taro.request({
        url: 'https://api.evosimy.top/reason',
        timeout: 60000,
        method: "POST",
        data: {
          problem: state.problem,
          openid: state.openid
        },
        header: {
          'content-type': 'application/json'
        },
        success: function (res) {
          var answer = '等式：' + res.data['expression'] + '\n\n计算结果：' + res.data['values'] + '\n\n问题解释：\n' + res.data['explanation'];
          state.answer = answer.replace(/(\n|\r|\r\n|↵)/g, '<br/>');
          state.loading = false;
          Taro.request({
            url: 'https://api.evosimy.top/getrecord',
            method: 'POST',
            data: {
              openid: state.openid
            },
            header: {
              'content-type': 'application/json'
            },
            success: function (res) {
              Taro.setStorageSync('dataRecord', res.data['data']);
              Taro.eventCenter.trigger('dataRecordChanged');
            }
          })
        },
        fail: function (res) {
          state.loading = false;
          console.log('failed')
        },
      })
    };

    return {
      ...toRefs(state),
      onOk,
      onCloseIcon,
      onCancel,
      handleClick,
      takePhoto
    }
  }
}
</script>

<style lang="scss">
.index {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
}
</style>

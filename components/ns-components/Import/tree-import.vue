<!--数据导入弹窗组件-->
<template>
  <ns-dialog class="tree-import" :title="title" :visible.sync="importVisible" :before-close="beforeClose">
    <ns-steps id="step" :space="200" :active="active">
      <ns-step title="上传文件"></ns-step>
      <ns-step title="导入数据"></ns-step>
      <ns-step title="导入完成"></ns-step>
    </ns-steps>

    <div class="firstStep" v-if="active === 1">
      <p>1、请按照数据模板的格式导入数据，点击
        <el-dropdown @command="handleCommand">
          <span class="el-dropdown-link">下载数据模板</span>
          <el-dropdown-menu slot="dropdown">
            <el-dropdown-item command="base">
              基础数据样表-标准版
            </el-dropdown-item>
            <el-dropdown-item command="senior">
              基础数据样表-高级版
            </el-dropdown-item>
          </el-dropdown-menu>
        </el-dropdown>
      </p>

      <p>2、请选择数据重复时的操作方式</p>
      <el-radio v-model="radio" label="1">覆盖导入</el-radio>
      <el-radio v-model="radio" label="0">不导入</el-radio>

      <p>3、选择模板文件并开始导入</p>
      <!--:on-change="fileType"-->
      <el-upload class="upload-demo"
                 ref="upload"
                 :action="'http://192.168.1.31/api/owner/owner-rest/import-house?guid='+guidRandom"
                 :on-change="fileType"
                 accept=".xls,.xlsx"
                 :file-list="fileList"
                 :headers="requestHeader"
                 :data="dataInfo"
                 :with-credentials="true"
                 :auto-upload="false">
        <el-button slot="trigger" size="small" type="primary">选取文件</el-button>

        <div slot="tip" class="el-upload__tip">只能上传xls、xlsx文件！</div>
      </el-upload>
    </div>

    <div class="secondStep" v-else-if="active === 2">
      <ns-tree-progress :now='now' :all='all'></ns-tree-progress>
      <P>数据正在导入中，已导入{{now}}条</P>
      <span>提示：数据导入完成之后将自动发送结果通知您</span>
    </div>

    <div class="thirdStep" v-else="active === 3">
      <ns-icon-svg class="failed" icon-class="CombinedShapeCopy"></ns-icon-svg>
      <p>导入失败</p>
      <ns-icon-svg class="success" icon-class="check-circle"></ns-icon-svg>
      <p>导入完成</p>
      <P>已成功导入100条</P>
      <ns-button type="primary" @click="back">继续导入</ns-button>
      <ns-button @click="cancel">关闭</ns-button>

    </div>

    <span slot="footer" class="dialog-footer">
			<el-button size="medium" type="primary" @click="submitUpload"
                 v-if="active === 1 ||  active === 2">开始导入</el-button>
            <el-button size="medium" @click="cancel">取 消</el-button>
        </span>
  </ns-dialog>
</template>

<script>
  import {importFetch} from '@/api/import';
  import ns from '@/utils/nsQuery/nsQuery';
  import * as store from '@/utils/nsQuery/nsStore'
  import {modelList} from '../../../api/summary'//下载请求地址
  import {download} from '../../../api/download/download'//下载请求地址
  import {mapGetters} from 'vuex'
  import EventHub from "@/utils/eventHub/eventHub.js";
  export default {
    props: {
      importVisible: {
        type: Object,
        default: function () {
          return {
            visible: false
          }
        }
      },
      title: {
        type: String,
        default: ''
      },
      active: {
        type: Number,
        default: 1
      }
    },
    computed: {
      ...mapGetters([
        'websocketUrl'
      ]),
    },
    data() {
      return {
        radio: '1',
        guidRandom:'',
        fileList: [],//上传的文件列表
        allowFileType: ['xls', 'xlsx'],//允许上传的文件格式类型
        requestHeader: store.requestHead.get(),
        time: new Date(),//本地时间
        innerVisible: false,
        websocket: null,
        dataInfo: {
          updateFlag: 1
        }
      }
    },
    created() {
      this.modelList();
      EventHub.$on('createWebsocket',this.createConWebSocket)
    },
    methods: {
      createConWebSocket: function (guid) {
          this.guidRandom = guid
        try {
          if ('WebSocket' in window) {
            this.websocket = new WebSocket('ws://' + this.$store.state.Store.core.websocketUrl + '/ws/websocket/superAdmin'+guid);
          } else {
            alert('Not support websocket!')
            return
          }
          //发生错误时的回调函数
          this.websocket.onerror = this.websocketOnerror
          //链接成功时的回调函数
          this.websocket.onopen = this.connectSuccess
          //接受消息时的回调函数
          this.websocket.onmessage = this.websocketOnmessage
        } catch (err) {
          console.log('connect faile in createConSocket:' + err)
        }

      },
      connectSuccess() {
        console.log(this.userId + ': connect success!')
      },
      closeWebsocket() {
        this.websocket.close()
      },
      //取消
      cancel() {
        this.closeWebsocket();
        this.websocket = null;
        this.now = this.all = 0;
        this.$set(this.importVisible, 'visible', false);
        this.active = 1;
        this.fileList = []

      },
      websocketOnmessage(message) {
        this.now = parseInt(message.data.split('S')[0])
        this.all = parseInt(message.data.split('S')[1])
      },
      websocketOnerror() {
        console.log('websocket error')
      },
      //继续导入
      back() {
        this.active = 1;
      },
      //关闭x按钮
      beforeClose() {
        this.$set(this.importVisible, 'visible', false);
        this.active = 1;
      },
      //提交
      submitUpload() {
        if (this.fileType == "xlsx" || this.fileType == "xls") {
          if (this.active++ > 2) this.active = 1;
          this.$refs.upload.submit();
        } else {
          this.$message({showClose: true, message: '请选择文件', type: 'error',});//msg
        }

      },
      //下载模板类型获取
      modelList() {
        modelList({
          "content-type": "application/json;charset=UTF-8",
          "date": this.time,
          "transfer-encoding": "chunked",
          "x-application-context": "fastdfs-server:8779"
        }).then(result => {
          console.log(result);
          this.flieBaseId = result.resultData[0].fileId;
          this.flieSeniorId = result.resultData[1].fileId;
        })
      },
      //文件类型判断
      fileType(file, fileList) {
        const extension = file.name.split('.');
        const fileType = extension[extension.length - 1];
        this.fileType = fileType;
        //符合类型，验证通过
        if (ns.base.arrContainObj(this.allowFileType, this.fileType)) {
          return true;
        }
        else {
          this.$message({showClose: true, message: '请选择正确的文件格式', type: 'error',});//msg
          fileList = fileList.splice(fileList.length - 1, 1);//删除最后一个添加的文件对象（错误文件）
        }
      },

      //显示下拉
      downLoadShow() {
        this.innerVisible = true
      },

      //下载
      handleCommand(command) {
        if (command === 'base') {
          download({
            "fileId": this.flieBaseId
          })
        } else if (command === 'senior') {
          download({
            "fileId": this.flieSeniorId
          })
        }
      },
      changeCallback(file, fileList) {
        this.fileList = fileList.slice(-3);
      },

    }
  }
</script>

<style rel="stylesheet/less" lang="scss">
  .tree-import {
    #step {
      margin-left: 80px;
    }

    .firstStep {
      margin-left: 10%;
      margin-top: 5%;
      p {
        margin-top: 10px;
      }
      .el-radio {
        margin-left: 5%;
      }
      .upload-demo {
        margin-left: 5%;
      }
      .el-dropdown-link {
        color: #0a7af8;
      }
    }

    .secondStep {
      text-align: center;
      p {
        margin-top: 10px;
      }
      margin-top: 5%;
      span {
        color: #D6D6D6;
      }
    }

    .thirdStep {
      text-align: center;
      margin-top: 5%;
      .success {
        color: #16CA38;
        font-size: 50px;
      }
      .failed {
        color: #FF0000;
        font-size: 50px;
      }
    }
    .el-step__icon {
      border-radius: 50%;
      border: 1px solid #d4d4d4;
      background: #d4d4d4;
      color: #ffffff;

    }
    .el-step__head.is-finish .el-step__icon {
      background: #0a7af8;
      border: 1px solid #0a7af8;
    }

    //==============================================
    .el-steps {
      background: white;
    }
    .is-horizontal .el-step {
      position: relative;
    }
    .is-horizontal .el-step__title {
      display: inline-block;
      height: 26px;
      line-height: 26px;
    }
    .el-step__title {

      font-weight: normal;
    }
    .is-horizontal .el-step__main {
      display: inline-block;
      position: absolute;
      top: 0px;
      padding-left: 18px;
      left: 20px;
      background: white;
    }
    .el-step__line.is-vertical {
      top: 35px;
      bottom: 7px;
    }
    .el-step__title.is-wait {
      color: #666666;
    }
    .el-step__title.is-process {
      color: #666666;
      font-weight: 400;

    }
    .treeProgress span {
      position: absolute;
      right: -10px;
    }
    .el-step__icon {
      line-height: 30px;
    }
    .el-step__title {
      width: 64px;
    }
  }
</style>

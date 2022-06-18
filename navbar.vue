<template>
  <div>
    <div class="navbar">
      <div class="palette">
        <i class="cpr" :class="icon ? 'el-icon-s-unfold' : 'el-icon-s-fold'" @click="changIcon"></i>
        <!-- <i class="el-icon-refresh-right" style="cursor:pointer;" @click="Refresh"></i> -->
      </div>
      <!-- <el-breadcrumb class="breadcrumb" separator-class="el-icon-arrow-right">
        <el-breadcrumb-item style="font-size:18px;">当前</el-breadcrumb-item>
        <el-breadcrumb-item v-for="(item, index) in levelList" :key="index">{{item}}</el-breadcrumb-item>
      </el-breadcrumb> -->
      <div class="Account">
        <!-- <span>{{ currentTime }}</span> -->

        <template v-if="cs">
          <el-dropdown @command="handleCs" trigger="click" style="margin-right:15px">
            <span class="el-dropdown-link">
              <el-badge :hidden="!unread || !duty" is-dot class="item">
                <i class="iconfont icon-xinxi cpr" :class="{info: duty === 0, success: duty === 1, 'danger offline': !online, 'new-msg': unread}"></i>
              </el-badge>
            </span>
            <el-dropdown-menu>
              <el-dropdown-item command="message">
                <el-badge :hidden="!unread || !duty" :max="99" :value="unread" class="item" style="font-size:12px;padding:0 10px">{{$t('cs.msg')}}</el-badge>
              </el-dropdown-item>
              <el-dropdown-item command="duty">
                <el-switch v-model="duty" active-text="在席" :active-value="1" :inactive-value="0" @change="switchDuty" style="font-size:12px;padding:0 10px"></el-switch>
              </el-dropdown-item>
            </el-dropdown-menu>
          </el-dropdown>
        </template>

        <ScreenFull></ScreenFull>
        <!-- <el-button size="mini" type="danger">退出</el-button> -->
        <el-dropdown @command="handleCommand" trigger="click">
          <span class="el-dropdown-link">{{$store.getters['permission/admin'].admin.nickname}}
            <i class="el-icon-arrow-down el-icon--right"></i>
          </span>
          <el-dropdown-menu slot="dropdown">
            <!-- <el-dropdown-item command="1" style="font-size:12px;padding:0 10px">修改资料</el-dropdown-item> -->
            <el-dropdown-item command="2" style="font-size:12px">修改密码</el-dropdown-item>
            <el-dropdown-item command="3" style="font-size:12px" @click="getOut">退出</el-dropdown-item>
          </el-dropdown-menu>
        </el-dropdown>
      </div>
    </div>
    <!-- <el-tabs v-model="editableTabsValue" @tab-click="handleClick(editableTabsValue)">
        <el-tab-pane v-for="(item) in editableTabs" :key="item.name" :label="item.title" :name="item.name">
          <span slot="label"> {{item.title}}<i v-show="item.name != 'layout'" class="el-icon-close" style="margin-left: 10px;" @click.stop="removeTab(item.name)"></i></span>
        </el-tab-pane>
      </el-tabs> -->
    <tags-view v-show="!$store.state.reactive.isMobile" />
  </div>
</template>
<script>
import ScreenFull from './modules/ScreenFull'
import { removeCookie } from '@/utils/cookies'
import { apiMessage } from '@/request/api'
import NIM from '@/assets/js/NIM_Web_NIM_v8.7.0'// 云信
import TagsView from './TagsView'
// import { mapState } from 'vuex';
// import { resetRouter } from '@/router';
export default {
  components: {
    ScreenFull,
    TagsView
  },
  data () {
    return {
      nim: null,
      duty: 0,
      player: {
        new: new Audio(require('@/assets/sound/wechat_new.mp3')),
        cur: new Audio(require('@/assets/sound/wechat_cur.mp3'))
      },
      icon: this.$store.state.reactive.isCollapse,
      timer: null,
      cs: false,
      //currentTime: new Date(),
      levelList: [],
      editableTabsValue: '',
      editableTabs: [
        {
          title: '首页',
          name: 'layout',
          path: '/layout'
        }
      ]
    }
  },
  // beforeMount() {
  //   window.addEventListener('resize', this.resizeHandler)
  // },
  created () {
    //this.handleDateTime();
    //console.log('admin', this.$store.getters['permission/admin'])
    let admin = this.$store.getters['permission/admin']
    console.info('admin', admin)
    if (admin.menu_list.includes('cs')) {
      this.cs = true
      apiMessage.config({ type: 'YUNXIN' }).then(res => {
        if (res.code === 2000) {
          this.nim = new NIM({
            debug: false,
            appKey: res.data.app_id,
            account: admin.admin.yunxin_account,
            token: admin
              .admin.yunxin_token,
            //db: false,
            autoMarkRead: false,
            rollbackDelMsgUnread: true,// 表示收到撤回消息是否同时撤销被撤回消息影响的未读数，默认为false。决定是否会将被撤回的消息计入未读计算。如置为true，则在被撤回消息未读场景下，撤回时未读消息数减一。
            //syncSessionUnread: true,// 是否同步会话未读数(开启数据库时有效，保证多端未读数相一致)
            onconnect: this.onConnect,
            onerror: this.onError,
            onwillreconnect: this.onWillReconnect,
            ondisconnect: this.onDisconnect,
            // 多端
            onloginportschange: this.onLoginPortsChange,
            // 会话
            //onsessions: this.onSessions,
            onupdatesession: this.onUpdateSession,
            // 消息
            onroamingmsgs: this.onRoamingMsgs,
            onofflinemsgs: this.onOfflineMsgs,
            onmsg: this.onMsg,
            onsysmsg: this.onSysMsg,//系统通知
            // 同步完成
            onsyncdone: this.onSyncDone
          })

          this.$store.commit('message/nim', this.nim)
        } else {
          this.$message.error(res.msg)
        }
      })
    }
  },
  computed: {
    unread () {
      return this.$store.getters['message/contacts'].reduce((total, item) => (total + item.unread), 0)
    },
    isMobile () {
      return this.$store.state.reactive.isMobile
    },
    isCollapse () {
      return this.$store.state.reactive.isCollapse
    },
    online () {
      return this.$store.getters['message/online']
    }
  },
  methods: {
    onConnect (obj) {
      // lastLoginDeviceId: 上次登录的设备的设备号
      // customTag: 客户端自定义tag
      // connectionId: 本次登录的连接号
      // ip: 客户端 IP
      // port: 客户端端口
      // country: 本次登录的国家
      console.info('连接成功', obj)
      this.$store.commit('message/online', true)
      this.$store.commit('message/connectionInfo', obj)
      this.$store.commit('message/error', {})
      if (this.nim) {
        this.nim.getServerSessions({
          //this.nim.getLocalSessions({
          // lastSessionId: lastSessionId,
          limit: 100,// 为本次查询的会话数量限制, 最多 100 条, 默认 100 条
          done: (error, obj) => {
            console.log('获取会话列表', error, obj)
            if (error) {
              console.warn('获取会话列表失败', error, obj)
              this.$message.error(`${this.$t('cs.yunxin')}获取会话列表失败：${error.message}`)
            } else {
              // 本地是 sessions, 服务端是 sessionList
              if (obj.sessionList.length) {
                this.$store.dispatch('message/setContacts', obj.sessionList)
              }
            }
          }
        })
      }
    },
    onWillReconnect (obj) {// 此时说明 `SDK` 已经断开连接, 请开发者在界面上提示用户连接已断开, 而且正在重新建立连接
      this.$store.commit('message/online', false)
      console.warn('即将重连', obj)
      this.$message.info(`${this.$t('cs.yunxin')}：即将重连`)
    },
    onDisconnect (error) {// 此时说明 `SDK` 处于断开状态, 开发者此时应该根据错误码提示相应的错误信息, 并且跳转到登录页面
      this.$store.commit('message/online', false)
      this.$store.commit('message/error', error)
      console.warn('连接断开', error)
      this.$message.warning(`${this.$t('cs.yunxin')}连接断开：${error.message}`)
      if (error) {
        switch (error.code) {
          case 302:// 账号或者密码错误, 请跳转到登录页面并提示错误
            console.warn(`${this.$t('cs.yunxin')}：账号或者密码错误`, error)
            this.$message.error(`${this.$t('cs.yunxin')}账号或者密码错误：${error.message}`)
            break

          case 417:// 重复登录, 已经在其它端登录了, 请跳转到登录页面并提示错误
            alert(`${this.$t('cs.yunxin')}：已经在其它地方登录了`)
            break

          case 'kicked':// 被踢, 请提示错误后跳转到登录页面
            alert(`${this.$t('cs.yunxin')}：被踢了`)
            break

          default:
            this.$message.error(`${this.$t('cs.yunxin')}：...`)
            break
        }
      }
    },
    onError (error, obj) {
      console.warn('发生错误', error, obj)
    },
    onLoginPortsChange (loginPorts) {
      console.warn('当前登录帐号在其它端的状态发生改变了', loginPorts)
    },
    onSessions (sessions) {
      console.info('收到会话列表', sessions)
      this.$store.dispatch('message/setContacts', sessions)
    },
    onUpdateSession (session) {
      console.info('会话更新了', session)
      this.$store.dispatch('message/setContact', session)
    },
    onRoamingMsgs (obj) {
      console.info('收到漫游消息', obj)
    },
    onOfflineMsgs (obj) {
      console.info('收到离线消息', obj)
    },
    onMsg (msg) {
      console.info('收到消息', msg)
      if (msg.flow === 'in' && this.duty === 1) {
        // 当前聊天对象
        if (this.$store.getters['message/curr'].to === msg.from) {
          this.player.cur.currentTime = 0
          this.player.cur.play()
        } else {
          this.player.new.currentTime = 0
          this.player.new.play()
        }
      }
      this.$store.commit('message/chats', {
        sessionId: msg.flow === 'in' ? msg.from : msg.to,
        msgs: [msg],
        push: true
      })
    },
    onSysMsg (sysMsg) {
      console.info('收到系统通知', sysMsg)
    },
    onSyncDone () {
      console.info('同步完成')
    },
    switchDuty () {
      console.log('this.duty', this.duty)
      apiMessage.duty({ status: this.duty }).then(res => {
        if (res.code !== 2000) {
          this.duty = this.duty ? 0 : 1
        }
      })
    },
    handleCs (command) {
      if (command == 'message') {
        if (this.$route.path != '/message') {
          this.$router.push('/message')
        }

        this.$bus.$emit("tabsChange", {
          title: this.$t('cs.msg'),
          name: 'cs',
          path: '/message'
        })
      }
    },
    removeTab (targetName) {
      // let former = this.editableTabsValue
      let tabs = [...this.editableTabs];
      let activeName = this.editableTabsValue;
      if (activeName === targetName) {
        tabs.forEach((item, index) => {
          if (item.name == targetName) {
            let obj = tabs[index + 1] || tabs[index - 1];
            this.editableTabsValue = obj.name;
            if (tabs[index + 1]) {
              this.$router.push(tabs[index + 1].path);
            } else {
              this.$router.push(tabs[index - 1].path);
            }
          }
        })
      }
      this.editableTabs = tabs.filter((item) => item.name != targetName);
      sessionStorage.setItem("TabsViews", JSON.stringify(this.editableTabs));
    },
    handleClick (val) {
      if (this.$route.name == val) {
        return
      }
      this.editableTabs.filter((item) => {
        if (item.name == val) {
          this.$router.push(item.path)
          return
        }
      })
    },
    handleDateTime () {
      let _this = this
      this.timer = setInterval(() => {
        let y = new Date().getFullYear()
        let m = new Date().getMonth() + 1
        let d = new Date().getDate()
        let h = new Date().getHours()
        let min = new Date().getMinutes()
        let s = new Date().getSeconds()
        min = min < 10 ? "0" + min : min
        s = s < 10 ? "0" + s : s
        //修改数据date
        _this.currentTime = y + "-" + m + "-" + d + " " + h + ":" + min + ": " + s
      }, 1000)
    },
    changIcon () {
      this.icon = !this.icon
      this.$store.commit("reactive/SET_COLLAPSE", this.icon);
    },
    Refresh () {
      this.$router.replace({
        path: '/refresh',
        query: {
          t: Date.now()
        }
      })
    },
    handleCommand (command) {
      if (command == '1') {
        this.amend();
      } else if (command == '2') {
        this.password();
      } else if (command == '3') {
        this.getOut();
      }
    },
    getOut () {
      console.log('退出')
      // resetRouter();
      sessionStorage.removeItem('access_token')
      sessionStorage.removeItem('admin')
      removeCookie('token')
      this.$store.dispatch('tagsView/delAllViews');
      this.$router.push("/")
    },
    amend () {
      if (this.$route.path != '/amendMaterial') {
        this.$router.push('/amendMaterial')
      }
    },
    password () {
      if (this.$route.path != '/amendPassword') {
        this.$router.push('/amendPassword')
      }
    }
  },
  watch: {
    immediate: true,
    isMobile (value) {
      this.icon = value;
      this.$store.commit("reactive/SET_COLLAPSE", this.icon);
    },
    isCollapse (value) {
      this.icon = value;
    }
  },
  beforeDestroy () {
    if (this.timer) {
      clearInterval(this.timer); // 在Vue实例销毁前，清除我们的定时器
    }
  }
}
</script>
<style lang="scss" scoped>
.navbar {
  color: #fff;
  background-color: #333742;
  padding: 0 20px;
  height: 50px;
  display: flex;
  justify-content: space-between;
  align-items: center;

  .palette {
    font-size: 20px;
    > i {
      margin: 0 10px;
    }
  }
  .breadcrumb {
    display: flex;
    align-items: center;
  }
  .Account {
    display: flex;
    align-items: center;
  }
}
>>> .el-tabs__header {
  margin: 0;
}
>>> .el-tabs__item.is-active {
  color: #209688;
}
>>> .el-tabs__active-bar {
  background-color: #209688;
}
>>> .el-tabs__item .el-icon-close:before {
  transform: scale(1);
}
.el-dropdown-link {
  cursor: pointer;
  color: #fff;
  font-size: 14px;
}
.el-icon-arrow-down {
  font-size: 12px;
}
.demonstration {
  display: block;
  color: #8492a6;
  font-size: 14px;
  margin-bottom: 20px;
}
.offline {
  animation: twinkling 0.8s infinite linear;
}
.new-msg {
  animation: twinkling 1.6s infinite linear;
}
@keyframes twinkling {
  from {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
>>> .el-dropdown-menu__item {
  text-align: right;
}
</style>
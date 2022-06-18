<template>
    <el-container>
        <el-container :class="{webapp: $store.state.reactive.isMobile}">
            <el-aside>
                <vuescroll ref="contacts" :ops="opts.scroller" style="background-color:#ECEAE8">
                    <ul class="contacts-list" v-if="contacts && contacts.length">
                        <li class="contacts-list-item" v-for="item in contacts" :key="item.id" :class="{'active': item.id === curr.id}" @click="switchContact(item)">
                            <el-row>
                                <el-col class="mobile" :xs="5" :sm="5" :md="5" :lg="5" :xl="5">
                                    <el-badge :max="99" :hidden="item.unread === 0" :value="item.unread" class="badge">
                                        <div class="info">
                                            <span class="name">{{nickname(item.to)}}</span>
                                        </div>
                                    </el-badge>
                                </el-col>
                                <el-col class="pc" :xs="5" :sm="5" :md="5" :lg="5" :xl="5">
                                    <el-badge :max="99" :hidden="item.unread === 0" :value="item.unread" class="badge">
                                        <el-avatar shape="square" size="large" :src="avatar(item.to)" @error="errorHandler">
                                            <img :src="`${require('@/assets/images/avatar.png')}`" />
                                        </el-avatar>
                                    </el-badge>
                                </el-col>
                                <el-col class="pc" :xs="19" :sm="19" :md="19" :lg="19" :xl="19">
                                    <div class="info">
                                        <span class="name">{{nickname(item.to)}}</span>
                                        <span class="time">{{getLastTime(item.lastMsg.time)}}</span>
                                    </div>
                                    <div class="chat" v-html="getLastMsg(item)"></div>
                                </el-col>
                            </el-row>
                        </li>
                    </ul>
                </vuescroll>
            </el-aside>
            <el-container>
                <el-image v-show="!curr.id" fit="contain" style="width:100px;height:100%;margin:0 auto;opacity:.3" :src="`${require('@/assets/images/logo3.png')}`"></el-image>
                <el-header height="50px" v-show="curr.id">
                    <span v-if="curr.to">{{nickname(curr.to)}}</span>
                    <i class="iconfont icon-sousuoliaotianjilu" title="聊天记录" @click="search.show = true"></i>
                </el-header>
                <el-main v-show="curr.id">
                    <vuescroll ref="chat" :ops="opts.scroller" style="background-color:#F5F5F5">
                        <!-- <i v-show="loadMore" class="el-icon-loading" style="position:absolute;top:0"></i> -->
                        <div id="chat">
                            <ul>
                                <li v-for="(item, index) in chats" :key="index" :class="{master: item.flow === 'out', 'time-format': item.time_format}" :id="`idServer-${item.idServer}`">
                                    <div class="time" v-if="item.time_format">{{item.time_format}}</div>
                                    <el-avatar shape="square" size="medium" :src="item.flow === 'out' ? csAvatar : avatar(item.from)" @error="errorHandler">
                                        <img :src="`${require('@/assets/images/avatar.png')}`" />
                                    </el-avatar>
                                    <div class="info">
                                        <!-- <div class="time">{{$moment(item.time).format("YY/M/D H:mm:ss")}}</div> -->
                                        <div class="content" :class="{image: item.type === 'image', transparent: /\[dynamic:\d+\]/.test(item.text)}">
                                            <el-image v-if="item.type === 'image'" :src="item.file.url" :preview-src-list="[item.file.url]">
                                                <div slot="error" class="image-slot">
                                                    <i class="el-icon-picture-outline"></i>
                                                </div>
                                            </el-image>
                                            <div v-else v-html="textReg(item.text, false)"></div>
                                            <i v-if="item.type === 'text' && !/\[dynamic:\d+\]/.test(item.text)" :class="item.flow === 'out' ? 'el-icon-caret-right' : 'el-icon-caret-left'"></i>
                                            <i class="el-icon-error" v-if="item.flow === 'out'" @click.stop="del(item)"></i>
                                        </div>
                                        <div class="read" v-if="item.isRemoteRead">已读</div>
                                    </div>
                                </li>
                            </ul>
                        </div>
                    </vuescroll>
                </el-main>
                <el-footer height="130px" v-show="curr.id">
                    <div v-show="unread > 0" id="unread" class="success">
                        <span @click="scrollDown"><i class="el-icon-d-arrow-right"></i>{{unread}}条新消息</span>
                        <el-divider direction="vertical"></el-divider>
                        <el-button size="small" type="text" icon="el-icon-close" @click.native.stop="unread = 0"></el-button>
                    </div>
                    <div class="tools">
                        <!-- 发送表情 -->
                        <popper v-if="emoji" root-class="emoji-popper" trigger="clickToOpen" :options="{
                                placement: 'top',
                                gpuAcceleration: true,
                                modifiers: { offset: { offset: '0,10px' } }
                            }">
                            <div class="popper">
                                <el-tabs type="border-card" tab-position="bottom">
                                    <el-tab-pane v-if="emoji.emoji">
                                        <i slot="label" class="iconfont icon-biaoqing"></i>
                                        <div class="emoji">
                                            <img v-for="item in emoji.emoji" :key="item.code" :src="item.pic_url" @click="addEmoji(item.code)" :title="item.name" />
                                        </div>
                                    </el-tab-pane>
                                    <el-tab-pane v-if="emoji.meme">
                                        <i slot="label" class="iconfont icon-dongtutexiao-01"></i>
                                        <div class="emoji">
                                            <img v-for="item in emoji.meme" :key="item.code" :src="item.pic_url" @click="sendMeme(item.code)" :title="item.name" />
                                        </div>
                                    </el-tab-pane>
                                </el-tabs>
                            </div>
                            <i slot="reference" class="iconfont icon-biaoqing" title="表情"></i>
                        </popper>
                        <!-- 发送图片 -->
                        <el-upload action="url" multiple :show-file-list="false" accept=".gif,.png,.jpg,.jpeg" :http-request="sendFile">
                            <i class="iconfont icon-tupian" title="图片"></i>
                        </el-upload>
                        <!-- 快捷回复 -->
                        <popper v-if="quickReply" root-class="quick-reply-popper" trigger="clickToOpen" :options="{
                                placement: 'top',
                                gpuAcceleration: true,
                                modifiers: { offset: { offset: '0,10px' } }
                            }">
                            <div class="popper">
                                <el-table :data="quickReply" :show-header="false" @cell-click="quickReplyClick">
                                    <el-table-column width="300" label="" prop="content"></el-table-column>
                                </el-table>
                            </div>
                            <i slot="reference" style="margin-left:2px" class="iconfont icon-kuaijiehuifu1" title="快捷回复"></i>
                        </popper>
                    </div>
                    <div class="input">
                        <textarea ref="input" v-model="content" @keydown.enter.prevent="lineFeed($event)"></textarea>
                    </div>
                    <el-popover placement="top-end" trigger="manual" popper-class="empty-popover" content="不能发送空白信息" v-model="emptyTxtTip">
                        <el-button slot="reference" size="mini" :disabled="!content" @click="sendMsg" title="按Enter键发送，按Ctrl+Enter换行">发送(S)</el-button>
                    </el-popover>
                </el-footer>
            </el-container>
        </el-container>
        <!-- 发送图片进度 -->
        <el-dialog
            :title="`发送图片(${upload.list.length})`"
            :visible="upload.show"
            width="480px"
            :show-close="false"
            :close-on-click-modal="false"
            :close-on-press-escape="false">
            <el-row :gutter="20" v-for="(item, index) in upload.list" :key="index">
                <el-col :span="4">
                    <img :src="item.file">
                </el-col>
                <el-col :span="20">
                    <el-progress :text-inside="true" :stroke-width="20" :percentage="item.percentage" :color="upload.colors"></el-progress>
                </el-col>
            </el-row>
        </el-dialog>
        <!-- 聊天记录搜索 -->
        <el-dialog v-if="curr.to" class="search"
            :title="nickname(curr.to)"
            :visible.sync="search.show"
            :close-on-click-modal="false">
            <div class="input">
                <el-input placeholder="搜索" v-model.trim="search.content" clearable size="mini" prefix-icon="el-icon-search">
                </el-input>
            </div>
            <el-tabs v-model="search.active">
                <el-tab-pane label="全部" name="all">
                    <div class="result" v-show="search.content">{{searchList('all').length}}条与“{{search.content}}”相关的聊天记录</div>
                    <ul v-if="searchList('all').length" class="search-list">
                        <li v-for="(item, index) in searchList('all')" :key="index">
                            <el-row>
                                <el-col :xs="0" :sm="4" :md="3" :lg="2">
                                    <el-avatar shape="square" size="medium" :src="item.flow === 'out' ? csAvatar : avatar(item.from)" @error="errorHandler">
                                        <img :src="`${require('@/assets/images/avatar.png')}`" />
                                    </el-avatar>
                                </el-col>
                                <el-col :xs="18" :sm="15" :md="17" :lg="18">
                                    <div class="name">{{item.flow === 'out' ? $store.getters['permission/admin'].admin.nickname : nickname(curr.to)}}</div>
                                    <div class="content">
                                        <el-image v-if="item.type === 'image'" :src="item.file.url" :preview-src-list="[item.file.url]">
                                            <div slot="error" class="image-slot">
                                                <i class="el-icon-picture-outline"></i>
                                            </div>
                                        </el-image>
                                        <div v-else v-html="textReg(item.text, false)"></div>
                                    </div>
                                </el-col>
                                <el-col class="time" :xs="6" :sm="5" :md="4" :lg="3">{{getLastTime(item.time)}}</el-col>
                            </el-row>
                        </li>
                    </ul>
                    <div v-else class="empty">无结果</div>
                </el-tab-pane>
                <el-tab-pane label="图片" name="file">
                    <div class="result" v-show="search.content">{{searchList('file').length}}条与“{{search.content}}”相关的图片</div>
                    <ul v-if="searchList('file').length" class="search-list">
                        <li v-for="(item, index) in searchList('file')" :key="index">
                            <el-row>
                                <el-col :xs="0" :sm="4" :md="3" :lg="2">
                                    <el-avatar shape="square" size="medium" :src="item.flow === 'out' ? csAvatar : avatar(item.from)" @error="errorHandler">
                                        <img :src="`${require('@/assets/images/avatar.png')}`" />
                                    </el-avatar>
                                </el-col>
                                <el-col :xs="18" :sm="15" :md="17" :lg="18">
                                    <div class="name">{{item.flow === 'out' ? $store.getters['permission/admin'].admin.nickname : nickname(curr.to)}}</div>
                                    <div class="content">
                                        <el-image :src="item.file.url" :preview-src-list="[item.file.url]">
                                            <div slot="error" class="image-slot">
                                                <i class="el-icon-picture-outline"></i>
                                            </div>
                                        </el-image>
                                    </div>
                                </el-col>
                                <el-col class="time" :xs="6" :sm="5" :md="4" :lg="3">{{getLastTime(item.time)}}</el-col>
                            </el-row>
                        </li>
                    </ul>
                    <div v-else class="empty">无结果</div>
                </el-tab-pane>
                <el-tab-pane label="链接" name="link">
                    <div class="result" v-show="search.content">{{searchList('link').length}}条与“{{search.content}}”相关的链接</div>
                    <ul v-if="searchList('link').length" class="search-list">
                        <li v-for="(item, index) in searchList('link')" :key="index">
                            <el-row>
                                <el-col :xs="0" :sm="4" :md="3" :lg="2">
                                    <el-avatar shape="square" size="medium" :src="item.flow === 'out' ? csAvatar : avatar(item.from)" @error="errorHandler">
                                        <img :src="`${require('@/assets/images/avatar.png')}`" />
                                    </el-avatar>
                                </el-col>
                                <el-col :xs="18" :sm="15" :md="17" :lg="18">
                                    <div class="name">{{item.flow === 'out' ? $store.getters['permission/admin'].admin.nickname : nickname(curr.to)}}</div>
                                    <div class="content">
                                        <div v-html="textReg(item.text, false)"></div>
                                    </div>
                                </el-col>
                                <el-col class="time" :xs="6" :sm="5" :md="4" :lg="3">{{getLastTime(item.time)}}</el-col>
                            </el-row>
                        </li>
                    </ul>
                    <div v-else class="empty">无结果</div>
                </el-tab-pane>
            </el-tabs>
        </el-dialog>
        <!-- 错误信息 -->
        <el-dialog class="error" :title="`${$t('cs.yunxin')}`"
        :visible="!!error.message"
        :close-on-click-modal="false"
        :close-on-press-escape="false"
        :show-close="false"
        :modal-append-to-body="false">
        <div class="main">
            <i class="el-icon-warning danger"></i>
            <span>{{error.message}}</span>
        </div>
        </el-dialog>
    </el-container>
</template>

<script>
import vuescroll from 'vuescroll/dist/vuescroll-native'
import Popper from 'vue-popperjs'
import 'vue-popperjs/dist/vue-popper.css'
import { apiMessage } from '@/request/api'
import { getBase64, keywordsColorful } from '@/utils/tools'
export default {
    name: 'message',
    components: {
        vuescroll,
        Popper
    },
    data() {
        return {
            //lastIdServer: String,
            //height: Number,//容器高度
            pageSize: 100,
            //loadMore: false,
            emptyTxtTip: false,
            unread: 0,
            curr: {},
            emoji: [],
            content: '',
            urlReg: /(https?:\/\/)?([A-Za-z0-9]+\.[A-Za-z0-9]+[\/=\?%\-&_~`@[\]\':+!]*([^<>\"\"])*)/g,
            search: {
                show: false,
                active: 'all',
                content: ''
            },
            upload: {
                show: false,
                list: [],
                colors: [
                    {color: '#909399', percentage: 30},
                    {color: '#e6a23c', percentage: 70},
                    {color: '#67c23a', percentage: 100}
                ],
                timer: null
            },
            quickReply: [{
                id: 1,
                content: '您好，请问有什么可以为您服务。'
            }, {
                id: 2,
                content: '非常感谢您这么好的建议，我们会向上级部门反映，因为有了您的建议，我们才会不断进步。'
            }],
            opts: {
                scroller: {
                    scrollPanel: {
                        scrollingX: true,
                        scrollingY: true
                    },
                    rail: {
                        gutterOfSide: 0
                    },
                    bar: {
                        onlyShowBarOnScroll: false,
                        //keepShow: true,
                        background: '#c3c3c3'
                    }
                }
            }
        }
    },
    created() {
        apiMessage.emoji().then(res => {
            if(res.code === 2000){
                this.emoji = res.data
            }else {
                this.$message.error(res.msg)
            }
        })

        const that = this
        document.onkeydown = function(e) {
            let len
            if((len = that.contacts.length) && that.curr.id && !that.content) {
                //事件对象兼容
                const e1 = e || window.event || arguments.callee.caller.arguments[0]
                const index = that.contacts.findIndex(item => item.id === that.curr.id)
                if(e1 && e1.keyCode == 38) {//上箭头
                    if(index === 0) {
                        return
                    }
                    that.switchContact(that.contacts[index - 1])
                }else if(e1 && e1.keyCode == 40) {//下箭头
                    if(index === len - 1) {
                        return
                    }
                    that.switchContact(that.contacts[index + 1])
                }
            }
        }
    },
    computed: {
        error() {
            console.log(this.$store.getters['message/error'])
            return this.$store.getters['message/error']
        },
        searchList() {
            return type => {
                let chats = []
                Object.assign(chats, this.$store.getters['message/chats'](this.curr.to))

                if(!chats.length) {
                    return chats
                }

                switch(type) {
                    case 'file':
                        chats = chats.filter(item => (item.type === 'image'))
                    break

                    case 'link':
                        chats = chats.filter(item => (
                            item.type === 'text' &&
                            this.urlReg.test(item.text)
                        ))
                    break
                }

                if(this.search.content) {
                    chats = chats.filter(item => (
                        item.type === 'text' &&
                        !/\[(emoji|dynamic):\d+\]/.test(item.text) &&
                        new RegExp(`(${this.search.content})`, 'g').test(item.text)
                    ))
                }

                return chats
            }
        },
        nickname() {
            return to => (this.$store.getters['message/users'] && this.$store.getters['message/users'][to] && this.$store.getters['message/users'][to].nickname) || `用户${to.replace(/[^\d]/, '')}`
        },
        avatar() {
            return to => (this.$store.getters['message/users'] && this.$store.getters['message/users'][to] && this.$store.getters['message/users'][to].avatar_url) || require('@/assets/images/avatar.png')
        },
        csAvatar() {
            return this.$store.getters['permission/admin'].admin.avatar || require('@/assets/images/logo2.png')
        },
        nim() {
            return this.$store.getters['message/nim'] || null// 网易云信对象
        },
        contacts() {
            let contact = this.$store.getters['message/contact']
            if(contact.id && this.curr.id && contact.id === this.curr.id) {// 来新消息了
                this.$nextTick(() => {
                    const process = this.$refs['chat'].getScrollProcess().v
                    if(process > .85 || (contact.lastMsg && contact.lastMsg.flow === 'out')) {
                        this.scrollDown()
                    }else {// 展示新消息按钮
                        this.unread++
                    }
                })
            }
            return this.$store.getters['message/contacts'] || []
        },
        chats() {
            if(this.curr.to) {
                let timestamp = 0,
                    noshow = 0 //不显示时间条数

                let msgReceiptTime = this.$store.getters['message/contacts'].find(item => item.id === this.curr.id).msgReceiptTime

                return (this.$store.getters['message/chats'](this.curr.to) || []).map(item => {
                    let time_format
                    if(!timestamp || item.time > timestamp + 300 * 1000 || noshow > 8) {
                        timestamp = item.time
                        noshow = 0
                        time_format = this.getLastTime(item.time, 2)
                    }else {
                        noshow++
                    }
                    // 查询消息是否被对方读过了
                    let isRemoteRead = false
                    if(item.flow === 'out') {
                        if(msgReceiptTime) {
                            if(item.time < msgReceiptTime) {
                                isRemoteRead = true
                            }
                        }else {
                            isRemoteRead = this.nim.isMsgRemoteRead(item)
                        }
                    }

                    return {
                        ...item,
                        time_format: time_format ? time_format : '',
                        isRemoteRead: isRemoteRead
                    }
                })
            }
            return []
        },
        online() {
            return this.$store.getters['message/online']
        }
    },
    methods: {
        errorHandler() {// 头像加载失败
            return true
        },
        quickReplyClick(row, column, cell, event) {// 快捷回复
            this.sendMsg('', row.content)
            this.focus()
        },
        getLastTime(time, type = 1) {
            const moment = this.$moment(time)
            // 当天0点
            const today = this.$moment().startOf('day').format('x')
            if(time > today) {// 当天
                return moment.format('H:mm')
            }else if(time > today - 86400000) {// 昨天
                return '昨天' + (type === 2 ? ` ${moment.format('H:mm')}` : '')
            }
            // 本周0点（周一）
            const week = this.$moment().startOf('week').format('x')
            if(time > week) {
                return moment.format('dddd') + (type === 2 ? ` ${moment.format('H:mm')}` : '')
            }

            return type === 1 ? moment.format('YY/M/D') : `${moment.format('YYYY年MM月DD日')} ${moment.format('H:mm')}`
        },
        getLastMsg(item) {
            let draft = this.$store.getters['message/draft'](item.id) || ''
            if(draft) { //有草稿
                return `<font class="txt draft">[草稿]</font>${this.textReg(draft)}`
            }

            switch(item.lastMsg.type) {
                case 'text':
                    let arr
                    if(arr = /\[dynamic:(\d+)\]/.exec(item.lastMsg.text)) {
                        if(arr[1] && this.emoji.meme.length) {
                            let obj = this.emoji.meme.find(item => item.code === `[dynamic:${arr[1]}]`)
                            if(obj && obj.name) {
                                return `[${obj.name}]`
                            }
                            return '[动画表情]'
                        }
                    }
                    return this.textReg(item.lastMsg.text)

                case 'image':
                    return '[图片]'
            }
            return ''
        },
        textReg(txt, trim = true) {
            if(!txt){
                return txt
            }

            let arr
            if(/\[emoji:\d+\]/.test(txt) && this.emoji.emoji.length) {
                txt = `<font class="txt">${txt}</font>`
                txt = txt.replace(/\[emoji:(\d+)\]/g, ($1, $2) => {
                    let obj = this.emoji.emoji.find(item => item.code === `[emoji:${$2}]`)
                    if(obj) {
                        return `</font><img class="msg-emoji" src="${obj.pic_url}" title="${obj.name}" alt="${obj.name}" /><font class="txt">`
                    }
                })
                txt = txt.replace(/(<font class="txt"><\/font>)/g, '')
            }else if(arr = /\[dynamic:(\d+)\]/.exec(txt)) {
                if(arr[1] && this.emoji.meme.length) {
                    let obj = this.emoji.meme.find(item => item.code === `[dynamic:${arr[1]}]`)
                    if(obj) {
                        return `<img class="msg-meme" src="${obj.pic_url}" title="${obj.name}" alt="${obj.name}" />`
                    }
                }
            }else {
                if(this.search.content) {// 输入了搜索关键词
                    txt = keywordsColorful(txt, this.search.content)
                }
                // 替换超链接
                txt = txt.replace(this.urlReg, (a, b, c) => (`<a class="msg-link" target="_blank" href="//${c}">${a}</a>`))
            }

            return trim ? txt.trim().replace(/[\r\n]|\n/g,'') : txt.replace(/[\r\n]|\n/g, '<br>')
        },
        del(msg) {
            this.nim.deleteMsg({
                msg: msg,
                done: error => {
                    if(error) {
                        console.log('撤回消息失败', error)
                        this.$message.error(`${this.$t('cs.yunxin')}：${error.message}`)
                    }else {

                    }
                }
            })
        },
        sendFile(file) {
            if(!file || !this.online){
                return
            }

            let size = file.file.size
            if(size > 100 * 1024 ** 2) { // 大于100M
                this.$message.error('不能发送超过100M大小的图片')
                return
            }

            let type = file.file.type
            if(!/^image\/(gif|png|jpe?g)$/i.test(type)) {
                this.$message.error('只能发送gif,png,jpg,jpeg格式图片')
                return
            }

            const that = this
            let index
            getBase64(file.file).then(base64 => {
                this.nim.sendFile({
                    scene: 'p2p',
                    to: this.curr.to,
                    type: 'image',
                    dataURL: base64,
                    beginupload: upload => {
                        // - 如果开发者传入 fileInput, 在此回调之前不能修改 fileInput
                        // - 在此回调之后可以取消图片上传, 此回调会接收一个参数 `upload`, 调用 `upload.abort();` 来取消文件上传
                        if(!that.upload.show) {
                            that.upload.show = true
                        }

                        index = that.upload.list.push({
                            file: base64,
                            percentage: 0
                        }) - 1
                    },
                    uploadprogress: obj => {
                        that.upload.list[index].percentage = obj.percentage
                        //console.log('文件总大小: ' + obj.total + 'bytes')
                        //console.log('已经上传的大小: ' + obj.loaded + 'bytes')
                        //console.log('上传进度: ' + obj.percentage)
                        //console.log('上传进度文本: ' + obj.percentageText)
                    },
                    uploaddone: (error, file) => {
                        //this.upload.show = false
                        if(error){
                            console.warn('上传图片失败', error, file)
                            that.$message.error(`${this.$t('cs.yunxin')}：发送图片失败`)
                        }
                    },
                    beforesend: msg => {
                        that.$store.commit('message/chats', {
                            sessionId: that.curr.to,
                            msgs: [msg],
                            push: true
                        })
                    },
                    done: (error, msg) => {
                        if(error) {
                            console.warn('发送图片失败', error, msg)
                            that.$message.error(`${this.$t('cs.yunxin')}：发送图片失败`)
                        }
                    }
                })
            })
        },
        addEmoji(code) {
            this.content += code
            this.focus()
        },
        sendMeme(code) {
            this.sendMsg('', code)
            this.focus()
        },
        // resize(vertical, horizontal, nativeEvent) {
        //     //console.log('resize-nativeEvent', nativeEvent)
        //     if(this.loadMore){
        //         this.$refs['chat'].scrollIntoView(`#idServer-${this.lastIdServer}`, 1)
        //         // this.$refs['chat'].scrollBy({
        //         //     dy: nativeEvent.height - this.height
        //         // }, 1)
        //         this.loadMore = false
        //     }
        //     //this.height = nativeEvent.height
        // },
        // scrolling(vertical, horizontal) {
        //     if(vertical.process === 0 && !this.loadMore){
        //         console.log('加载更多')
        //         let msg = this.$store.getters['message/chats'](this.curr.to)[0]
        //         this.loadMore = true
        //         this.lastIdServer = msg.idServer
        //         this.getHistoryMsgs(msg)
        //     }else if(vertical.process === 1 && this.unread > 0) {
        //         this.unread = 0
        //     }
        // },
        lineFeed(e) {
            if(this.content) {
                if(e.ctrlKey && e.keyCode === 13) {
                    console.log(e.ctrlKey, e.keyCode)
                    this.content += '\n'
                }else if(e.keyCode === 13) {
                    this.sendMsg()
                }
            }else {
                this.emptyTxtTip = true
                setTimeout(() => {
                    this.emptyTxtTip = false
                }, 2000);
            }
            return false
        },
        focus() {
            this.$refs['input'].focus()
            this.$refs['input'].click()
        },
        empty() {
            this.content = ''
        },
        sendMsg(event, content = '') {
            if(!this.online) {
                return
            }

            const msg = this.nim.sendText({
                scene: 'p2p',
                to: this.curr.to,
                text: content ? content : this.content,
                needMsgReceipt: true,
                done: (error, msg) => {
                    if(error) {
                        console.warn('发送消息失败', error, msg)
                        this.$message.error(`${this.$t('cs.yunxin')}：发送消息失败`)
                    }else {

                    }
                }
            })

            this.$store.commit('message/chats', {
                sessionId: this.curr.to,
                msgs: [msg],
                push: true
            })
            //清空输入框
            if(!content) {
                this.empty()
            }
        },
        switchContact(item) {// 切换联系人
            if(this.curr && this.curr.id === item.id){
                return
            }
            const old_id = this.curr.id

            this.curr = {...item}
            this.$store.commit('message/curr', this.curr)
            // 设置当前会话
            this.nim.setCurrSession(this.curr.id)

            let chat = this.$store.getters['message/chats'](this.curr.to)
            if(!chat.length) {//没有内容
                this.getHistoryMsgs(null, true)
            }else {
                //if(chat.length < this.pageSize){
                    this.getHistoryMsgs(chat[0], true)
                //}else {
                    //this.scrollDown()
                //}
            }

            if(this.unread > 0) {
                this.unread = 0
            }

            this.search.active = 'all'
            this.search.content = ''
            // 保存草稿
            if(this.content) {
                this.$store.commit('message/draft', {
                    sessionId: old_id,
                    msg: this.content
                })
                this.empty()
            }
            // 获取草稿
            let draft = this.$store.getters['message/draft'](this.curr.id) || ''
            if(draft) {
                this.content = draft
                this.$store.commit('message/draft', {
                    sessionId: this.curr.id,
                    msg: ''
                })
            }
            // 发送已读回执
            this.nim.sendMsgReceipt({
                msg: item.lastMsg,
                done: (error, obj) => {
                    if(error) {
                        console.error('发送消息已读回执失败', error, obj)
                    }
                }
            })

            this.$nextTick(() => {
                this.focus()
            })
        },
        getHistoryMsgs(lastMsg = null, scroll2Down = false) { // 获取历史聊天记录
            let params = {
                scene: 'p2p',
                to: this.curr.to,
                asc: true,
                limit: this.pageSize,
                done: (error, obj) => {
                    if(!error) {
                        if(obj.msgs.length){
                            console.log("🚀 ~ file: message.vue ~ line 748 ~ getHistoryMsgs ~ obj.msgs", obj.msgs)
                            // 递归拉取聊天记录
                            this.getHistoryMsgs(obj.msgs[0], true)

                            this.$store.commit('message/chats', {
                                sessionId: this.curr.to,
                                msgs: obj.msgs,
                                push: lastMsg ? false : true
                            })

                            // if(scroll2Down) {
                            //     this.scrollDown()
                            // }
                        }else if(scroll2Down) {
                            if(this.unread > 0) {
                                this.unread = 0
                            }
                            this.scrollDown()
                        }
                    }else {
                        this.$message.error(`${this.$t('cs.yunxin')}：获取历史消息失败`)
                    }
                }
            }

            if(lastMsg) {
                params.lastMsgId = lastMsg.idServer
                params.endTime = lastMsg.time
            }

            this.nim.getHistoryMsgs(params)
        },
        scrollDown() {
            setTimeout(() => {
                this.$refs['chat'].scrollTo({
                    y: '200%'
                }, 1)
            }, 100)
        }
    },
    watch: {
        'upload.list': {
            handler(val, oldVal) {
                if(this.upload.timer) {
                    clearTimeout(this.upload.timer)
                }

                this.upload.timer = setTimeout(() => {
                    let list =  this.upload.list.find(item => (item.percentage !== 100)) || []
                    if(!list.length) {
                        this.upload.show = false
                        this.upload.list = []
                    }
                }, 1000)
            },
            deep: true
        }
    }
}
</script>

<style lang="scss" scoped>
.el-container {
    height: 100%;
    box-shadow: 0 2px 12px 0 rgba(0, 0, 0, .1);
    &.webapp {
        .el-aside {
            width: 100px!important;
            li {
                height: 20px!important;
                .pc {
                    display: none;
                }
                .mobile {
                    display: block;
                }
            }
        }
        .el-main {
            width: 100%;
            #chat ul li {
                padding-right: 20%;
                .info {
                    >.image {
                        height: 80px;
                    }
                }
                &.master {
                    padding-left: 20%;
                }
            }
        }
    }
    .el-footer {
        width: 100%;
        background-color: #FFFFFF;
        color: #333;
        text-align: center;
        position: relative;
        #unread {
            position: absolute;
            top: -50px;
            right: 15px;
            height: 30px;
            line-height: 30px;
            font-size: 14px;
            border: 1px solid #EDEDED;
            background-color: #FFFFFF;
            span {
                cursor: pointer;
                i {
                    transform: rotate(90deg);
                    margin: 0 10px;
                }
            }
            button {
                padding-right: 10px;
            }
        }
        .tools {
            height: 30px;
            line-height: 30px;
            display: flex;
            margin-top: 5px;
            i {
                display: inline-block;
                cursor: pointer;
                width: 30px;
                text-align: center;
                font-size: 20px;
                color: #999;
                &:hover {
                    color: rgb(105, 105, 105);
                }
            }
        }
        .input {
            height: 60px;
            textarea {
                display: block;
                width: 100%;
                height: 100%;
                resize: none;
                border: 0 none;
                outline: none;
                font-size: 14px;
                line-height: 18px;
            }
        }
        .el-button {
            float: right;
        }
    }

    .el-aside {
        background-color: #ECEAE8;
        color: #333;
        text-align: center;
        border-right: 1px solid #ddd;
        .pc {
            display: block;
        }
        .mobile {
            display: none;
        }
        .contacts-list {
            padding: 0;
            margin: 0 auto;
            .contacts-list-item {
                list-style-type: none;
                text-align: left;
                padding: 10px;
                height: 40px;
                &.active {
                    background-color: #C6C5C5;
                }
                &:hover:not(.active) {
                    background-color: #e0dfdf;
                }
                .el-avatar--square {
                    border-radius: 0;
                    background-color: #fff;
                    margin: 0 auto;
                    >>>img {
                        display: block;
                        margin: 0 auto;
                    }
                }
                .info {
                    display: flex;
                    height: 20px;
                    line-height: 20px;
                    color: inherit;
                    .name {
                        color: #000;
                        font-size: 14px;
                        overflow: hidden;
                        text-overflow: ellipsis;
                        white-space: nowrap;
                        padding-right: 10px;
                    }
                    .time {
                        flex: 1;
                        color: #999;
                        font-size: 12px;
                        text-align: right;
                    }
                }
                .chat {
                    color: #999;
                    font-size: 13px;
                    overflow: hidden;
                    text-overflow: ellipsis;
                    white-space: nowrap;
                    height: 20px;
                    line-height: 20px;
                    max-width: 200px;
                    img {
                        height: 20px;
                    }
                }
            }
        }
    }
    .el-header {
        background-color: #F5F5F5;
        line-height: 50px;
        display: flex;
        position: relative;
        span {
            flex: 1;
            padding: 0 20px;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
            font-size: 18px;
        }
        i {
            cursor: pointer;
            font-size: 20px;
            margin-right: 10px;
        }
    }
    .el-main {
        background-color: #F5F5F5;
        color: #333;
        text-align: center;
        line-height: 160px;
        overflow-y: scroll;
        border-top: 1px solid #ddd;
        border-bottom: 1px solid #ddd;
        padding: 0 0 0 6px;
        #chat {
            min-height: 100%;
            background-color: #F5F5F5;
            padding: 0;
            ul {
                list-style-type: none;
                padding: 15px 0 0;
                margin: 10px;
                li {
                    display: flex;
                    line-height: 30px;
                    padding-right: 50%;
                    text-align: left;
                    margin-bottom: 20px;
                    position: relative;
                    &.time {
                        margin-top: 30px;
                    }
                    span {
                        width: 36px;
                        border-radius: 0;
                        background-color: #fff;
                        >>>img {
                            margin: 0 auto;
                        }
                    }
                    .info{
                        margin-left: 10px;
                        position: relative;
                        flex: 1;
                        color: inherit;
                        >.time{
                            text-align: left;
                            font-size: 13px;
                            color: #999;
                            line-height: 16px;
                        }
                        >.content {
                            float: left;
                            font-size: 14px;
                            background-color: #fff;
                            padding: 8px 12px;
                            // margin-top: 2px;
                            border-radius: 2px;
                            word-break: break-all;
                            word-wrap: break-word;
                            line-height: 20px;
                            text-align: justify;
                            position: relative;
                            .el-image {
                                display: table-cell;
                                height: inherit;
                                max-width: 100%;
                                border-radius: 2px;
                                >>>img {
                                    width: auto;
                                    max-width: 100%;
                                    object-fit: cover;
                                }
                            }
                            .el-icon-caret-left {
                                position: absolute;
                                left: -6px;
                                top: 5px;
                                font-size: 12px;
                                color: #fff;
                            }
                        }
                        >.transparent{
                            background-color: transparent!important;
                            padding: 0;
                        }
                        >.image {
                            height: 150px;
                            padding: 0;
                        }
                        .read {
                            clear: both;
                            font-size: 11px;
                            float: right;
                            line-height: 28px;
                            color: #909399;
                        }
                    }
                    &.time-format {
                        padding-top: 30px;
                        position: relative;
                        >.time {
                            position: absolute;
                            margin: 0 auto;
                            left: 50%;
                            top: 0;
                            transform: translate(-50%, -50%);
                            text-align: center;
                            background-color: #DADADA;
                            color: #FFFFFF;
                            display: inline-block;
                            font-size: 12px;
                            padding: 1px 6px;
                            height: 20px;
                            line-height: 20px;
                            border-radius: 2px;
                            white-space: nowrap;
                        }
                    }
                    &.master {
                        padding-right: 0;
                        padding-left: 50%;
                        flex-direction: row-reverse;
                        .info {
                            margin-left: 0;
                            margin-right: 10px;
                            >.time{
                                text-align: right;
                            }
                            >.content {
                                background-color: #9EEA6A;
                                float: right;
                                // &:hover {
                                //     .el-icon-error {
                                //         display: inline-block;
                                //     }
                                // }
                                .el-icon-error {
                                    position: absolute;
                                    display: none;
                                    top: -9px;
                                    right: -10px;
                                    cursor: pointer;
                                    color: rgba(144, 147, 153, .5);
                                }
                                .el-icon-caret-right {
                                    position: absolute;
                                    font-size: 12px;
                                    top: 5px;
                                    right: -6px;
                                    color: #9EEA6A;
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

.error {
    >>>.el-dialog {
        width: 300px;
        .main {
            display: table;
            padding: 0;
            i {
                display: table-cell;
                width: 20px;
            }
            span {
                line-height: 20px;
            }
        }
    }
}
.search {
    >>>.el-dialog {
        width: 50%;
        min-width: 300px;
        margin-top: 10vh!important;
    }
    .input {
        margin-bottom: 20px;
    }
    .empty {
        height: 350px;
        line-height: 350px;
        font-size: 18px;
        text-align: center;
        color: #9b9b9b;
    }
    .result {
        text-align: center;
        font-size: 13px;
        color: #9b9b9b;
    }
    .search-list {
        margin: 0;
        padding: 0;
        list-style-type: none;
        height: 350px;
        overflow-y: scroll;
        li {
            border-bottom: 1px solid #f6f5ec;
            padding: 18px 0;
            &:last-child {
                border-bottom: 0 none;
            }
            .el-avatar {
                background: #FFFFFF;
            }
            .name {
                color: #9b9b9b;
                font-size: 12px;
            }
            .content {
                margin-top: 4px;
                text-align: justify;
                .el-image {
                    height: 150px;
                    max-width: 90%;
                    >>>img{
                        object-fit: cover;
                    }
                }
            }
            .time {
                text-align: right;
                color: #9b9b9b;
                font-size: 11px;
            }
        }
    }
}

.emoji {
    width: 320px;
    display: flex;
    flex-wrap: wrap;
    img {
        width: 30px;
        height: 30px;
        padding: 5px;
        border-radius: 5px;
        &:hover{
            background-color: #F5F5F5;
        }
    }
}

.el-row {
    margin-bottom: 20px;
    &:last-child {
        margin-bottom: 0;
    }
    img {
        height: 20px;
    }
}
</style>

<style lang="scss">
.txt {
    display: inline-block;
    vertical-align: middle;
}
.draft {
    color: red;
}
.empty-popover {
    padding: 10px;
    min-width: auto;
}
.quick-reply-popper {
    .popper {
        padding: 0;
        box-shadow: 0 2px 4px rgba(0, 0, 0, .12), 0 0 6px rgba(0, 0, 0, .04);
    }
    table {
        border: 0 none;
    }
    tr:last-child{
        td {
            border-bottom: 0 none;
        }
    }
    .cell {
        text-align: justify;
        &:hover{
            cursor: pointer;
        }
    }
}
.emoji-popper {
    padding: 0!important;
    width: auto;
    height: 200px;
    .popper {
        padding: 0;
        box-shadow: 0 2px 4px rgba(0, 0, 0, .12), 0 0 6px rgba(0, 0, 0, .04);
    }
    .el-tabs {
        border: 0 none;
        .el-tabs__content {
            height: 160px;
            overflow-y: scroll;
            padding: 5px 10px!important;
        }
        .el-tabs__header {
            margin-top: 0!important;
        }
        .iconfont {
            margin-right: 0;
            font-size: 20px;
        }
    }
}
.msg-emoji {
    display: inline-block;
    height: 20px;
    vertical-align: middle;
}
.msg-meme {
    display: block;
    width: 60px;
}
.chat .msg-meme {
    display: inline-block;
    width: 20px;
    vertical-align: middle;
}
.msg-link {
    color: #576B87;
    text-decoration: none;
    &:hover {
        text-decoration: underline;
    }
}
</style>
<template>
    <div class="menu">
        <div class="status">
            <span v-if="online">在</span>
            <span v-if="!online">离</span>
            <img class="comi offline" :src="comiOffline" v-if="!online" @click="previewImg(comiOffline)">
            <img class="comi online" :src="comiOnline" v-if="online" @click="previewImg(comiOnline)">
            <span>线</span>
        </div>
        <el-dropdown trigger="click" v-if="online">
            <el-button type="primary">群<el-icon class="el-icon--right"><arrow-down /></el-icon></el-button>
            <template #dropdown>
                <el-dropdown-menu>
                    <el-dropdown-item v-for="group in groups" @click="chooseGroup(group)">{{ group["name"] }}</el-dropdown-item>
                </el-dropdown-menu>
            </template>
        </el-dropdown>
        <el-button type="success" @click="this.loginDialog = true" v-if="!online">Login</el-button>
        <el-button type="warning" @click="logout" v-if="online">Logout</el-button>
    </div>
    <div class="chat">
        <div class="chatname">{{ chatName }}</div>
        <div class="msgspace" ref="msgspace">
            <message v-for="(data, index) in msgList" :memberName="data.sender.memberName" :qq="data.sender.id" :messageChain="data.messageChain" :isLeft="!data.sender.bot" :key="index" @goDown="goDown" @imgPreview="previewImg"></message>
        </div>
        <div class="content" contenteditable ref="content" @click="handle_click"></div>
        <div class="functions" ref="functions">
            <el-button type="primary" @click="facesShow = !facesShow">表情</el-button>
            <el-dropdown trigger="click">
                <el-button type="primary">艾特<el-icon class="el-icon--right"><arrow-up /></el-icon></el-button>
                <template #dropdown>
                    <el-dropdown-menu>
                        <el-dropdown-item @click="at('all')">全体成员</el-dropdown-item>
                        <el-dropdown-item v-for="member in members" @click="at(member)">{{ member["memberName"] }}</el-dropdown-item>
                    </el-dropdown-menu>
                </template>
            </el-dropdown>
            <input type="file" ref="file" @change="handle_file" multiple="multiple" v-show="false">
            <el-button type="primary" @click="chooseImg">图片</el-button>
            <el-button type="primary" class="send" @click="send">发送</el-button>
            <div class="faces" v-show="facesShow">
                <img v-for="(title, id) in qqFaceMap" :src="`https://www.emojiall.com/img/platform/qq/${id}@2x.gif`"
                    :title="title" @click="chooseFace(id, title)"
                >
            </div>
        </div>
    </div>
    <el-dialog v-model="loginDialog" title="登录后使用conixBot" center draggable>
        <img class="comi" src="https://tvax4.sinaimg.cn/large/007YVyKcly1h2jky1tq57g308e08ekjl.gif">
        <el-input v-model="qq" placeholder="conixBot's qq" />
        <el-input v-model="key" placeholder="conixBot's verify_key" show-password/>
        <template #footer>
            <span class="dialog-footer">
                <el-button type="primary" @click="login">Confirm</el-button>
            </span>
        </template>
    </el-dialog>
    <el-image-viewer @close="this.imageViewer = false" :url-list="imageList" v-if="imageViewer" :hide-on-click-modal="true"/>
</template>

<script>
import { ElButton, ElDialog, ElInput, ElNotification, ElSkeleton, ElDropdown, ElDropdownMenu, ElDropdownItem, ElIcon, ElImageViewer } from 'element-plus'
import { apiBaseURI } from '../config.js'
import { ArrowUp, ArrowDown } from '@element-plus/icons-vue'
import Message from '../components/message.vue'
import { qqFaceMap, qqFaceMapReverse } from '../utils/qqFace.js'
import "element-plus/es/components/button/style/css"
import "element-plus/es/components/dialog/style/css"
import "element-plus/es/components/input/style/css"
import "element-plus/es/components/notification/style/css"
import "element-plus/es/components/skeleton/style/css"
import "element-plus/es/components/dropdown/style/css"
import "element-plus/es/components/image-viewer/style/css"

export default {
    data() {
        return {
            wsURI: "",
            loginDialog: false,
            qq: "",
            key: "",
            ws: null,
            online: false,
            groups: [],
            group: {}, //选择的qq群
            groupMap: new Map(), //群哈希表 键为群id，值为相关信息的对象
            members: [],
            memberMap: new Map(), //哈希表 键为qq，值为相关信息的对象
            chatName: "", //群名或者群昵称
            facesShow: false, //是否显示表情预览框
            msgList: [], //消息列表 包含所有消息的data字段，data中包含messageChain和sender
            newMsg: null, //我们新发的消息，当发送成功后会被加入到msgList中
            imageViewer: false, //是否展现图片预览框
            imageList: [], //用于ImageViewer
            comiOffline: "https://tvax4.sinaimg.cn/large/007YVyKcly1h2jky1tq57g308e08ekjl.gif", //离线的古见
            comiOnline: "https://tva4.sinaimg.cn/large/007YVyKcly1h2kl62sylyj30dp0fkjvq.jpg", //在线的古见
            qqFaceMap: qqFaceMap, //qq表情正向表，id为键，含义为值

        }
    },
    components: { ElButton, ElDialog, ElInput, ElSkeleton, ElDropdown, ElDropdownMenu, ElDropdownItem, ArrowUp, ArrowDown, ElIcon, Message, ElImageViewer },
    mounted() {
        if (localStorage.getItem("token")) { //如果有token则自动发起请求得到wsURI
            this.getWSURI()
        }
        document.body.addEventListener("click", (e) => {
            if (this.facesShow == true) {
                if (!this.$refs.functions.contains(e.target)) { //点击sendspace以外的区域实现自动关闭表情框
                    this.facesShow = false
                } else if (e.target == this.$refs.functions) { //点击除按钮之外的空白区域也可关闭
                    this.facesShow = false
                }
            }
        })
    },
    methods: {
        login() { //登录成功后将获得token
            fetch(`${apiBaseURI}/bot/login`, {
                method: "POST",
                body: new URLSearchParams({qq: this.qq, key: this.key})
            }).then(res => res.json()).then(res => {
                if (res.success) {
                    localStorage.setItem("token", res.token)
                    this.getWSURI() //利用token获取wsURI，之后建立ws连接
                }
            })
        },
        logout() { //切断ws并且删除token
            this.ws.close()
            localStorage.removeItem("token")
        },
        getWSURI() { //jwt鉴权后将得到wsURI
            fetch(`${apiBaseURI}/bot/ws`, {
                headers: { authorization: localStorage.getItem("token") }
            }).then(res => res.json()).then(res => {
                if (res.success) {
                    this.wsURI = res.wsURI
                    this.initWS()
                } else {
                    localStorage.clear("token")
                    ElNotification({ title: 'Error', message: 'token过期 请重新登录', type: 'error' })
                }
            })
        },
        initWS() {
            let socket = new WebSocket(this.wsURI)
            socket.addEventListener("message", this.handle_message)
            socket.addEventListener("close", this.handle_close)
            this.ws = socket
        },
        handle_message(msg) {
            let result = JSON.parse(msg.data)
            console.log(result)
            if (result.syncId == "") { //空表示为连接过程
                if (result.data.code == 0) {
                    this.loginDialog = false
                    this.online = true
                    if (this.qq) {
                        localStorage.setItem("qq", this.qq)
                    }
                    this.groupList() //登录成功后去查询qq群列表
                    this.botProfile() //获得bot的信息
                    if (this.key) {
                        ElNotification({ title: 'Success', message: '登录成功!', type: 'success' })
                    } else {
                        ElNotification({ title: 'Success', message: '自动登录成功!', type: 'success' })
                    }
                } else {
                    ElNotification({ title: 'Error', message: '请重新登录!', type: 'error' })
                    this.logout()
                }
            } else if (result.syncId == "-1") { //-1表示新消息推送
                this.msgList.push(result.data)
            } else if (result.syncId == "1") { //1表示发送群消息
                if (result.data.code == 0) {
                    this.$refs.content.innerText = ""
                    console.log(this.newMsg)
                    this.msgList.push(this.newMsg)
                    ElNotification({ title: 'Success', message: '发送成功!', type: 'success' })
                } else {
                    ElNotification({ title: 'Error', message: '发送失败!', type: 'error' })
                }
            } else if (result.syncId == "2") { //2表示获得群列表
                this.groups = result.data.data
                for (let group of this.groups) {
                    this.groupMap.set(group.id, {name: group.name, members: []})
                    this.memberList(group.id)
                }
            } else if (result.syncId == "3") { //3表示获得群成员列表
                for (let member of result.data.data) { //更新哈希表
                    if (!this.memberMap.has(member.id)) {
                        this.memberMap.set(member.id, {memberName: member.memberName})
                    }
                    this.groupMap.get(member.group.id).members.push(member) //将群成员加入groupMap的members列表中
                }
            } else if (result.syncId == "4") { //4表示获取bot资料
                const memberName = result.data.nickname
                const qq = Number(localStorage.getItem("qq"))
                this.memberMap.set(qq, {memberName})
            }
        },
        handle_close() {
            if (this.online)  {
                this.online = false
                ElNotification({ title: 'Success', message: '下线成功！', type: 'success' })
            }
        },
        sendGroupMessage(content) {
            this.ws.send(JSON.stringify({
                syncId: 1,
                command: "sendGroupMessage", // 命令字
                subCommand: null,
                content
            }))
        },
        groupList() {
            this.ws.send(JSON.stringify({
                syncId: 2,
                command: "groupList", // 命令字
                subCommand: null,
                content: null
            }))
        },
        chooseGroup(group) {
            this.group = group
            this.chatName = group["name"]
            this.members = this.groupMap.get(group.id).members
        },
        memberList(groupId) {
            this.ws.send(JSON.stringify({
                syncId: 3,
                command: "memberList", // 命令字
                subCommand: null,
                content: { target: groupId }
            }))
        },
        botProfile() {
            this.ws.send(JSON.stringify({
                syncId: 4,
                command: "botProfile", // 命令字
                subCommand: null,
                content: {}
            }))
        },
        send() {
            let messageChain = []
            const nodes = this.$refs.content.childNodes
            nodes.forEach(element => {
                if (element.nodeName == "IMG") {
                    const src = element.src
                    if (/^data:image\/jpeg/.test(src)) {
                        messageChain.push({type: "Image", base64: element.src.slice(23)})
                    } else if (/^data:image\/png/.test(src)) {
                        messageChain.push({type: "Image", base64: element.src.slice(22)})
                    } else {
                        if (element.className == "face") { //如果是qq表情，发送Face消息类型
                            messageChain.push({type: "Face", name: element.title})
                        } else {
                            messageChain.push({type: "Image", url: element.src})
                        }
                    }
                } else if (element.nodeName == "A" && element.className == "at") {
                    const qq = element.data
                    messageChain.push({type: "At", target: Number(qq)})
                } else if (element.nodeName == "A" && element.className == "atall") {
                    messageChain.push({type: "AtAll"})
                } else {
                    messageChain.push({type: "Plain", text: element.textContent})
                }
            })
            console.log(messageChain)
            this.sendGroupMessage({ target: this.group["id"], messageChain })
            this.newMsg = {sender: {memberName: "conixBot", bot: true, id: localStorage.getItem("qq")}, messageChain}
        },
        chooseFace(id, title) {
            let faceImg = document.createElement("img")
            faceImg.src = `https://www.emojiall.com/img/platform/qq/${id}@2x.gif`
            if (title) {
                faceImg.classList.add("face")
                faceImg.title = title
            }
            this.$refs.content.appendChild(faceImg)
            this.facesShow = false
        },
        goDown() {
            setTimeout(() => {
                this.$refs.msgspace.scrollTo({
                    top: this.$refs.msgspace.scrollHeight,
                    left: 0,
                    behavior: 'smooth'
                });
            }, 100)
        },
        at(member) {
            const memberName = member.memberName ?? "全体成员"
            const qq = member.id ?? "all"
            let a = document.createElement("a")
            a.textContent = `@${memberName}`
            a.title = `${memberName}(${qq})`
            a.data = qq
            a.href = "https://conix.ml"
            if (member == "all") {
                a.className = "atall"
            } else {
                a.className = "at"
            }
            this.$refs.content.appendChild(a)
        },
        chooseImg() {
            console.log(this.$refs.file)
            this.$refs.file.click()
        },
        handle_file() {
            for (let file of this.$refs.file.files) {
                if (/^image/.test(file.type)) {
                    const reader = new FileReader()
                    reader.readAsDataURL(file)
                    reader.onload = () => {
                        const src = reader.result
                        const img = document.createElement("img")
                        img.src = src
                        this.$refs.content.appendChild(img)
                    }
                }
            }
        },
        previewImg(src) {
            this.imageList = [src]
            this.imageViewer = true
        },
        handle_click(e) { //由于v-html内部无法绑定事件故用父元素来进行 事件代理
            if (e.target.nodeName == "IMG") {
                this.previewImg(e.target.src)
            }
        }
    },
    computed: {
        faceURI() {
            const list = []
            for (let index of Object.keys(qqFaceMap)) {
                list.push(`https://www.emojiall.com/img/platform/qq/${index}@2x.gif`)
            }
            return list
        }
    }
}
</script>

<style lang="scss">
    * {
        padding: 0;
        margin: 0;
    }
    html, body, div#app {
        width: 100%;
        height: 100%;
    }
    div.menu {
        width: 100%;
        height: 48px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        background-color: #337ecc;
        box-sizing: border-box;
        padding: 0px 5%;
        div.status {
            display: flex;
            align-items: center;
            img.comi {
                width: 40px;
                border-radius: 10px;
            }
            span {
                margin: 0 10px;
                color: #FFF;
            }
        }
    }
    .el-overlay-dialog {
        .el-input:nth-of-type(1) {
            margin: 10px 0px;
        }
        @media screen and (min-width: 768px) {
            .el-dialog {
                width: 500px;
            }
        }
        @media screen and (max-width: 768px) {
            .el-dialog {
                width: 80%;
            }
        }
        .el-dialog--center .el-dialog__body {
            display: flex;
            flex-direction: column;
            justify-content: space-around;
            align-items: center;
            padding: 15px;
        }
        img.comi {
            width: 60px;
            border-radius: 10px;
        }
    }
    div.chat {
        width: 80%;
        margin: auto;
        height: calc(100% - 48px);
        div.chatname {
            box-sizing: border-box;
            width: 100%;
            height: 48px;
            display: flex;
            justify-content: center;
            align-content: center;
            line-height: 48px;
            overflow: hidden;
            color: #337ecc;
            border: 2px #337ecc solid;
            border-top: none;
        }
        div.msgspace {
            box-sizing: border-box;
            width: 100%;
            height: calc(75% - 48px);
            border: 2px #337ecc solid;
            border-top: none;
            border-bottom: none;
            display: flex;
            flex-direction: column;
            overflow: scroll;
            padding-bottom: 10px;
            overflow-x: hidden;
            overflow-y: auto;
        }
        div.content {
            box-sizing: border-box;
            width: 100%;
            height: calc(25% - 32px);
            border: 2px #337ecc solid;
            overflow: scroll;
            padding: 10px;
            overflow-x: hidden;
            overflow-y: auto;
            img {
                width: 60px;
            }
        }
        a {
            text-decoration: none;
            margin: 5px;
            color: #409EFF;
        }
        div.functions {
            height: 32px;
            display: flex;
            justify-content: space-between;
            position: relative;
            @media screen and (min-width: 420px) {
                div.faces {
                    position: absolute;
                    left: 0;
                    bottom: 32px;
                    background-color: rgba(#337ecc, 0.2);
                    width: 337px;
                    height: 320px;
                    display: flex;
                    flex-wrap: wrap;
                    overflow: scroll;
                    overflow-x: auto;
                    img {
                        width: 32px;
                        height: 32px;
                        flex: 0 0 10%;
                    }
                }
            }
            @media screen and (max-width: 420px) {
                div.faces {
                    position: absolute;
                    left: 0;
                    bottom: 32px;
                    background-color: rgba(#337ecc, 0.2);
                    width: 160px;
                    height: 160px;
                    display: flex;
                    flex-wrap: wrap;
                    overflow: scroll;
                    overflow-x: auto;
                    img {
                        width: 32px;
                        height: 32px;
                        flex: 0 0 20%;
                    }
                }
            }
        }
    }
</style>


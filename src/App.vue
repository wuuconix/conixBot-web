<template>
    <div class="menu">
        <div class="status">
            <span v-if="online">在</span>
            <span v-if="!online">离</span>
            <img class="comi offline" src="https://tvax4.sinaimg.cn/large/007YVyKcly1h2jky1tq57g308e08ekjl.gif" v-if="!online">
            <img class="comi online" src="https://tva4.sinaimg.cn/large/007YVyKcly1h2kl62sylyj30dp0fkjvq.jpg" v-if="online">
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
        <el-dropdown trigger="click" v-if="online">
            <el-button type="primary">好友<el-icon class="el-icon--right"><arrow-down /></el-icon></el-button>
            <template #dropdown>
                <el-dropdown-menu>
                    <el-dropdown-item v-for="member in members" @click="chooseMember(member)">{{ member["memberName"] }}</el-dropdown-item>
                </el-dropdown-menu>
            </template>
        </el-dropdown>
        <el-button type="success" @click="this.loginDialog = true" v-if="!online">Login</el-button>
        <el-button type="warning" @click="logout" v-if="online">Logout</el-button>
    </div>
    <div class="chat">
        <div class="chatname">{{ chatName }}</div>
        <div class="msgspace"></div>
        <div class="sendspace" ref="sendspace">
            <div class="content" contenteditable ref="content"></div>
            <el-button type="primary" @click="facesShow = !facesShow">表情</el-button>
            <el-button type="primary" class="send" @click="send">发送</el-button>
            <div class="faces" v-show="facesShow">
                <img v-for="uri in faceURI" :src="uri" @click="chooseFace(uri)">
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
</template>

<script>
import { ElButton, ElDialog, ElInput, ElNotification, ElSkeleton, ElDropdown, ElDropdownMenu, ElDropdownItem, ElIcon } from 'element-plus'
import { apiBaseURI } from '../config.js'
import { ArrowDown } from '@element-plus/icons-vue'
import "element-plus/es/components/button/style/css"
import "element-plus/es/components/dialog/style/css"
import "element-plus/es/components/input/style/css"
import "element-plus/es/components/notification/style/css"
import "element-plus/es/components/skeleton/style/css"
import "element-plus/es/components/dropdown/style/css"

export default {
    data() {
        return {
            wsURI: "",
            loginDialog: false,
            qq: "",
            key: "",
            ws: null,
            online: false,
            action: "listen",
            groups: [],
            group: {}, //选择的qq群
            members: [],
            merber: {}, //选择的qq群成员
            chatType: "group", //聊天类型
            chatName: "", //群名或者群昵称
            facesShow: false //是否显示表情预览框
        }
    },
    components: { ElButton, ElDialog, ElInput, ElSkeleton, ElDropdown, ElDropdownMenu, ElDropdownItem, ArrowDown, ElIcon },
    mounted() {
        if (localStorage.getItem("token")) { //如果有token则自动发起请求得到wsURI
            this.getWSURI()
        }
        document.body.addEventListener("click", (e) => {
            console.log(e.target)
            if (this.facesShow) {
                if (!this.$refs.sendspace.contains(e.target)) { //点击sendspace以外的区域实现自动关闭表情框
                    this.facesShow = false
                } else if (e.target == this.$refs.content) { //点击输入框也可以关闭
                    this.facesShow = false
                } else if (e.target == this.$refs.sendspace) { //点击除按钮之外的空白区域也可关闭
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
            this.action = "login"
            let socket = new WebSocket(this.wsURI)
            socket.addEventListener("message", this.handle_message)
            socket.addEventListener("close", this.handle_close)
            this.ws = socket
        },
        handle_message(msg) {
            let result = JSON.parse(msg.data)
            console.log(result)
            if (result.syncId == "-1") { //新消息
                console.log("以上是新消息")
            } else { //我们请求的回显
                if (this.action == "login") {
                    if (result.data.code == 0) {
                        this.loginDialog = false
                        this.online = true
                        this.groupList() //登录成功后去查询qq群列表
                        ElNotification({ title: 'Success', message: '登录成功!', type: 'success' })
                    } else {
                        ElNotification({ title: 'Error', message: '登录失败!', type: 'error' })
                    }
                } else if (this.action == "sendGroupMessage" || this.action == "sendTempMessage") {
                    if (result.data.code == 0) {
                        this.$refs.content.innerText = ""
                        ElNotification({ title: 'Success', message: '发送成功!', type: 'success' })
                    } else {
                        ElNotification({ title: 'Error', message: '发送失败!', type: 'error' })
                    }
                } else if (this.action == "groupList") {
                    if (result.data.code == 0) {
                        this.groups = result.data.data
                        this.action = "listen"
                    }
                } else if (this.action == "memberList") {
                    if (result.data.code == 0) {
                        this.members = result.data.data
                        this.action = "listen"
                    }
                }
            }

        },
        handle_close() {
            this.online = false
            ElNotification({ title: 'Success', message: '下线成功！', type: 'success' })
        },
        sendGroupMessage(content) {
            this.action = "sendGroupMessage"
            this.ws.send(JSON.stringify({
                syncId: 114514,
                command: "sendGroupMessage", // 命令字
                subCommand: null,
                content
            }))
        },
        sendTempMessage(content) {
            this.action = "sendTempMessage"
            this.ws.send(JSON.stringify({
                syncId: 114514,
                command: "sendTempMessage", // 命令字
                subCommand: null,
                content
            }))
        },
        groupList() {
            this.action = "groupList"
            this.ws.send(JSON.stringify({
                syncId: 114514,
                command: "groupList", // 命令字
                subCommand: null,
                content: null
            }))
        },
        chooseGroup(group) {
            this.chatType = "group"
            this.group = group
            this.chatName = group["name"]
            this.memberList()
        },
        chooseMember(member) {
            this.chatType = "member"
            this.member = member
            this.chatName = member["memberName"]
        },
        memberList() {
            this.action = "memberList"
            this.ws.send(JSON.stringify({
                syncId: 114514,
                command: "memberList", // 命令字
                subCommand: null,
                content: { target: this.group["id"] }
            }))
        },
        send() {
            let messageChain = []
            const nodes = this.$refs.content.childNodes
            nodes.forEach(element => {
                if (element.nodeName == "IMG") {
                    const src = element.src
                    if (/^data/.test(src)) {
                        messageChain.push({type: "Image", base64: element.src.slice(22)})
                    } else {
                        messageChain.push({type: "Image", url: element.src})
                    }
                } else {
                    messageChain.push({type: "Plain", text: element.textContent})
                }
            })
            console.log(messageChain)
            if (this.chatType == "group") {
                this.sendGroupMessage({ target: this.group["id"], messageChain })
            } else {
                this.sendTempMessage({ qq: this.member["id"], group: this.group["id"], messageChain })
            }
        },
        chooseFace(uri) {
            let faceImg = document.createElement("img")
            faceImg.src = uri
            this.$refs.content.appendChild(faceImg)
            this.facesShow = false
        }
    },
    computed: {
        faceURI() {
            const list = []
            for (let i = 1; i <= 224; i++) {
                if ([115, 129, 130, 141].includes(i)) { //这些表情无了
                    continue
                }
                let id = String(i)
                if (id.length == 1) {
                    id = "00" + id
                } else if (id.length == 2) {
                    id = "0" + id
                }
                list.push(`https://www.emojiall.com/img/platform/qq/${id}@2x.gif`)
            }
            return list
        }
    }
}
</script>

<style>
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
        height: 64px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        background-color: #337ecc;
        box-sizing: border-box;
        padding: 0px 5%;
    }
    img.github {
        width: 50px;
        border-radius: 25px;
    }
    img.comi {
        width: 55px;
        border-radius: 10px;
    }
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
    div.status {
        display: flex;
        align-items: center;
    }
    div.status span {
        margin: 0 10px;
        color: #FFF;
    }
    div.chat {
        width: 80%;
        margin: auto;
        height: calc(100% - 64px);
    }
    div.chatname {
        box-sizing: border-box;
        width: 100%;
        height: 64px;
        display: flex;
        justify-content: center;
        align-content: center;
        line-height: 64px;
        overflow: hidden;
        color: #337ecc;
        border: 2px #337ecc solid;
        border-top: none;
    }
    div.msgspace {
        box-sizing: border-box;
        width: 100%;
        height: calc(75% - 64px);
        border: 2px #337ecc solid;
        border-top: none;
        border-bottom: none;
    }
    div.sendspace {
        width: 100%;
        height: 25%;
        position: relative;
    }
    .el-button.send {
        position: absolute;
        right: 0;
        bottom: 0;
    }
    div.content {
        box-sizing: border-box;
        width: 100%;
        height: calc(100% - 32px);
        border: 2px #337ecc solid;
        overflow: scroll;
    }
    div.content img {
        width: 60px;
    }

    @media screen and (min-width: 420px) {
        div.faces {
            position: absolute;
            left: 0;
            bottom: calc(25vh - 14px);
            background-color: #337ecc;
            width: 337px;
            height: 320px;
            display: flex;
            flex-wrap: wrap;
            overflow: scroll;
        }
        div.faces img {
            width: 32px;
            height: 32px;
            flex: 0 0 10%;
        }
    }
    @media screen and (max-width: 420px) {
        div.faces {
            position: absolute;
            left: 0;
            bottom: calc(25vh - 14px);
            background-color: #337ecc;
            width: 160px;
            height: 160px;
            display: flex;
            flex-wrap: wrap;
            overflow: scroll;
        }
        div.faces img {
            width: 32px;
            height: 32px;
            flex: 0 0 20%;
        }
    }

</style>

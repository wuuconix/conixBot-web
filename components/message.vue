<template>
    <div :class="{aname: true, left: isLeft, right: !isLeft}">
        <img class="avatar" :src="`https://q1.qlogo.cn/g?b=qq&nk=${qq}&s=640`" @load="$emit('goDown')">
        <span>{{ memberName }}</span>
    </div>
    <div class="msgWrapper" :class="{msgWrapper: true, left: isLeft, right: !isLeft}">
        <div class="msg" v-html="msgHTML" @click="handle_click"></div>
    </div>
</template>

<script>
import { qqFaceMapReverse } from '../utils/qqFace.js'
export default {
    props: ["memberName", "messageChain", "isLeft", "qq"],
    emits: ["goDown", "imgPreview"],
    computed: {
        msgHTML() {
            let div = document.createElement("div")
            for (let i = 0; i < this.messageChain.length; i++) {
                const type = this.messageChain[i].type
                if (type == "Plain") {
                    div.appendChild(document.createTextNode(this.messageChain[i].text))
                } else if (type == "Image") {
                    let img = document.createElement("img")
                    if (this.messageChain[i].base64) { //机器人发送图片是base64格式的，首先检测它
                        img.src = `data:image/png;base64,${this.messageChain[i].base64}`
                    } else {
                        img.src = this.messageChain[i].url
                    }
                    img.classList.add("msg")
                    div.appendChild(img)
                } else if (type == "At") {
                    const qq = this.messageChain[i].target
                    console.log(this.$parent.memberMap)
                    const memberName = this.$parent.memberMap.get(qq).memberName
                    let a = document.createElement("a")
                    a.textContent = `@${memberName}`
                    a.title = `${memberName}(${qq})`
                    a.data = qq
                    a.href = "javascript:void(0);"
                    a.className = "at"
                    div.appendChild(a)
                } else if (type == "AtAll") {
                    let a = document.createElement("a")
                    a.textContent = `@全体成员`
                    a.title = `@全体成员(all)`
                    a.href = "javascript:void(0);"
                    a.className = "atall"
                    div.appendChild(a)
                } else if (type == "Face") {
                    let faceImg = document.createElement("img")
                    const name = this.messageChain[i].name
                    const id = qqFaceMapReverse[name]
                    faceImg.src = `https://www.emojiall.com/img/platform/qq/${id}@2x.gif`
                    faceImg.title = name
                    faceImg.className = "face"
                    div.appendChild(faceImg)
                }
            }
            return div.innerHTML
        }
    },
    methods: {
        handle_click(e) { //由于v-html内部无法绑定事件故用父元素来进行 事件代理
            if (e.target.nodeName == "IMG") {
                this.$parent.previewImg(e.target.src)
            }
        }
    },
}
</script>

<style scoped>
    div.aname {
        display: flex;
        align-items: center;
        justify-content: flex-start;
        margin: 5px;
    }
    div.aname.right {
        flex-direction: row-reverse;
    }
    img.avatar {
        width: 32px;
        height: 32px;
        border-radius: 16px;
    }
    div.aname.right img.avatar {
        margin-left: 10px;
    }
    div.aname.left img.avatar {
        margin-right: 10px;
    }
    div.msgWrapper {
        display: flex;
        align-content: center;
        justify-content: flex-start;
    }
    div.msgWrapper.left {
        margin-left: 40px;
    }
    div.msgWrapper.right {
        flex-direction: row-reverse;
        margin-right: 40px;
    }
    div.msg {
        background-color:#d1edc4;
        padding: 10px;
        border-radius: 10px;
        width: fit-content;
    }
</style>

<style>
    img.msg {
        max-width: 200px;
    }
</style>
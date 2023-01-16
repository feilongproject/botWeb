<template>
    <div class="sendMsg">
        <table>
            <tr>
                <th>功能</th>
                <th>数据输入</th>
                <th>选择操作对象</th>
                <th>开始操作</th>
            </tr>
            <tr>
                <th>服务器图片管理</th>
                <th>
                    <label for="uploadInput">
                        <input style="padding: 2.5px" id="uploadInput" @change="changeFile" type="file" />
                    </label>
                    <img :src="imageInfo.src" style="max-height: 400px; max-width: 400px" alt="未成功显示图片, 可能是: 未选择/未成功加载" />
                </th>
                <th>
                    <label for="ImageSelect">
                        <select id="imageSelect" style="width: 50%" v-model="imageInfo.selectd" @change="selectImage" required>
                            <option key="NONE" value="NONE">未选择</option>
                            <option v-for="(options, id) in imageInfo.all" :key="id" :value="options.id">{{ options.id }}</option>
                        </select>
                    </label>
                </th>
                <th>
                    <button @click="uploadImage">上传图片</button>
                    <button @click="deleteImage">删除图片</button>
                </th>
            </tr>
            <tr>
                <th><label for="sendMsgContent">向子频道发送消息</label></th>
                <th>
                    <textarea id="sendMsgContent" name="sendMsgContent" class="sendMsgContent" v-model="sendMsgContent"></textarea>
                </th>
                <th style="width: 30%">
                    <label for="channelSelect">
                        <select id="channelSelect" style="width: 50%" v-model="channelInfo.selectd" @change="selectChannel" required>
                            <option key="NONE" value="NONE">未选择</option>
                            <option v-for="(options, id) in channelInfo.all" :key="id" :value="options.id">{{ options.name }}</option>
                        </select>
                    </label>
                </th>
                <th>
                    <button @click="sendMsg(true, false)">发送文字</button>
                    <button @click="sendMsg(true, true)">发送图片&文字</button>
                    <button @click="sendMsg(false, true)">发送图片</button>
                    <hr />
                    <div>发送状态</div>
                    <div :style="sended.class">{{ sended.info }}</div>
                </th>
            </tr>
            <tr>
                <th><label for="keywordContent">关键词查询</label></th>
                <th>
                    <textarea id="keywordContent" name="keywordContent" class="sendMsgContent" v-model="keywords.selectd.content" v-bind:readonly="keywords.readonly"></textarea>
                </th>
                <th>
                    <label for="keywordSelect">
                        <select id="keywordSelect" style="width: 50%" v-model="keywords.selectd.index" @change="selectKeyword" required>
                            <option key="NONE" value="0" v-if="keywords.selectd.keyword == 'NONE'">未选择</option>
                            <option v-for="(options, id) in keywords.all" :key="id" :value="id">{{ options.keyword }}</option>
                        </select>
                        <div v-if="keywords.selectd.keyword != 'NONE'">
                            <!-- <span>{{ keywords.selectd }}</span> -->
                            <div>类型: {{ keywords.selectd.type == "accurate" ? "精确" : "模糊" }}</div>
                            <div>状态: {{ keywords.selectd.status == "checked" ? "启用" : "禁用" }}</div>
                            <div>创建者: {{ keywords.selectd.ownerName }}</div>
                        </div>
                    </label>
                </th>
                <th>
                    <button @click="keywordExec.changeStatus">{{ keywords.selectd.status == "checked" ? "禁用" : "启用" }}</button>
                    <button @click="keywordExec.changeType">更改为{{ keywords.selectd.type == "accurate" ? "模糊" : "精确" }}类型</button>
                    <button @click="keywordExec.saveImage">{{ keywords.selectd.imageName == "NONE" ? "添加" : "移除" }}当前图片</button>
                    <button @click="keywordExec.saveContent">保存回复内容</button>
                    <button @click="keywordExec.delete">删除</button>
                    <hr />
                    <div>状态</div>
                    <div :style="keywordExec.status.value.class">{{ keywordExec.status.value.info }}</div>
                </th>
            </tr>
        </table>
    </div>
    <hr style="height: 10px; margin: 40px 0px; background-color: #0af" />
    <div class="botStatus">
        <div class="botStatusInfo">
            <div>bot状态</div>
            <div v-for="(status, id) in botStatus" class="botStatusInfo" :style="`border: solid 3px #` + status.classType">{{ status.msg }}</div>
        </div>
        <div class="botServer">
            <span>bot服务器连接信息</span>
            <label for="packageVersion">
                网页版本:
                <input id="webVersion" style="width: 50px" :value="webVersion" readonly />
                后端版本
                <input id="serverVersion" style="width: 50px" :value="serverVersion" readonly />
            </label>
            <label class="botServerWsType">
                连接类型
                <span class="input"> <input id="botServerWsTypeWs" v-model="botServer.wsType" type="radio" value="ws" /> ws <input id="botServerWsTypeWss" v-model="botServer.wsType" type="radio" value="wss" />wss </span>
            </label>
            <label for="botServerInputIp">地址<input id="botServerInputIp" type="text" v-model="botServer.ip" /></label>
            <label for="botServerInputPort">端口<input id="botServerInputPort" type="text" v-model="botServer.port" /></label>
            <label for="botServerInputToken">Token<input id="botServerInputToken" type="text" v-model="botServer.token" /></label>
            <div class="buttons">
                <button @click="saveServer">保存服务器设置</button>
                <button @click="closeServer">关闭连接</button>
                <button @click="refreshData">刷新数据</button>
                <button @click="changeReconnent">{{ botServer.reConnent ? "关闭" : "开启" }}自动重连</button>
            </div>
        </div>
    </div>
    <!-- <Notice ref="noticeShow" title="发送状态" delay="">
        {{ sendedInfo }}
    </Notice> -->
</template>

<script setup lang="ts">
import path from "path-browserify";
import { ref, watch } from "vue";
//import Notice from "./Notice.vue";
import { version as webVersion } from "../../package.json";

const noticeShow = ref();
const serverVersion = ref("未知");
const botStatus = ref([{ msg: "正在建立连接", classType: "0af" }]);
const botServer = ref({
    token: localStorage.getItem("botServerToken") || "",
    port: localStorage.getItem("botServerPort") || "8848",
    ip: localStorage.getItem("botServerIp") || "127.0.0.1",
    wsType: localStorage.getItem("botServerWsType") || "ws",
    reConnent: localStorage.getItem("reConnent") == "false" ? false : true,
});
const channelInfo = ref({
    selectd: "NONE",
    all: [{ id: "NONE", name: "未选择", selected: true, type: 0 }],
});
channelInfo.value.all = [];
const imageInfo = ref({
    src: "",
    image: null as null | File,
    selectd: "NONE",
    all: [{ id: "NONE", selected: true }],
});
imageInfo.value.all = [];
const sendMsgContent = ref("");
const sended = ref({
    info: "未发送",
    class: "border: solid 3px #0af",
});
const keywords = ref({
    readonly: true,
    selectd: { index: 0, keyword: "NONE", content: "", ownerId: "", ownerName: "", refOwnerId: "", refOwnerName: "", status: "", type: "", imageName: "NONE" },
    all: [{ keyword: "未选择", content: "", ownerId: "", ownerName: "", refOwnerId: "", refOwnerName: "", status: "", type: "", imageName: "" }],
});
keywords.value.all = [];
const keywordExec = {
    changeStatus: () => {
        if (keywords.value.selectd.keyword == "NONE") return (keywordExec.status.value = { info: "未选择数据, 无法通过", class: "border: solid 3px #f00" });
        wsSend({ key: "keyword.changeStatus", data: keywords.value.selectd }, "keyword.changeStatus");
    },
    changeType: () => {
        if (keywords.value.selectd.keyword == "NONE") return (keywordExec.status.value = { info: "未选择数据, 无法更改", class: "border: solid 3px #f00" });
        wsSend({ key: "keyword.changeType", data: keywords.value.selectd }, "keyword.changeType");
    },
    saveContent: () => {
        if (keywords.value.selectd.keyword == "NONE") return (keywordExec.status.value = { info: "未选择数据, 无法保存", class: "border: solid 3px #f00" });
        wsSend({ key: "keyword.saveContent", data: keywords.value.selectd }, "keyword.saveContent");
    },
    saveImage: () => {
        if (keywords.value.selectd.keyword == "NONE") return (keywordExec.status.value = { info: "未选择数据, 无法操作", class: "border: solid 3px #f00" });
        keywords.value.selectd.imageName = keywords.value.selectd.imageName == "NONE" ? imageInfo.value.selectd : "NONE";
        wsSend({ key: "keyword.saveImage", data: keywords.value.selectd }, "keyword.saveImage");
    },
    delete: () => {
        if (keywords.value.selectd.keyword == "NONE") return (keywordExec.status.value = { info: "未选择数据, 无法删除", class: "border: solid 3px #f00" });
        wsSend({ key: "keyword.delete", data: keywords.value.selectd }, "keyword.delete");
    },
    status: ref({
        info: "未操作",
        class: "border: solid 3px #0af",
    }),
};
//var intervalId: NodeJS.Timer | number | null = null;
var ws: WebSocket | null = null;
const wsIntentMessage: { [key: string]: (data?: any) => void } = {
    ok: () => {
        intervalHandler();
        // if (intervalId) clearInterval(intervalId);
        // intervalHandler();
        // intervalId = setInterval(intervalHandler, 10 * 1000);
    },
    error: (err) => {
        botStatus.value.push({ msg: `Error: ${err}`, classType: "f00" });
    },
    version: (data) => {
        serverVersion.value = data.serverVersion;
    },
    "channel.getList": (data: SaveGuild[]) => {
        //console.log(`接收到信息:`, data);
        channelInfo.value.all = [];
        for (const guild of data) {
            for (const channel of guild.channel) {
                if (![0, 2, 10005].includes(channel.type)) continue;
                channelInfo.value.all.push({
                    ...channel,
                    name: `${guild.name} - ${channel.name}`,
                    selected: false,
                });
            }
        }
    },
    "channel.recMsg": (data) => {
        //console.log("channel.recMsg", data);
        sended.value = { info: `发送成功`, class: "border: solid 3px #0af" };
    },
    "keyword.get": (_keywords) => {
        keywords.value.all = _keywords;
    },
    "keyword.changeStatus": (_keyword) => {
        keywords.value.all[_keyword.index] = _keyword;
        keywords.value.selectd = _keyword;
        keywordExec.status.value = { info: `已${keywords.value.selectd.status == "checked" ? "启用" : "禁用"}`, class: "border: solid 3px #0af" };
    },
    "keyword.changeType": (_keyword) => {
        keywords.value.all[_keyword.index] = _keyword;
        keywords.value.selectd = _keyword;
        keywordExec.status.value = { info: `已更改类型为: ${keywords.value.selectd.type == "accurate" ? "精确" : "模糊"}`, class: "border: solid 3px #0af" };
    },
    "keyword.saveContent": (_keyword) => {
        keywordExec.status.value = { info: `已保存${_keyword.type == "accurate" ? "精确" : "模糊"}关键词"${_keyword.keyword}"的内容`, class: "border: solid 3px #0af" };
    },
    "keyword.saveImage": (data) => {
        if (data.imageName == "NONE") {
            imageInfo.value.selectd = "NONE";
            imageInfo.value.src = "";
        }
        wsSend({ key: "keyword.get" }, "keyword.get");
    },
    "keyword.delete": async (_keyword) => {
        if (!_keyword.isDel) return;
        keywords.value.readonly = true;
        keywordExec.status.value = { info: `已删除${_keyword.type == "accurate" ? "精确" : "模糊"}关键词"${_keyword.keyword}"`, class: "border: solid 3px #0af" };
        intervalHandler();
        keywords.value.selectd = { index: 0, keyword: "NONE", content: "", ownerId: "", ownerName: "", refOwnerId: "", refOwnerName: "", status: "", type: "", imageName: "NONE" };
    },
    "image.getList": async (data) => {
        imageInfo.value.all = [];
        imageInfo.value.selectd = "NONE";
        for (const imageName of data) {
            imageInfo.value.all.push({ id: imageName, selected: false });
        }
    },
    "image.sendReady": async () => {
        if (imageInfo.value.image) wsSend(await imageInfo.value.image.arrayBuffer(), "image.sendGone");
    },
    "image.sendGone": async (data) => {
        botStatus.value.push({ msg: `于${new Date().toLocaleString()}成功上传图片`, classType: "0f0" });
        imageInfo.value.selectd = data.name;
        imageInfo.value.all.push({ id: data.name, selected: false });
        //wsSend({ key: "image.getList" }, "image.getList");
    },
    "image.delete": async (data) => {
        imageInfo.value.selectd = "NONE";
        imageInfo.value.image = null;
        imageInfo.value.src = "";
        wsSend({ key: "image.getList" }, "image.getList");
    },
};
watch(botStatus.value, () => {
    while (botStatus.value.length > 10) botStatus.value.shift();
});

function intervalHandler() {
    //if (!botServer.value.autoFetch) return;
    wsSend({ key: "keyword.get" }, "keyword.get");
    wsSend({ key: "image.getList" }, "image.getList");
    wsSend({ key: "channel.getList" }, "channel.getList");
}
function wsSend(content: { key: string; retKey?: string; data?: {} } | ArrayBuffer, desc: string) {
    const type = content.toString();
    if (content instanceof ArrayBuffer) {
        botStatus.value.push({ msg: `于${new Date().toLocaleString()}发送二进制图片: ${desc}`, classType: "0af" });
        return ws!.send(content);
    } else if (content instanceof Object) {
        botStatus.value.push({ msg: `于${new Date().toLocaleString()}发送数据: ${desc}`, classType: "0af" });
        ws!.send(JSON.stringify(content));
    }
}

function saveServer(e: MouseEvent) {
    localStorage.setItem("botServerIp", botServer.value.ip);
    localStorage.setItem("botServerPort", botServer.value.port);
    localStorage.setItem("botServerToken", botServer.value.token);
    localStorage.setItem("botServerWsType", botServer.value.wsType);
    //localStorage.setItem("autoFetch", `${botServer.value.autoFetch}`);
    localStorage.setItem("reConnent", `${botServer.value.reConnent}`);
}
function closeServer(e: MouseEvent) {
    ws!.close(1000, "客户端主动关闭");
}
function refreshData(e: MouseEvent) {
    intervalHandler();
}
function changeReconnent() {
    botServer.value.reConnent = !botServer.value.reConnent;
    botStatus.value.push({ msg: `已${botServer.value.reConnent ? "开启" : "关闭"}自动重连`, classType: "00f" });
    if (botServer.value.reConnent) init();
}

function selectChannel(e: Event) {
    //console.log(e, channelInfo.value);
}
function selectKeyword(e: Event) {
    //console.log((e.target as any).options);
    keywords.value.readonly = false;
    const _index = (e.target as any).selectedIndex;
    const index = keywords.value.selectd.content == "" ? _index - 1 : _index;
    keywords.value.selectd = Object.assign({ index }, keywords.value.all[index]);
    if (keywords.value.selectd.imageName == "NONE" || !keywords.value.selectd.imageName) {
        imageInfo.value.selectd = "NONE";
        imageInfo.value.image = null;
        imageInfo.value.src = "";
    } else {
        imageInfo.value.selectd = keywords.value.selectd.imageName;
        wsSend({ key: "image.get", data: keywords.value.selectd.imageName }, "image.get");
    }
}
function selectImage(e: Event) {
    if (imageInfo.value.selectd == "NONE") {
        imageInfo.value.image = null;
        imageInfo.value.src = "";
    } else {
        wsSend({ key: "image.get", data: imageInfo.value.selectd }, "image.get");
    }
}
function changeFile(e: Event) {
    const file = (e.target as any).files[0] as File;
    if (file && file.size) {
        imageInfo.value.image = file;
        imageInfo.value.src = getObjectURL() || "";
    } else imageInfo.value.image = null;
}
function uploadImage(e: Event) {
    if (!imageInfo.value.image || !imageInfo.value.image.size || !imageInfo.value.image.name || !imageInfo.value.image.type) {
        return botStatus.value.push({ msg: `于${new Date().toLocaleString()}发送图片失败, 请检查是否选中`, classType: "f00" });
    }
    const _pathname = path.parse(imageInfo.value.image.name);
    const sendInfo = {
        name: `${_pathname.name}-${new Date().getTime() + _pathname.ext}`,
        size: imageInfo.value.image.size,
        type: imageInfo.value.image.type,
        lastModified: imageInfo.value.image.lastModified,
    };
    wsSend({ key: "image.sendReady", data: sendInfo }, "image.sendReady");
}
function deleteImage(e: Event) {
    if (imageInfo.value.selectd == "NONE") {
        return botStatus.value.push({ msg: `于${new Date().toLocaleString()}删除图片失败, 请检查是否选中`, classType: "f00" });
    } else wsSend({ key: "image.delete", data: imageInfo.value.selectd }, "image.delete");
}
function getObjectURL(data?: Blob | null): string {
    data = data || imageInfo.value.image;
    if (!data) {
        return "";
    } else if (window.URL) {
        return window.URL.createObjectURL(data);
    } else if (window.webkitURL) {
        return window.webkitURL.createObjectURL(data);
    } else return "";
}

function sendMsg(withContent: boolean, withImage: boolean) {
    // console.log(noticeShow.value);
    console.log(`发送至: ${channelInfo.value.selectd}, 发送内容: ${sendMsgContent.value}`);
    if (channelInfo.value.selectd == "NONE") {
        sended.value = { info: "错误: 未指定发送频道", class: "border: solid 3px #f00" };
    } else if (withContent && !sendMsgContent.value) {
        sended.value = { info: "错误: 发送内容为空", class: "border: solid 3px #f00" };
    } else if (withImage && !imageInfo.value.selectd) {
        sended.value = { info: "错误: 发送图片为空", class: "border: solid 3px #f00" };
    } else {
        sended.value = { info: "发送中", class: "border: solid 3px #00f" };
        wsSend(
            {
                key: "channel.postMsg",
                retKey: "channel.recMsg",
                data: {
                    channelId: channelInfo.value.selectd,
                    content: withContent ? sendMsgContent.value : undefined,
                    imageName: withImage ? imageInfo.value.selectd : undefined,
                },
            },
            "channel.postMsg"
        );
    }

    // noticeShow.value.noticeShow = !noticeShow.value.noticeShow;
}

init();
function init() {
    ws = new WebSocket(`${botServer.value.wsType}://${botServer.value.ip}:${botServer.value.port}`, [`${webVersion}|${botServer.value.token}`]);

    ws.onopen = () => {
        console.log("连接开启，状态：", ws!.readyState);
        botStatus.value.push({ msg: `连接开启`, classType: "0af" });
    };
    ws.onmessage = (e) => {
        if (e.data instanceof Blob) {
            botStatus.value.push({
                msg: `于${new Date().toLocaleString()}接收二进制图片`,
                classType: "0f0",
            });
            imageInfo.value.src = getObjectURL(e.data);
        } else {
            const data = JSON.parse(e.data);
            botStatus.value.push({
                msg: `于${new Date().toLocaleString()}接收数据: ${data.key}`,
                classType: "0f0",
            });
            console.log(`接收数据:`, data);
            wsIntentMessage[data.key](data.data);
        }
    };
    ws.onclose = (e) => {
        // if (intervalId) {
        //     clearInterval(intervalId);
        //     intervalId = null;
        // }
        serverVersion.value = "未知";
        const closeInfo = `${e.reason ? e.reason : "未知原因"} | 连接被关闭, 正在重新连接`;
        console.log(closeInfo);
        botStatus.value.push({ msg: closeInfo, classType: "f00" });
        (function aaa(times: number) {
            setTimeout(() => {
                if (!botServer.value.reConnent) return;
                console.log(`重新连接中, 等待${times}s, 服务器状态: ${ws!.readyState}`);
                botStatus.value.push({
                    msg: closeInfo + `\n重新连接中, 等待${times}s`,
                    classType: "f00",
                });
                if (times) aaa(times - 1);
                else init();
            }, 1000);
        })(5);
    };
    /* ws.onerror = (e) => {
    console.log("发生错误，状态:" + ws.readyState, e);
    botStatus.value = "连接发生错误";
    botError.value = "连接发生错误";
}; */
}

interface SaveGuild {
    name: string;
    id: string;
    channel: SaveChannel[];
}

interface SaveChannel {
    name: string;
    id: string;
    type: number;
}
</script>

<style lang="scss" scoped>
.botStatus {
    white-space: pre-line;
    align-items: flex-start;
}

.botStatusInfo {
    display: flex;
    width: 70%;
    flex-direction: column;
    justify-content: space-around;
    align-items: center;
}

.botStatus,
.sendMsg {
    display: flex;
    justify-content: space-evenly;
    textarea:read-only {
        background-color: #000;
        background-repeat: no-repeat;
        background-position: center;
    }
}

.sendMsgContent {
    width: 100%;
    color: #fff;
    background-color: #333;
    height: 150px;
}

.botServer {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    border: solid 3px #0af;

    .botServerWsType {
        .input {
            padding: 0px;
        }
        input {
            margin: 0px;
        }
    }

    span,
    .buttons {
        align-self: center;
    }

    .buttons {
        display: flex;
        flex-direction: column;
    }
}
</style>

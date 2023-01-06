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
                    <button @click="sendMsg">点我发送</button>
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
                            <div>审核状态: {{ keywords.selectd.status == "checked" ? "通过" : "正在审核" }}</div>
                            <div>创建者: {{ keywords.selectd.ownerName }}</div>
                        </div>
                    </label>
                </th>
                <th>
                    <button @click="keywordExec.passCheck">点我通过审核</button>
                    <button @click="keywordExec.saveContent">点我保存回复内容</button>
                    <button @click="keywordExec.delete">点我删除</button>
                    <hr />
                    <div>发送状态</div>
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
import { ref, watch } from "vue";
import Notice from "./Notice.vue";

const noticeShow = ref();
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
const sendMsgContent = ref("");
const sended = ref({
    info: "未发送",
    class: "border: solid 3px #0af",
});
const keywords = ref({
    readonly: true,
    selectd: { index: 0, keyword: "NONE", content: "", ownerId: "", ownerName: "", refOwnerId: "", refOwnerName: "", status: "", type: "" },
    all: [{ keyword: "未选择", content: "", ownerId: "", ownerName: "", refOwnerId: "", refOwnerName: "", status: "", type: "" }],
});
keywords.value.all = [];
const keywordExec = {
    passCheck: () => {
        if (keywords.value.selectd.keyword == "NONE") return (keywordExec.status.value = { info: "未选择数据, 无法通过", class: "border: solid 3px #f00" });
        wsSend({ key: "keyword.passCheck", data: keywords.value.selectd }, "keyword.passCheck");
    },
    saveContent: () => {
        if (keywords.value.selectd.keyword == "NONE") return (keywordExec.status.value = { info: "未选择数据, 无法保存", class: "border: solid 3px #f00" });
        wsSend({ key: "keyword.saveContent", data: keywords.value.selectd }, "keyword.saveContent");
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
    "keyword.passCheck": (_keyword) => {
        keywords.value.all[_keyword.index] = _keyword;
        keywords.value.selectd = _keyword;
        if (_keyword.status == "checked") keywordExec.status.value = { info: `已通过审核`, class: "border: solid 3px #0af" };
    },
    "keyword.saveContent": (_keyword) => {
        keywordExec.status.value = { info: `已保存${_keyword.type == "accurate" ? "精确" : "模糊"}关键词"${_keyword.keyword}"的内容`, class: "border: solid 3px #0af" };
    },
    "keyword.delete": async (_keyword) => {
        if (!_keyword.isDel) return;
        keywords.value.readonly = true;
        keywordExec.status.value = { info: `已删除${_keyword.type == "accurate" ? "精确" : "模糊"}关键词"${_keyword.keyword}"`, class: "border: solid 3px #0af" };
        intervalHandler();
        keywords.value.selectd = { index: 0, keyword: "NONE", content: "", ownerId: "", ownerName: "", refOwnerId: "", refOwnerName: "", status: "", type: "" };
    },
};
watch(botStatus.value, () => {
    while (botStatus.value.length > 10) botStatus.value.shift();
});

function intervalHandler() {
    //if (!botServer.value.autoFetch) return;
    wsSend({ key: "channel.getList" }, "channel.getList");
    wsSend({ key: "keyword.get" }, "keyword.get");
}
function wsSend(content: { key: string; retKey?: string; data?: {} }, desc: string) {
    botStatus.value.push({
        msg: `于${new Date().toLocaleString()}发送数据: ${desc}`,
        classType: "0af",
    });
    ws!.send(JSON.stringify(content));
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
    //console.log(keywords.value.selectd);
    keywords.value.readonly = false;
    const _index = (e.target as any).selectedIndex;
    const index = keywords.value.selectd.content == "" ? _index - 1 : _index;
    keywords.value.selectd = Object.assign({ index }, keywords.value.all[index]);
}

function sendMsg() {
    // console.log(noticeShow.value);
    console.log(`发送至: ${channelInfo.value.selectd}, 发送内容: ${sendMsgContent.value}`);
    if (channelInfo.value.selectd == "NONE") {
        sended.value = { info: "错误: 未指定发送频道", class: "border: solid 3px #f00" };
    } else if (!sendMsgContent.value) {
        sended.value = { info: "错误: 发送内容为空", class: "border: solid 3px #f00" };
    } else {
        sended.value = { info: "发送中", class: "border: solid 3px #00f" };
        wsSend(
            {
                key: "channel.postMsg",
                retKey: "channel.recMsg",
                data: {
                    channelId: channelInfo.value.selectd,
                    content: sendMsgContent.value,
                },
            },
            "channel.postMsg"
        );
    }

    // noticeShow.value.noticeShow = !noticeShow.value.noticeShow;
}

init();
function init() {
    ws = new WebSocket(`${botServer.value.wsType}://${botServer.value.ip}:${botServer.value.port}`, botServer.value.token ? [botServer.value.token] : undefined);

    ws.onopen = () => {
        console.log("连接开启，状态：", ws!.readyState);
        botStatus.value.push({
            msg: `连接开启`,
            classType: "0af",
        });
    };
    ws.onmessage = (e) => {
        const data = JSON.parse(e.data);
        botStatus.value.push({
            msg: `于${new Date().toLocaleString()}接收数据: ${data.key}`,
            classType: "0f0",
        });
        console.log(`接收数据:`, data);
        wsIntentMessage[data.key](data.data);
    };
    ws.onclose = (e) => {
        // if (intervalId) {
        //     clearInterval(intervalId);
        //     intervalId = null;
        // }
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
        background-image: url("/noData.svg");
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

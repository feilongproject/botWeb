<template>
    <div class="sendMsg">
        <table>
            <tr>
                <th>功能</th>
                <th>数据输入</th>
                <th>选择发送对象</th>
                <th>开始发送!</th>
            </tr>
            <tr>
                <th><label for="sendMsgContent">向子频道发送消息</label></th>
                <th>
                    <textarea id="sendMsgContent" name="sendMsgContent" class="sendMsgContent" v-model="sendMsgContent"></textarea>
                </th>
                <th style="width: 30%">
                    <label for="channelSelect">选择子频道</label>
                    <select id="channelSelect" style="width: 50%" v-model="channelInfo.selectd" @change="selectChannel" required>
                        <option v-if="channelInfo.all.length != 0 && channelInfo.selectd == 'NONE'" value="NONE">未选择</option>
                        <option v-for="(options, id) in channelInfo.all" :key="id" :value="options.id">{{ options.name }}</option>
                    </select>
                </th>
                <th>
                    <button @click="sendMsg">点我发送</button>
                    <hr />
                    <div>发送状态</div>
                    <div :style="sendedClass">{{ sendedInfo }}</div>
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
            <div><label for="botServerInputUrl">地址</label> <input id="botServerInputUrl" type="text" v-model="botServerUrl" /></div>
            <div><label for="botServerInputToken">Token</label> <input id="botServerInputToken" type="text" v-model="botServerToken" /></div>
            <div>
                <button @click="saveServer">保存服务器设置</button>
                <button @click="closeServer">关闭连接</button>
                <button @click="changeReconnent">{{ botServerReconnent ? "关闭" : "开启" }}自动重连</button>
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
const botServerUrl = ref(localStorage.getItem("botServerUrl") || "ws://127.0.0.1:8848");
const botServerToken = ref(localStorage.getItem("botServerToken") || "");
const botServerReconnent = ref(true);
const sendMsgContent = ref("");
const sendedInfo = ref("未发送");
const sendedClass = ref("border: solid 3px #0af");
const channelInfo = ref({
    selectd: "NONE",
    all: [{ id: "NONE", name: "未选择", selected: true, type: 0 }],
});
var ws = new WebSocket(`${botServerUrl.value}?token=${botServerToken.value}`, botServerToken.value ? [botServerToken.value] : undefined);

const wsIntentMessage: { [key: string]: (data: any) => void } = {
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
        sendedInfo.value = `发送成功`;
        sendedClass.value = "border: solid 3px #0af";
    },
};

watch(botStatus.value, () => {
    while (botStatus.value.length > 5) botStatus.value.shift();
});

init();
function init(isReconnent = false) {
    if (isReconnent) ws = new WebSocket(`${botServerUrl.value}?token=${botServerToken.value}`, botServerToken.value ? [botServerToken.value] : undefined);

    ws.onopen = () => {
        console.log("连接开启，状态：", ws.readyState);
        botStatus.value.push({
            msg: `连接开启`,
            classType: "0af",
        });
        ws.send(JSON.stringify({ key: "channel.getList" }));
    };
    ws.onmessage = (e) => {
        botStatus.value.push({
            msg: `于${new Date().toLocaleString()}接收数据`,
            classType: "0af",
        });
        const data = JSON.parse(e.data);
        console.log(`接收数据:`, data);
        wsIntentMessage[data.key](data.data);
    };
    ws.onclose = (e) => {
        const closeInfo = `${e.reason ? e.reason : "未知原因"} | 连接被关闭, 正在重新连接`;
        console.log(closeInfo);
        botStatus.value.push({ msg: closeInfo, classType: "f00" });
        (function aaa(times: number) {
            setTimeout(() => {
                if (!botServerReconnent.value) return;
                console.log(`重新连接中, 等待${times}s, 服务器状态: ${ws.readyState}`);
                botStatus.value.push({
                    msg: closeInfo + `\n重新连接中, 等待${times}s`,
                    classType: "f00",
                });
                if (times) aaa(times - 1);
                else init(true);
            }, 1000);
        })(5);
    };
    /* ws.onerror = (e) => {
    console.log("发生错误，状态:" + ws.readyState, e);
    botStatus.value = "连接发生错误";
    botError.value = "连接发生错误";
}; */
}

function saveServer(e: MouseEvent) {
    localStorage.setItem("botServerUrl", botServerUrl.value);
    localStorage.setItem("botServerToken", botServerToken.value);
}
function closeServer(e: MouseEvent) {
    ws.close(1000, "客户端主动关闭");
}

function selectChannel(e: Event) {
    //console.log(e, channelInfo.value);
}

function changeReconnent() {
    botServerReconnent.value = !botServerReconnent.value;
    botStatus.value.push({ msg: `已${botServerReconnent.value ? "开启" : "关闭"}自动重连`, classType: "00f" });
    if (botServerReconnent.value) init(true);
}

function sendMsg() {
    // console.log(noticeShow.value);
    console.log(`发送至: ${channelInfo.value.selectd}, 发送内容: ${sendMsgContent.value}`);
    if (channelInfo.value.selectd == "NONE") {
        sendedInfo.value = "错误: 未指定发送频道";
        sendedClass.value = "border: solid 3px #f00";
    } else if (!sendMsgContent.value) {
        sendedInfo.value = "错误: 发送内容为空";
        sendedClass.value = "border: solid 3px #f00";
    } else {
        botStatus.value.push({
            msg: `于${new Date().toLocaleString()}发送数据`,
            classType: "0af",
        });
        sendedInfo.value = "发送中";
        sendedClass.value = "border: solid 3px #00f";
        ws.send(
            JSON.stringify({
                key: "channel.postMsg",
                retKey: "channel.recMsg",
                data: {
                    channelId: channelInfo.value.selectd,
                    content: "[web测试]" + sendMsgContent.value,
                },
            })
        );
    }

    // noticeShow.value.noticeShow = !noticeShow.value.noticeShow;
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
    height: 300px;
    width: 70%;
    flex-direction: column;
    justify-content: space-around;
    align-items: center;
}

.botStatus,
.sendMsg {
    display: flex;
    justify-content: space-evenly;
}

.sendMsgContent {
    width: 100%;
    color: #fff;
    background-color: #333;
    height: 200px;
}

.botServer {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    border: solid 3px #0af;

    span,
    div:last-child {
        align-self: center;
    }

    div:last-child {
        display: flex;
        flex-direction: column;
    }
}
</style>

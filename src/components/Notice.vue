<template>
    <div v-if="noticeShow" class="notice">
        <div class="body">
            <div class="title">
                <span>{{ title }}</span>
            </div>
            <div class="content">
                <slot></slot>
            </div>
            <div>
                <button class="btn" @click="noticeShow = !noticeShow">{{ delayClose ? `还有${delayClose}s` : "" }}关闭</button>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, watch } from "vue";

const props = defineProps(["title", "delay"]);
const title = ref(props.title);
const delay = ref(Number(props.delay));
const delayClose = ref(0);
const noticeShow = ref(false);

watch(noticeShow, (newValue, oldValue) => {
    //console.log(newValue && delay.value, newValue, delay.value);
    if (newValue && delay.value) {
        delayClose.value = delay.value;
        (function timeOut() {
            setTimeout(() => {
                if (delayClose.value != 0) {
                    delayClose.value--;
                    timeOut();
                } else noticeShow.value = false;
            }, 1000);
        })();
    }
});

defineExpose({ noticeShow });
</script>

<style scoped>
.notice {
    position: fixed;
    top: 0;
    bottom: 80%;
    left: 80%;
    right: 0;
    background-color: rgba(0, 0, 0, 0.4);
}
.body {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    align-items: center;
    width: 80%;
    height: 80%;
    margin: 10%;
    background-color: #333333;
    box-shadow: 0 0 5px 1px darkgray;
}
.title {
    color: #ffffff;
    margin-top: 10px;
    font-size: 20px;
}
.content {
    width: 500px;
    height: 100%;
    margin-top: 10px;
    text-align: center;
    overflow-y: auto;
}
</style>

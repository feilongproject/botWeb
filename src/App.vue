<template>
    <div class="board">
        <h1>botWeb</h1>
    </div>
    <ul class="board menu">
        <li class="menu-item">主页</li>
        <li class="menu-item">
            <a class="menu-item" href="https://github.com/feilongproject/botWeb/" target="_blank">开源地址</a>
        </li>
    </ul>
    <div class="board">
        <Main />
    </div>
</template>

<script setup lang="ts">
import Main from "./components/Main.vue";

console.log("欢迎使用botWeb");
console.log("%c 作者 -> 飞龙project ", "background:#000; color:#fff");
console.log("%c GitHub -> https://github.com/feilongproject/botWeb/ ", "background:#000; color:#fff");
</script>

<style scoped>
/* .main {
    background-color: #0df;
    box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
    border-radius: 5px;
    display: flex;
    flex-direction: column;
    margin-left: 20px;
    margin-right: 20px;
} */
li {
    list-style: none;
}

.menu {
    border-radius: 5px;
    display: flex;
    flex-direction: row;
}

.menu-item {
    margin: 5px 5px 5px 5px;
}
</style>

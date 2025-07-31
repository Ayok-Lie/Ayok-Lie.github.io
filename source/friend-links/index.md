---
title: 友链
date: 2023-10-05 14:55:22
---

## Friend Links

<div style="display: flex; justify-content: space-between;">
    <div>
        <img src="https://example.com/avatar1.jpg" alt="Avatar 1" style="width: 60px; height: 60px; border-radius: 50%;">
    </div>
    <div>
        <img src="https://example.com/avatar2.jpg" alt="Avatar 2" style="width: 60px; height: 60px; border-radius: 50%;">
    </div>
    <!-- 添加更多头像 -->
</div>

<button onclick="randomVisit()">Random visit</button>
<button onclick="applyFriendList()">Apply friend list</button>

<script>
function randomVisit() {
    // 随机访问逻辑
}

function applyFriendList() {
    // 应用好友列表逻辑
}
</script>

## Personal (5)

Record every single step of the way.

<div class="personal-friends">
    <div class="friend-card">
        <img src="https://example.com/fantasy-ke.jpg" alt="fantasy-ke" style="width: 100%; height: auto;">
        <h3>fantasy-ke</h3>
        <p>我不生产代码，我只是个搬运工。</p>
    </div>
    <div class="friend-card">
        <img src="https://example.com/jonty.jpg" alt="jonty" style="width: 100%; height: auto;">
        <h3>Jonty</h3>
        <p>On the way to becoming a DevOps engineer.</p>
    </div>
    <!-- 添加更多个人友链卡片 -->
</div>

<style>
.personal-friends {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
}

.friend-card {
    width: calc(20% - 20px);
    background-color: #f9f9f9;
    padding: 10px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.friend-card img {
    border-radius: 8px;
}
</style>
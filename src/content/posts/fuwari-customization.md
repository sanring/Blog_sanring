---
title: Fuwari 博客主题深度定制记录
published: 2026-03-20
description: 记录为 Fuwari 博客主题添加友链系统、主页 Hero 横幅、打字机特效等功能的完整过程
tags: [博客, Astro, 前端, 教程]
category: '技术'
draft: false
lang: ''
---

# Fuwari 博客主题深度定制记录

本文记录了对 Astro 博客主题 [Fuwari](https://github.com/saicaca/fuwari) 的一系列深度定制，包括友链系统、主页 Hero 横幅、打字机古诗特效等功能的实现过程，以及调试中遇到的各种坑。所有改动均附 diff 代码块。

---

## 一、友链系统

### 1.1 添加友链菜单入口

**文件：`src/config.ts`**

```diff
 navbar: [
     LinkPreset.Home,
     LinkPreset.Archive,
+    { name: '友人帐', url: '/friends/' },
     LinkPreset.About,
 ]
```

### 1.2 创建友链页面

新建 **`src/pages/friends.astro`**，包含友链数据、标签筛选 UI 和卡片网格布局：

```astro
---
import MainGridLayout from "../layouts/MainGridLayout.astro";
import FriendCard from "@components/FriendCard.astro";

const friends = [
    {
        name: "fuwari",
        description: "博客主题",
        url: "https://github.com/saicaca/fuwari",
        avatar: "assets/images/demo-avatar.png",
        tag: "博客"
    },
    // ... 更多友链
]
const tags = ["全部", ...new Set(friends.map(f => f.tag).filter(Boolean))];
---
<MainGridLayout title="友人帐" description="友情链接">
    <!-- 标签筛选按钮 -->
    <div class="flex flex-wrap gap-2 px-2">
        {tags.map(tag => (
            <button class="filter-btn ..." data-tag={tag}>{tag}</button>
        ))}
    </div>
    <!-- 友链卡片网格 -->
    <div class="grid grid-cols-1 md:grid-cols-2 gap-4" id="friend-grid">
        {friends.map(friend => (
            <div class="friend-card-wrapper" data-tag={friend.tag}>
                <FriendCard {...friend} />
            </div>
        ))}
    </div>
</MainGridLayout>
```

### 1.3 新建友链卡片组件

新建 **`src/components/FriendCard.astro`**，支持本地图片和远程 URL 头像。

### 1.4 标签筛选逻辑（Swup 兼容）

> [!IMPORTANT]
> 友链页面的 `<script>` 标签位于 `<MainGridLayout>` 外部，不在 Swup 的 `#swup-container` 内。Swup 替换内容时**不会重新执行**这些脚本。必须将筛选逻辑移到 `MainGridLayout.astro` 的全局脚本中。

**文件：`src/layouts/MainGridLayout.astro`**（全局 `<script is:inline>` 块内新增）

```diff
+    // === Friends page tag filtering ===
+    function setupFilters() {
+        const buttons = document.querySelectorAll('.filter-btn');
+        const cards = document.querySelectorAll('.friend-card-wrapper');
+        if (!buttons.length) return;
+
+        function filter(selectedTag) {
+            buttons.forEach(b => {
+                b.classList.toggle('active-filter',
+                    b.getAttribute('data-tag') === selectedTag);
+            });
+            cards.forEach(card => {
+                const cardTag = card.getAttribute('data-tag');
+                card.style.display =
+                    (selectedTag === '全部' || cardTag === selectedTag)
+                    ? 'block' : 'none';
+            });
+        }
+
+        buttons.forEach(btn => {
+            btn.addEventListener('click', () => {
+                filter(btn.getAttribute('data-tag'));
+            });
+        });
+        filter('全部');
+    }
+    setupFilters();
```

---

## 二、主页全屏 Hero 横幅

### 2.1 调整横幅高度为全屏

**文件：`src/constants/constants.ts`**

```diff
 export const BANNER_HEIGHT = 35;
-export const BANNER_HEIGHT_EXTEND = 15;
+export const BANNER_HEIGHT_EXTEND = 73;
 export const BANNER_HEIGHT_HOME = BANNER_HEIGHT + BANNER_HEIGHT_EXTEND;
```

> 35vh + 73vh = 108vh，略大于视口高度，保证全屏覆盖。

### 2.2 添加 Hero 覆盖层（标题 + 打字机 + 箭头）

**文件：`src/layouts/MainGridLayout.astro`**（`#banner-wrapper` 内部新增）

```diff
     </ImageWrapper>
+    <div class="absolute inset-0 flex flex-col items-center justify-center
+         text-white z-20" id="hero-overlay"
+         style="text-shadow: 0 2px 4px rgba(0,0,0,0.8);">
+        <div class="text-4xl md:text-5xl font-bold mb-4 font-hero">
+            {siteConfig.title}
+        </div>
+        <div class="text-lg md:text-xl flex items-center font-hero">
+            <span id="typewriter-text"></span>
+            <span class="animate-pulse ml-1">|</span>
+        </div>
+        <div class="absolute bottom-8 left-1/2 -translate-x-1/2
+             pointer-events-auto cursor-pointer text-white/80 hover:text-white"
+             onclick="const mg = document.getElementById('main-grid');
+                      if(mg) window.scrollBy({
+                          top: mg.getBoundingClientRect().top,
+                          behavior: 'smooth'
+                      });">
+            <Icon name="material-symbols:keyboard-arrow-down-rounded"
+                  class="text-5xl animate-bounce" />
+        </div>
+    </div>
 </div>}
```

### 2.3 修复向下箭头点击无响应

Hero 区域的箭头位于 `#banner-wrapper`（z-10），而内容区容器（z-30）虽然视觉上透明，`pointer-events-auto` 会拦截所有点击。需要将 `pointer-events-auto` 下移到 `#main-grid`：

```diff
 <div class="absolute w-full z-30 pointer-events-none" ...>
-    <div class="relative max-w-[...] mx-auto pointer-events-auto">
-        <div id="main-grid" class="... mx-auto gap-4 px-0 md:px-4"
+    <div class="relative max-w-[...] mx-auto pointer-events-none">
+        <div id="main-grid" class="... mx-auto gap-4 px-0 md:px-4 pointer-events-auto"
```

### 2.4 SPA 导航兼容 CSS

Hero 覆盖层的显隐通过 CSS 控制，确保 Swup 跳转正常切换：

```diff
+    body:not(.lg\:is-home) #hero-overlay {
+        opacity: 0 !important;
+        pointer-events: none !important;
+    }
```

---

## 三、打字机古诗特效（Hitokoto API）

### 3.1 接入一言 API

使用 [Hitokoto（一言）](https://hitokoto.cn/) API 获取随机古诗词，参数 `c=i` 仅返回诗词类：

```javascript
async function fetchNextPhrase() {
    try {
        const response = await fetch('https://v1.hitokoto.cn?c=i');
        const data = await response.json();
        return data.hitokoto;
    } catch (e) {
        return "欲买桂花同载酒，终不似，少年游。";
    }
}
```

### 3.2 动画参数

| 参数 | 说明 | 当前值 |
|------|------|--------|
| 打字速度 | 每个字的打字间隔 | `50ms` |
| 删除速度 | 每个字的删除间隔 | `50ms` |
| 打完暂停 | 显示完整诗句的停留时间 | `2000ms` |
| 删完暂停 | 删除完到开始新诗句的间隔 | `150ms` |

### 3.3 解决异步竞态导致的文字闪烁

打字机效果在页面跳转时会出现文字闪烁。根本原因是 **异步竞态条件**：

当 `initTypewriter()` 被调用时，如果上一个实例正停在 `await fetch()` 上：

1. `clearTimeout()` 只能清除已排队的定时器，**无法终止挂起的 async 函数**
2. 网络请求返回后，旧实例恢复执行并调度新的 `setTimeout`
3. 新旧两个 `type()` 链条同时运行，争抢同一个 `<span>`，导致闪烁

**解决方案**：引入 `instanceId` 取消令牌：

```diff
 let typeTimeout = null;
+let typeInstanceId = 0;

 function initTypewriter() {
     const el = document.getElementById('typewriter-text');
     if (!el) return;

-    if (typeTimeout) clearTimeout(typeTimeout);
+    typeInstanceId++;
+    const myId = typeInstanceId;
+    if (typeTimeout) clearTimeout(typeTimeout);

     // ...

     async function type() {
-        if (!document.getElementById('typewriter-text')) return;
+        if (myId !== typeInstanceId) return;  // 已被新实例取代
+        if (!document.getElementById('typewriter-text')) return;

         if (currentPhrase === "") {
-            await fetchNextPhrase();
+            currentPhrase = await fetchNextPhrase();
+            if (myId !== typeInstanceId) return;  // await 后再检查
         }

         // ... 打字逻辑 ...

-        typeTimeout = setTimeout(type, typeSpeed);
+        typeTimeout = setTimeout(() => {
+            if (myId === typeInstanceId) type();  // 回调中也检查
+        }, typeSpeed);
     }
 }
```

### 3.4 Swup 事件监听器防重复

使用全局守卫防止 `content:replace` 被多次注册：

```diff
+    if (!window._swupContentBound) {
+        window._swupContentBound = true;
+        document.addEventListener('swup:content:replace', () => {
+            if (document.getElementById('typewriter-text')) {
+                initTypewriter();
+            }
+            setupFilters();
+        });
+    }
```

---

## 四、Butterfly 风格视差滚动（探索记录）

### 4.1 目标

参考 [Butterfly 主题](https://butterfly.js.org/) 实现首页背景图固定不动、内容从下方滑上覆盖图片的效果。

### 4.2 源码分析

Butterfly 核心实现仅一行 CSS（`source/css/_layout/head.styl`）：

```stylus
&.full_page
  height: $index_top_img_height
  background-attachment: fixed
```

Butterfly 使用 CSS `background-image` + `background-attachment: fixed`。Fuwari 使用 `<img>` 标签，无法直接使用此属性。

### 4.3 尝试与结论

| 方案 | 原理 | 结果 |
|------|------|------|
| `position: fixed` + `lg:is-home` | 图片固定，靠 CSS 类切换 | ❌ Swup 的 `visit:start` 在淡出前就移除类，导致图片弹回产生抖动 |
| JS `butterfly-fixed` 类 + Swup 生命周期 | 在 `content:replace`（淡出后）才移除固定 | ⚠️ 可行但增加额外复杂度 |
| `::before` 纯色背景遮罩 | `#main-grid` 上叠加纯色背景覆盖固定图片 | ⚠️ 与 Fuwari 原始过渡动画冲突 |

> [!WARNING]
> **结论**：Fuwari 的 Swup 过渡系统精心设计了 `transition duration-700` + `lg:is-home` 类切换的时序配合。`position: fixed` 方案会与这套机制冲突。如需此效果，建议将 `ImageWrapper` 组件的 `<img>` 替换为 CSS `background-image`，使用原生 `background-attachment: fixed`。

---

## 五、自定义字体

```diff
+    .font-hero {
+        font-family: 'SimSun', 'Songti SC', 'STSong', serif;
+    }
```

如需自定义字体文件，放到 `public/fonts/` 后：

```diff
+    @font-face {
+        font-family: 'HeroFont';
+        src: url('/fonts/your-font.ttf') format('truetype');
+        font-display: swap;
+    }
+    .font-hero {
-        font-family: 'SimSun', 'Songti SC', 'STSong', serif;
+        font-family: 'HeroFont', serif;
+    }
```

---

## 六、更换主页背景图

**文件：`src/config.ts`**

```diff
 banner: {
     enable: true,
-    src: "assets/images/demo-banner.png",
+    src: "https://picture.sanring.sbs/api/random",
     position: "center",
 }
```

---

## 七、涉及的文件清单

| 文件 | 操作 | 修改内容 |
|------|------|----------|
| `src/config.ts` | 修改 | 菜单项、横幅图片配置 |
| `src/constants/constants.ts` | 修改 | `BANNER_HEIGHT_EXTEND` → 73 |
| `src/layouts/MainGridLayout.astro` | 修改 | Hero 覆盖层、打字机、筛选、pointer-events 修复 |
| `src/pages/friends.astro` | 新建 | 友链页面 |
| `src/components/FriendCard.astro` | 新建 | 友链卡片组件 |
| `src/content/spec/friends.md` | 新建 | 友链页面描述文案 |

---

## 八、踩坑总结

> [!CAUTION]
> **Swup SPA 导航下的脚本问题**
> 
> Fuwari 的 Swup 只替换 `#swup-container` 内的内容。页面级 `<script>` 不在容器内，Swup 不会执行。
> 
> **解决**：将脚本移到 `MainGridLayout.astro` 的全局 `<script is:inline>` 中，通过 `swup:content:replace` 事件重新绑定。

> [!WARNING]
> **事件监听器重复注册**
> 
> `<script is:inline>` 每次页面加载都会执行。重复 `addEventListener` 导致同一事件被多次监听。
> 
> **解决**：使用 `window._swupContentBound` 守卫标志，确保全局只注册一次。

> [!CAUTION]
> **异步函数的竞态条件**
> 
> `clearTimeout()` 无法终止正在 `await` 中挂起的 async 函数。旧实例恢复后会与新实例同时运行，导致 UI 闪烁。
> 
> **解决**：使用递增的 `instanceId` 作为取消令牌，在 `await` 前后和 `setTimeout` 回调内都检查 ID 有效性。

> [!WARNING]
> **`position: fixed` 与 Swup 过渡冲突**
> 
> Swup 在 `visit:start`（页面淡出前）就移除 `lg:is-home` 类。CSS 据此切换 `position: fixed` 会导致布局跳动。
> 
> **解决**：避免在 Fuwari 中使用 `position: fixed` 控制 banner。如需固定效果应使用 `background-attachment: fixed`。

> [!NOTE]
> **pointer-events 层叠遮挡**
> 
> Fuwari z-index 层级：banner (z-10) → content (z-30)。z-30 层的 `pointer-events-auto` 即使视觉透明也会拦截 z-10 的点击。需将 `pointer-events-auto` 精确设置在实际内容元素 `#main-grid` 上。

> [!NOTE]
> **CSS 选择器中的冒号转义**
> 
> Fuwari 的 body 类名 `lg:is-home` 含冒号，CSS 中须写 `body.lg\:is-home`。漏掉反斜杠选择器静默失效。

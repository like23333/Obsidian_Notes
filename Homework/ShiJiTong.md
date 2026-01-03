# 事记通 (ShiJiTong) - 基于 HarmonyOS 的全栈个人事务管理系统开发文档

## 1. 项目概述 (Project Overview)

### 1.1 项目背景
随着移动互联网的发展，个人事务碎片化日益严重。为了解决用户在不同场景下记录灵感、管理待办事项的需求，本项目开发了一款名为“事记通”的移动端应用。该应用不仅具备传统的笔记与待办功能，更引入了**“端云一体化”**的架构思想，解决了弱网环境下的数据可用性问题。

### 1.2 核心特性
*   **全栈开发**：前端采用 **HarmonyOS (ArkTS/ArkUI)**，后端采用 **Node.js (Express) + MySQL**。
*   **离线优先 (Offline First)**：基于 **SQLite** 构建本地持久化存储，支持无网状态下的增删改查。
*   **差异化同步**：设计了基于 `server_id` 的状态机同步算法，智能合并本地与云端数据。
*   **沉浸式体验**：适配 HarmonyOS 全屏显示，动态计算状态栏避让区域。
*   **安全机制**：包含隐私政策弹窗、密码加密传输（基础）、用户鉴权机制。

---

## 2. 系统架构设计 (System Architecture)

系统采用经典的 **C/S (Client-Server)** 三层架构设计：

1.  **表现层 (Presentation Layer)**:
    *   运行于 HarmonyOS 设备。
    *   使用 **ArkUI** 构建声明式界面。
    *   通过 **ViewModel** (页面的 `@State` 状态) 驱动 UI 更新。
2.  **业务逻辑层 (Business Logic Layer)**:
    *   **Client端**: `HttpUtil` 封装网络请求，`SQLiteHelper` 处理本地缓存逻辑。
    *   **Server端**: Node.js 处理路由分发、业务校验、文件流处理。
3.  **数据持久层 (Data Persistence Layer)**:
    *   **Client端**: HarmonyOS `RelationalStore` (SQLite)，用于离线存储。
    *   **Server端**: MySQL 5.7+，作为数据的“唯一真理源 (Single Source of Truth)”。

---

## 3. 详细功能模块说明

### 3.1 用户认证模块
*   **注册/登录**: 支持手机号作为唯一凭证。登录成功后，服务器返回 `userId`，客户端将其持久化至 `AppStorage`，并在后续请求中携带。
*   **隐私合规**: 应用首次启动时，通过 `PersistentStorage` 检测标志位。若为首次启动，强制弹出 `PrivacyDialog`（隐私政策），用户同意后方可进入。
*   **密码找回**: 独立的 `ForgetPage`，通过验证手机号和 ID 匹配性来重置密码。

### 3.2 待办事项 (Todo) 模块
*   **双重存储策略**:
    *   **新增**: 用户点击添加 -> 写入本地 SQLite (标记 `server_id=0`) -> 尝试发起网络请求。
        *   若网络成功: 获取服务器返回的 ID，更新本地记录的 `server_id`。
        *   若网络失败: 保持本地记录，待下次同步时上传。
    *   **查询**: 页面加载时优先读取本地 SQLite 数据（毫秒级渲染），随后静默触发后台同步。
*   **状态管理**: 支持待办事项的“完成/未完成”状态切换，通过视觉样式（删除线、透明度）区分。

### 3.3 云笔记 (Cloud Notes) 模块
*   **富文本与列表**: 首页展示笔记列表，点击进入详情页编辑。
*   **智能标题**: 若用户未输入标题，系统自动截取正文前 10 个字符作为标题。
*   **操作同步**: 支持笔记的新建、编辑保存、删除操作，逻辑与 Todo 模块一致，均包含离线兜底机制。

### 3.4 数据统计仪表盘
*   **聚合查询**: 服务端 `/api/stats` 接口采用 SQL 聚合查询 (`COUNT`)，一次性返回：
    *   笔记总数
    *   待办总数
    *   已完成数
    *   未完成数
*   **可视化**: 前端使用 `Circle` 组件绘制动态环形进度条，直观展示任务完成率。

### 3.5 启动与引导
*   **启动页 (Splash)**: 加载服务端配置的广告图，包含 3秒倒计时与“跳过”功能。
*   **引导页 (Guide)**: 首次安装应用时展示轮播图，介绍核心功能。

---

## 4. 数据库详细设计

### 4.1 本地数据库 (SQLite - RDB)
位于客户端，文件名为 `ShiJiTong.db`。

**表1：待办事项表 (`local_todos`)**



| 字段名 | 类型 | 作用描述 |
| :--- | :--- | :--- |
| `id` | INTEGER | **主键**，本地唯一标识，自增。 |
| `server_id`| INTEGER | **核心同步字段**。`0` 表示该数据仅在本地，需上传；非 `0` 表示已同步，对应服务端 ID。 |
| `content` | TEXT | 待办内容。 |
| `status` | INTEGER | `0`: 进行中, `1`: 已完成。 |

**表2：笔记表 (`local_notes`)**


| 字段名 | 类型 | 作用描述 |
| :--- | :--- | :--- |
| `id` | INTEGER | **主键**。 |
| `server_id`| INTEGER | 同步标记字段，逻辑同上。 |
| `user_id` | INTEGER | 数据所属用户 ID。 |
| `title` | TEXT | 笔记标题。 |
| `content` | TEXT | 笔记正文。 |

### 4.2 服务端数据库 (MySQL)
位于服务器，库名为 `db_shijitong`。

**表3：用户表 (`users`)**


| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| `id` | INT | PK, AI | 用户全局唯一 ID |
| `phone` | VARCHAR(20) | UNIQUE | 登录账号 |
| `password` | VARCHAR(50) | NOT NULL | 登录密码 |
| `avatar` | VARCHAR(255) | - | 头像相对路径 (如 `/uploads/xxx.jpg`) |

**表4：待办表 (`todos`)**
*   **外键**: `user_id` 关联 `users.id` (级联删除)。
*   **字段**: `content`, `status`, `created_at`。

**表5：笔记表 (`notes`)**
*   **外键**: `user_id` 关联 `users.id` (级联删除)。
*   **字段**: `title`, `content`, `created_at`。

---

## 5. 关键技术实现原理

### 5.1 数据同步算法 (Sync Algorithm)
代码位置：`entry/src/main/ets/common/database/SQLiteHelper.ets`

同步流程被封装在 `syncOverwrite` 方法中，采用 **"Push-Pull-Merge"** 策略：

1.  **Push (上传离线数据)**:
    *   查询 SQLite 中所有 `server_id = 0` 的记录。
    *   遍历这些记录，调用后端 `ADD` 接口。
2.  **Pull (拉取全量数据)**:
    *   调用后端 `GET` 接口，获取该用户云端所有数据。
3.  **Merge (本地覆盖)**:
    *   开启数据库事务 (`beginTransaction`)。
    *   **删除**所有已同步的数据 (`server_id != 0`)。注意：保留 `server_id == 0` 的数据，防止在同步过程中用户新产生的数据丢失。
    *   **插入**从云端拉取的新数据。
    *   提交事务 (`commit`)。

### 5.2 网络请求封装
代码位置：`entry/src/main/ets/common/utils/HttpUtil.ets`

*   **问题**: ArkTS 的 `@ohos.net.http` 返回的是 JSON 字符串，且在严格模式下类型推断困难。
*   **解决**: 封装 `HttpUtil` 类，提供静态 `get/post` 方法。内部处理 `http.createHttp()` 的生命周期，自动解析 `JSON.parse`，并定义统一的泛型接口 `ResponseResult`，使得业务层调用时代码极其简洁，不仅解决了回调地狱，还统一了异常处理。

### 5.3 沉浸式 UI 适配
代码位置：`entry/src/main/ets/entryability/EntryAbility.ts`

为了适应不同机型的刘海屏和挖孔屏：
1.  在 `onWindowStageCreate` 中调用 `window.setWindowLayoutFullScreen(true)`。
2.  获取 `AvoidAreaType.TYPE_SYSTEM` 的高度。
3.  将高度存入 `AppStorage.setOrCreate('topHeight', val)`。
4.  在所有页面的根容器设置 `padding({ top: px2vp(this.topHeight) })`，确保内容不被状态栏遮挡。

---

## 6. 服务端接口文档 (API Reference)

服务端基准地址：`http://[你的IP]:3000`

### 6.1 公共模块
*   **获取配置**
    *   `GET /api/common/config`
    *   响应: `{ code: 200, data: { splash_bg: "url", ads: [...] } }`

### 6.2 待办模块
*   **获取列表**
    *   `GET /api/todos?userId=1`
*   **新增待办**
    *   `POST /api/todos`
    *   Body: `{ "userId": 1, "content": "复习高数" }`
*   **更新状态**
    *   `POST /api/todos/update`
    *   Body: `{ "id": 105, "status": 1 }`

### 6.3 个人中心
*   **上传头像**
    *   `POST /api/user/avatar`
    *   Header: `Content-Type: multipart/form-data`
    *   Body: `avatar` (File), `userId` (String)

*(其他接口如 Login, Register, Notes 遵循相同 RESTful 规范)*

---

## 7. 部署指南

### 环境要求
*   **前端**: DevEco Studio 3.1+, HarmonyOS SDK API 9。
*   **后端**: Node.js 14+, MySQL 5.7+。

### 步骤一：数据库初始化
1.  打开 MySQL 管理工具 (Navicat/DBeaver)。
2.  新建数据库 `db_shijitong`。
3.  执行 `db_shijitong.sql` 脚本建表。

### 步骤二：后端服务启动
1.  进入 `SHIJITONG-SERVER` 目录。
2.  安装依赖: `npm install`。
3.  **配置修改**: 打开 `app.js`，修改 `mysql.createConnection` 中的 `password` 为你的数据库密码。
4.  创建上传目录: 手动在 `public` 下创建 `uploads` 文件夹。
5.  启动: `node app.js`。

### 步骤三：客户端编译运行
1.  打开 `entry/src/main/ets/common/constants/Api.ets`。
2.  **关键配置**: 将 `SERVER_IP` 的值修改为你电脑的 **IPv4 地址** (例如 `http://192.168.1.101:3000`)。
3.  连接真机或启动模拟器，点击运行。

---

## 8. 附录：核心代码文件清单

```text
ShiJiTong
├── common
│   ├── database/SQLiteHelper.ets  <-- 本地数据库管理 (核心)
│   └── utils/HttpUtil.ets         <-- 网络请求工具
├── entryability/EntryAbility.ts   <-- 全屏适配逻辑
├── pages
│   ├── MainPage.ets               <-- 主界面 (Tabs)
│   ├── TodoPage.ets               <-- 待办业务逻辑
│   └── SplashPage.ets             <-- 启动广告逻辑
└── SHIJITONG-SERVER/app.js        <-- 后端业务逻辑
```
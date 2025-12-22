# ShiJiTong

# 事记通 (ShiJiTong) 全栈项目工程文档

## 1. 项目概述

本项目是一个基于 HarmonyOS (ArkTS) 客户端与 Node.js (Express) 服务端的分布式个人事务管理系统。系统架构采用**离线优先 (Offline-First)** 策略，通过本地 SQLite 数据库与云端 MySQL 数据库的双向同步机制，确保用户在弱网或无网环境下仍可正常使用核心功能。

---

## 2. 项目目录结构

### 2.1 前端结构 (HarmonyOS - ShiJiTong)

项目遵循 MVVM 架构模式，通过模块化分层实现关注点分离。

```
ShiJiTong
├── AppScope
│   ├── resources
│   │   └── base
│   │       └── media               // [全局资源] 存放应用图标、启动图等全局静态资源
│   └── app.json5                   // [全局配置] 定义应用包名、版本号及权限申请
├── entry
│   └── src
│       └── main
│           ├── ets
│           │   ├── common                  // [公共层] 存放跨模块复用的工具与常量
│           │   │   ├── constants
│           │   │   │   └── Api.ets             // [常量] 集中管理后端 API 端点与环境配置
│           │   │   ├── database
│           │   │   │   └── SQLiteHelper.ets    // [持久化层] 封装 RDB Store，实现本地数据的 CRUD 及云端同步策略
│           │   │   └── utils
│           │   │       └── HttpUtil.ets        // [网络层] 封装 HTTP 请求，统一处理请求头、超时及响应拦截
│           │   ├── entryability
│           │   │   └── EntryAbility.ets        // [应用入口] 管理 UIAbility 生命周期，负责数据库初始化与窗口配置
│           │   ├── model                   // [模型层] 定义领域对象 (Domain Objects)
│           │   │   ├── NoteModel.ets           // 笔记实体类 (包含本地 ID 与服务端 ID 映射)
│           │   │   └── TodoModel.ets           // 待办实体类
│           │   ├── pages                   // [视图层] 页面级组件
│           │   │   ├── SplashPage.ets          // 启动页：处理广告预加载与路由分发
│           │   │   ├── LoginPage.ets           // 登录页：处理身份认证与 Session 持久化
│           │   │   ├── RegisterPage.ets        // 注册页
│           │   │   ├── ForgetPage.ets          // 密码重置页
│           │   │   ├── MainPage.ets            // 主容器：包含 Tabs 导航、离线同步触发器及新手引导逻辑
│           │   │   ├── TodoPage.ets            // 待办业务页：实现乐观 UI 更新与本地优先存储
│           │   │   └── NoteEditPage.ets        // 笔记编辑页
│           │   └── view                    // [组件层] 可复用的 UI 组件
│           │       └── PrivacyDialog.ets       // 隐私政策弹窗
│           ├── resources                   // [模块资源]
│           │   └── base
│           │       ├── media               // 业务相关图片资源 (icon, background 等)
│           │       └── element             // 字符串、颜色等资源配置
│           └── module.json5                // [模块配置] 定义 Ability 属性及页面路由表

```

### 2.2 后端结构 (Node.js - shijitong-server)

采用典型的 MVC 分层结构（简化版），提供 RESTful API 服务。

```
shijitong-server
├── node_modules                    // [依赖库] 包含 express, mysql2, multer 等运行时依赖
├── public                          // [静态资源] Web 服务器根目录
│   ├── images                      // 系统预置图片资源 (广告图、背景图)
│   └── uploads                     // 用户上传文件存储目录 (头像等)
├── app.js                          // [服务入口] 配置中间件、数据库连接池及路由分发
├── db_shijitong.sql                // [数据库脚本] 数据库初始化 DDL 语句
├── package.json                    // [项目元数据] 定义脚本命令与依赖版本
└── package-lock.json               // 依赖版本锁定文件

```

---

## 3. 数据库设计 (MySQL)

**文件：`db_shijitong.sql`**

```sql
CREATE DATABASE `db_shijitong`;
USE `db_shijitong`;

-- 用户表：存储用户鉴权信息及基础资料
CREATE TABLE `users` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `phone` VARCHAR(20) NOT NULL UNIQUE COMMENT '注册手机号，唯一索引',
  `password` VARCHAR(100) NOT NULL COMMENT '用户密码',
  `nickname` VARCHAR(50) DEFAULT '用户' COMMENT '用户昵称',
  `avatar` VARCHAR(255) DEFAULT '' COMMENT '头像资源的相对路径'
);

-- 笔记表：存储富文本笔记内容
CREATE TABLE `notes` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `user_id` INT NOT NULL COMMENT '关联用户ID',
  `title` VARCHAR(100) COMMENT '笔记标题',
  `content` TEXT COMMENT '笔记正文内容',
  `created_at` DATETIME DEFAULT NOW() COMMENT '创建时间'
);

-- 待办事项表：存储任务清单及状态
CREATE TABLE `todos` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `user_id` INT NOT NULL COMMENT '关联用户ID',
  `content` VARCHAR(255) NOT NULL COMMENT '待办事项描述',
  `status` TINYINT DEFAULT 0 COMMENT '状态标识：0-未完成，1-已完成',
  `created_at` DATETIME DEFAULT NOW() COMMENT '创建时间'
);

```

---

## 4. 后端服务实现

**文件：`shijitong-server/app.js`**

```jsx
const express = require('express');
const bodyParser = require('body-parser');
const mysql = require('mysql2');
const cors = require('cors');
const app = express();
const multer = require('multer');
const path = require('path');
const fs = require('fs');

// 初始化中间件：允许跨域请求与 JSON 请求体解析
app.use(cors());
app.use(bodyParser.json());
// 托管静态资源，用于前端访问图片文件
app.use(express.static('public'));

// 初始化文件上传目录，确保路径存在
const uploadDir = path.join(__dirname, 'public/uploads');
if (!fs.existsSync(uploadDir)) {
    fs.mkdirSync(uploadDir, { recursive: true });
}

/**
 * 全局请求日志中间件
 * 用于监控所有流入的 HTTP 请求，记录方法、URL 及参数，便于调试。
 */
app.use((req, res, next) => {
    console.log(`[REQUEST] ${req.method} ${req.url}`);
    if (req.method === 'POST' || req.method === 'PUT') {
        console.log('Body:', JSON.stringify(req.body));
    } else if (Object.keys(req.query).length > 0) {
        console.log('Query:', JSON.stringify(req.query));
    }
    next();
});

/**
 * 数据库连接池配置
 * 生产环境建议使用连接池 (createPool) 以提高并发性能。
 */
const db = mysql.createConnection({
    host: 'localhost',
    user: 'root',
    password: 'like20040525',
    database: 'db_shijitong'
});

db.connect((err) => {
    if (err) {
        console.error('[DB Error] Connection failed:', err);
    } else {
        console.log('[DB Info] MySQL Connected successfully.');
    }
});

// ================= 业务接口路由 =================

/**
 * 用户注册接口
 * 逻辑：先检查手机号是否存在，不存在则插入新记录。
 */
app.post('/api/register', (req, res) => {
    const { phone, password } = req.body;
    const sqlCheck = "SELECT * FROM users WHERE phone = ?";
    db.query(sqlCheck, [phone], (err, results) => {
        if (err) return res.json({ code: 500, msg: '服务器内部错误' });
        if (results.length > 0) {
            res.json({ code: 409, msg: '该手机号已注册，请直接登录' });
        } else {
            const sqlInsert = "INSERT INTO users (phone, password) VALUES (?, ?)";
            db.query(sqlInsert, [phone, password], (insertErr, result) => {
                if (insertErr) return res.json({ code: 500, msg: '注册失败' });
                console.log(`[Register] Success. New ID: ${result.insertId}`);
                res.json({ code: 200, msg: '注册成功', data: { id: result.insertId, phone } });
            });
        }
    });
});

/**
 * 用户登录接口
 * 逻辑：验证手机号与密码匹配性，返回用户基础信息。
 */
app.post('/api/login', (req, res) => {
    const { phone, password } = req.body;
    const sqlCheck = "SELECT * FROM users WHERE phone = ?";
    db.query(sqlCheck, [phone], (err, results) => {
        if (err) return res.json({ code: 500, msg: '服务器内部错误' });
        if (results.length > 0) {
            const user = results[0];
            if (user.password === password) {
                console.log(`[Login] Success. User ID: ${user.id}`);
                res.json({ code: 200, msg: '登录成功', data: user });
            } else {
                res.json({ code: 401, msg: '密码错误' });
            }
        } else {
            res.json({ code: 404, msg: '账号不存在，请先注册' });
        }
    });
});

/**
 * 密码重置接口
 * 逻辑：需要同时验证用户ID和手机号，确保操作安全性。
 */
app.post('/api/forget', (req, res) => {
    const { userId, phone, newPass } = req.body;
    const sqlUpdate = "UPDATE users SET password = ? WHERE id = ? AND phone = ?";
    db.query(sqlUpdate, [newPass, userId, phone], (err, result) => {
        if (err) return res.json({ code: 500, msg: '服务器内部错误' });
        if (result.affectedRows > 0) {
            res.json({ code: 200, msg: '密码重置成功' });
        } else {
            res.json({ code: 400, msg: '信息验证失败(ID或手机号错误)' });
        }
    });
});

// --- 待办事项 (Todo) 模块 ---

app.get('/api/todos', (req, res) => {
    const { userId } = req.query;
    db.query("SELECT * FROM todos WHERE user_id = ?", [userId], (err, results) => {
        if (err) return res.json({ code: 500, data: [] });
        res.json({ code: 200, data: results });
    });
});

app.post('/api/todos', (req, res) => {
    const { userId, content } = req.body;
    db.query("INSERT INTO todos (user_id, content) VALUES (?, ?)", [userId, content], (err, result) => {
        if (err) return res.json({ code: 500 });
        res.json({ code: 200, data: { id: result.insertId } });
    });
});

app.post('/api/todos/update', (req, res) => {
    const { id, status } = req.body;
    db.query("UPDATE todos SET status = ? WHERE id = ?", [status, id], (err, result) => {
        if (err) return res.json({ code: 500 });
        res.json({ code: 200, msg: '更新成功' });
    });
});

app.post('/api/todos/update_content', (req, res) => {
    const { id, content } = req.body;
    db.query("UPDATE todos SET content = ? WHERE id = ?", [content, id], (err, result) => {
        if (err) return res.json({ code: 500, msg: '服务器错误' });
        res.json({ code: 200, msg: '内容更新成功' });
    });
});

app.post('/api/todos/delete', (req, res) => {
    const { id } = req.body;
    db.query("DELETE FROM todos WHERE id = ?", [id], (err, result) => {
        if (err) return res.json({ code: 500, msg: '服务器错误' });
        res.json({ code: 200, msg: '删除成功' });
    });
});

// --- 系统配置与统计模块 ---

app.get('/api/common/config', (req, res) => {
    res.json({
        code: 200,
        data: {
            splash_bg: '/images/splash.jpg',
            ads: [
                '/images/ad1.jpg',
                '/images/ad2.jpg',
                '/images/ad3.jpg'
            ]
        }
    });
});

app.get('/api/stats', (req, res) => {
    const { userId } = req.query;
    if (!userId) return res.json({ code: 400, msg: '缺少User ID' });

    // 聚合查询逻辑：分别统计笔记数、待办总数及完成数
    const sqlNotes = "SELECT COUNT(*) as count FROM notes WHERE user_id = ?";
    const sqlTodos = "SELECT COUNT(*) as count FROM todos WHERE user_id = ?";
    const sqlDone  = "SELECT COUNT(*) as count FROM todos WHERE user_id = ? AND status = 1";

    db.query(sqlNotes, [userId], (err, noteRes) => {
        if (err) return res.json({ code: 500, msg: '查询错误' });
        const noteCount = noteRes[0].count;

        db.query(sqlTodos, [userId], (err, todoRes) => {
            const todoTotal = todoRes[0].count;
            db.query(sqlDone, [userId], (err, doneRes) => {
                const todoDone = doneRes[0].count;
                res.json({
                    code: 200,
                    data: {
                        noteCount: noteCount,
                        todoTotal: todoTotal,
                        todoDone: todoDone,
                        todoPending: todoTotal - todoDone
                    }
                });
            });
        });
    });
});

// --- 云笔记 (Note) 模块 ---

app.get('/api/notes', (req, res) => {
    const { userId } = req.query;
    db.query("SELECT * FROM notes WHERE user_id = ? ORDER BY created_at DESC", [userId], (err, results) => {
        if (err) return res.json({ code: 500, msg: '查询失败', data: [] });
        res.json({ code: 200, data: results });
    });
});

app.post('/api/notes', (req, res) => {
    const { userId, title, content } = req.body;
    let finalTitle = title;
    if (!finalTitle && content) {
        finalTitle = content.substring(0, 10);
    }
    db.query("INSERT INTO notes (user_id, title, content) VALUES (?, ?, ?)",
        [userId, finalTitle, content],
        (err, result) => {
            if (err) return res.json({ code: 500, msg: '保存失败: ' + err.message });
            res.json({ code: 200, msg: '保存成功' });
    });
});

app.post('/api/notes/update', (req, res) => {
    const { id, title, content } = req.body;
    let finalTitle = title;
    if (!finalTitle && content) {
        finalTitle = content.substring(0, 10);
    }
    const sql = "UPDATE notes SET title = ?, content = ? WHERE id = ?";
    db.query(sql, [finalTitle, content, id], (err, result) => {
        if (err) return res.json({ code: 500, msg: '更新失败' });
        res.json({ code: 200, msg: '更新成功' });
    });
});

app.post('/api/notes/delete', (req, res) => {
    const { id } = req.body;
    const sql = "DELETE FROM notes WHERE id = ?";
    db.query(sql, [id], (err, result) => {
        if (err) return res.json({ code: 500, msg: '删除失败' });
        res.json({ code: 200, msg: '删除成功' });
    });
});

// --- 文件上传模块 ---

const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'public/uploads/');
    },
    filename: (req, file, cb) => {
        // 使用时间戳+随机数防止文件名冲突
        const uniqueSuffix = Date.now() + Math.round(Math.random() * 1E9);
        cb(null, uniqueSuffix + path.extname(file.originalname));
    }
});
const upload = multer({ storage: storage });

app.post('/api/user/avatar', upload.single('avatar'), (req, res) => {
    const userId = req.body.userId;
    const file = req.file;
    if (!file) return res.json({ code: 400, msg: '请选择图片' });
    if (!userId) return res.json({ code: 400, msg: '缺少 UserID' });

    const avatarUrl = `/uploads/${file.filename}`;
    const sql = "UPDATE users SET avatar = ? WHERE id = ?";
    db.query(sql, [avatarUrl, userId], (err, result) => {
        if (err) return res.json({ code: 500, msg: '数据库更新失败' });
        res.json({
            code: 200,
            msg: '头像上传成功',
            data: { avatar: avatarUrl }
        });
    });
});

app.get('/api/user/info', (req, res) => {
    const { userId } = req.query;
    db.query("SELECT id, phone, avatar, nickname FROM users WHERE id = ?", [userId], (err, results) => {
        if (err || results.length === 0) return res.json({ code: 404 });
        res.json({ code: 200, data: results[0] });
    });
});

app.post('/api/user/nickname', (req, res) => {
    const { userId, nickname } = req.body;
    if (!userId || !nickname) return res.json({ code: 400, msg: '参数不完整' });
    const sql = "UPDATE users SET nickname = ? WHERE id = ?";
    db.query(sql, [nickname, userId], (err, result) => {
        if (err) return res.json({ code: 500, msg: '数据库更新失败' });
        res.json({ code: 200, msg: '修改成功' });
    });
});

// 监听端口
app.listen(3000, () => {
    console.log('Server running at <http://127.0.0.1:3000>');
});

```

---

## 5. 前端客户端实现 (HarmonyOS)

### 5.1 公共基础设施层

**文件：`src/main/ets/common/constants/Api.ets`**

```tsx
/**
 * API 接口常量定义类
 * 集中管理所有后端服务接口地址，便于环境切换与维护。
 */
export class Api {
  // 服务端基础配置，需根据实际网络环境（如模拟器宿主IP）调整
  static readonly SERVER_IP: string = 'http://[本机地址]:3000';
  static readonly BASE_URL: string = Api.SERVER_IP + '/api';

  // 账户相关接口
  static readonly LOGIN = Api.BASE_URL + '/login';
  static readonly REGISTER = Api.BASE_URL + '/register';
  static readonly FORGET_PASS = Api.BASE_URL + '/forget';
  static readonly UPDATE_NICKNAME = Api.BASE_URL + '/user/nickname';
  static readonly UPLOAD_AVATAR = Api.BASE_URL + '/user/avatar';
  static readonly GET_USER_INFO = Api.BASE_URL + '/user/info';

  // 待办事项接口
  static readonly GET_TODOS = Api.BASE_URL + '/todos';
  static readonly ADD_TODO = Api.BASE_URL + '/todos';
  static readonly UPDATE_TODO = Api.BASE_URL + '/todos/update';
  static readonly UPDATE_TODO_CONTENT = Api.BASE_URL + '/todos/update_content';
  static readonly DELETE_TODO = Api.BASE_URL + '/todos/delete';

  // 云笔记接口
  static readonly GET_NOTES = Api.BASE_URL + '/notes';
  static readonly ADD_NOTE = Api.BASE_URL + '/notes';
  static readonly UPDATE_NOTE = Api.BASE_URL + '/notes/update';
  static readonly DELETE_NOTE = Api.BASE_URL + '/notes/delete';

  // 系统公共接口
  static readonly GET_CONFIG = Api.BASE_URL + '/common/config';
  static readonly GET_STATS = Api.BASE_URL + '/stats';
}

```

**文件：`src/main/ets/common/utils/HttpUtil.ets`**

```tsx
import http from '@ohos.net.http';

/**
 * 标准网络响应数据结构
 */
export interface ResponseResult {
  code: number;
  msg?: string;
  data?: Record<string, Object>;
}

/**
 * HTTP 请求工具类
 * 封装 @ohos.net.http 模块，提供统一的异步 GET/POST 方法。
 * 针对 ArkTS 严格模式进行了类型适配，自动处理 JSON 解析。
 */
export class HttpUtil {

  /**
   * 发送 GET 请求
   * @param url 请求地址
   * @returns Promise 返回解析后的 ResponseResult 对象，请求失败返回 null
   */
  static async get(url: string): Promise<ResponseResult | null> {
    let httpRequest = http.createHttp();
    try {
      let result = await httpRequest.request(url, {
        method: http.RequestMethod.GET,
        expectDataType: http.HttpDataType.STRING
      });
      // 严格模式下，需显式声明类型断言
      return JSON.parse(result.result.toString()) as ResponseResult;
    } catch (err) {
      console.error('Http Get Error:', JSON.stringify(err as Object));
      return null;
    }
  }

  /**
   * 发送 POST 请求
   * @param url 请求地址
   * @param data 请求体数据 (JSON)
   * @returns Promise 返回解析后的 ResponseResult 对象
   */
  static async post(url: string, data: Record<string, Object>): Promise<ResponseResult | null> {
    let httpRequest = http.createHttp();
    try {
      let result = await httpRequest.request(url, {
        method: http.RequestMethod.POST,
        extraData: data,
        header: { 'Content-Type': 'application/json' },
        expectDataType: http.HttpDataType.STRING
      });
      return JSON.parse(result.result.toString()) as ResponseResult;
    } catch (err) {
      console.error('Http Post Error:', JSON.stringify(err as Object));
      return null;
    }
  }
}

```

### 5.2 数据持久化与同步层

**文件：`src/main/ets/common/database/SQLiteHelper.ets`**

```tsx
import relationalStore from '@ohos.data.relationalStore';
import common from '@ohos.app.ability.common';
import { TodoModel } from '../../model/TodoModel';
import { NoteModel } from '../../model/NoteModel';

// 服务端数据模型定义，用于同步时的类型映射
export interface ServerTodoItem {
  id: number;
  content: string;
  status: number;
}

export interface ServerNoteItem {
  id: number;
  user_id: number;
  title: string;
  content: string;
}

/**
 * SQLite 数据库操作辅助类
 * 负责管理应用本地数据库 (RDB)，提供 CRUD 接口。
 * 核心功能：处理本地数据与服务端数据的同步逻辑（Conflict Resolution）。
 */
export class SQLiteHelper {
  private rdbStore: relationalStore.RdbStore | null = null;
  private tableNameTodo: string = 'local_todos';
  private tableNameNote: string = 'local_notes';

  /**
   * 初始化数据库
   * 在应用启动时调用，创建本地持久化文件及表结构。
   * 安全级别设置为 S1 (SecurityLevel.S1)，确保基本安全性。
   */
  async initDB(context: common.UIAbilityContext): Promise<void> {
    const config: relationalStore.StoreConfig = {
      name: 'ShiJiTong.db',
      securityLevel: relationalStore.SecurityLevel.S1
    };

    // 待办事项表：包含 server_id 字段用于标记同步状态 (0 为未同步)
    const sqlTodo = `CREATE TABLE IF NOT EXISTS ${this.tableNameTodo} (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      server_id INTEGER,
      content TEXT,
      status INTEGER
    )`;

    // 笔记表
    const sqlNote = `CREATE TABLE IF NOT EXISTS ${this.tableNameNote} (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      server_id INTEGER,
      user_id INTEGER,
      title TEXT,
      content TEXT
    )`;

    try {
      this.rdbStore = await relationalStore.getRdbStore(context, config);
      if (this.rdbStore) {
        await this.rdbStore.executeSql(sqlTodo);
        await this.rdbStore.executeSql(sqlNote);
        console.info('[SQLite] DB Init Success');
      }
    } catch (err) {
      console.error(`[SQLite] DB Init Failed: ${JSON.stringify(err)}`);
    }
  }

  // ==================== 待办事项 (Todo) 操作 ====================

  /**
   * 插入新的待办事项
   */
  async insert(todo: TodoModel): Promise<void> {
    if (!this.rdbStore) return;
    const valueBucket: relationalStore.ValuesBucket = {
      content: todo.content,
      server_id: todo.server_id,
      status: todo.status
    };
    await this.rdbStore.insert(this.tableNameTodo, valueBucket);
  }

  /**
   * 更新本地待办内容
   */
  async updateTodo(todo: TodoModel): Promise<void> {
    if (!this.rdbStore) return;
    const valueBucket: relationalStore.ValuesBucket = {
      content: todo.content,
      server_id: todo.server_id,
      status: todo.status
    };
    let predicates = new relationalStore.RdbPredicates(this.tableNameTodo);
    predicates.equalTo('id', todo.id);
    await this.rdbStore.update(valueBucket, predicates);
  }

  /**
   * 查询所有本地待办数据
   */
  async queryAll(): Promise<Array<TodoModel>> {
    if (!this.rdbStore) return [];
    let predicates = new relationalStore.RdbPredicates(this.tableNameTodo);
    let resultSet = await this.rdbStore.query(predicates);
    let list: Array<TodoModel> = [];
    while (resultSet.goToNextRow()) {
      list.push(new TodoModel(
        resultSet.getString(resultSet.getColumnIndex('content')),
        resultSet.getLong(resultSet.getColumnIndex('server_id')),
        resultSet.getLong(resultSet.getColumnIndex('status')),
        resultSet.getLong(resultSet.getColumnIndex('id'))
      ));
    }
    resultSet.close();
    return list;
  }

  /**
   * 获取所有未同步到服务端的待办事项
   * 条件：server_id 等于 0
   */
  async getUnsyncedTodos(): Promise<Array<TodoModel>> {
    if (!this.rdbStore) return [];
    let predicates = new relationalStore.RdbPredicates(this.tableNameTodo);
    predicates.equalTo('server_id', 0);
    let resultSet = await this.rdbStore.query(predicates);
    let list: Array<TodoModel> = [];
    while (resultSet.goToNextRow()) {
      list.push(new TodoModel(
        resultSet.getString(resultSet.getColumnIndex('content')),
        0,
        resultSet.getLong(resultSet.getColumnIndex('status')),
        resultSet.getLong(resultSet.getColumnIndex('id'))
      ));
    }
    resultSet.close();
    return list;
  }

  /**
   * 待办事项全量同步覆盖
   * 策略：
   * 1. 保留所有 server_id = 0 (本地新建未上传) 的数据。
   * 2. 删除所有 server_id != 0 的数据。
   * 3. 插入服务端拉取的最新数据。
   * 结果：本地数据库 = 本地未同步数据 + 云端最新数据。
   */
  async syncOverwrite(serverList: Array<ServerTodoItem>): Promise<void> {
    if (!this.rdbStore) return;
    this.rdbStore.beginTransaction();
    try {
      let predicates = new relationalStore.RdbPredicates(this.tableNameTodo);
      predicates.notEqualTo('server_id', 0);
      await this.rdbStore.delete(predicates);

      for (let item of serverList) {
        const bucket: relationalStore.ValuesBucket = {
          content: item.content,
          server_id: item.id,
          status: item.status
        };
        await this.rdbStore.insert(this.tableNameTodo, bucket);
      }
      this.rdbStore.commit();
    } catch (e) {
      this.rdbStore.rollBack();
    }
  }

  // ==================== 云笔记 (Note) 操作 ====================

  /**
   * 插入笔记记录
   */
  async insertNote(note: NoteModel): Promise<void> {
    if (!this.rdbStore) return;
    const bucket: relationalStore.ValuesBucket = {
      server_id: note.server_id,
      user_id: note.user_id,
      title: note.title,
      content: note.content
    };
    await this.rdbStore.insert(this.tableNameNote, bucket);
  }

  /**
   * 更新笔记记录
   */
  async updateNote(note: NoteModel): Promise<void> {
    if (!this.rdbStore) return;
    const bucket: relationalStore.ValuesBucket = {
      title: note.title,
      content: note.content,
      server_id: note.server_id
    };
    let predicates = new relationalStore.RdbPredicates(this.tableNameNote);
    predicates.equalTo('id', note.id);
    await this.rdbStore.update(bucket, predicates);
  }

  /**
   * 删除笔记记录
   */
  async deleteNote(id: number): Promise<void> {
    if (!this.rdbStore) return;
    let predicates = new relationalStore.RdbPredicates(this.tableNameNote);
    predicates.equalTo('id', id);
    await this.rdbStore.delete(predicates);
  }

  /**
   * 查询所有笔记 (按 ID 倒序)
   */
  async queryAllNotes(): Promise<Array<NoteModel>> {
    if (!this.rdbStore) return [];
    let predicates = new relationalStore.RdbPredicates(this.tableNameNote);
    predicates.orderByDesc('id');
    let resultSet = await this.rdbStore.query(predicates);
    let list: Array<NoteModel> = [];
    while (resultSet.goToNextRow()) {
      list.push(new NoteModel(
        resultSet.getString(resultSet.getColumnIndex('content')),
        resultSet.getString(resultSet.getColumnIndex('title')),
        resultSet.getLong(resultSet.getColumnIndex('user_id')),
        resultSet.getLong(resultSet.getColumnIndex('server_id')),
        resultSet.getLong(resultSet.getColumnIndex('id'))
      ));
    }
    resultSet.close();
    return list;
  }

  /**
   * 获取未同步的笔记
   */
  async getUnsyncedNotes(): Promise<Array<NoteModel>> {
    if (!this.rdbStore) return [];
    let predicates = new relationalStore.RdbPredicates(this.tableNameNote);
    predicates.equalTo('server_id', 0);
    let resultSet = await this.rdbStore.query(predicates);
    let list: Array<NoteModel> = [];
    while (resultSet.goToNextRow()) {
      list.push(new NoteModel(
        resultSet.getString(resultSet.getColumnIndex('content')),
        resultSet.getString(resultSet.getColumnIndex('title')),
        resultSet.getLong(resultSet.getColumnIndex('user_id')),
        0,
        resultSet.getLong(resultSet.getColumnIndex('id'))
      ));
    }
    resultSet.close();
    return list;
  }

  /**
   * 笔记全量同步覆盖
   * 逻辑与待办事项同步一致。
   */
  async syncOverwriteNotes(serverList: Array<ServerNoteItem>): Promise<void> {
    if (!this.rdbStore) return;
    this.rdbStore.beginTransaction();
    try {
      let predicates = new relationalStore.RdbPredicates(this.tableNameNote);
      predicates.notEqualTo('server_id', 0);
      await this.rdbStore.delete(predicates);

      for (let item of serverList) {
        const bucket: relationalStore.ValuesBucket = {
          server_id: item.id,
          user_id: item.user_id,
          title: item.title,
          content: item.content
        };
        await this.rdbStore.insert(this.tableNameNote, bucket);
      }
      this.rdbStore.commit();
    } catch (e) {
      this.rdbStore.rollBack();
    }
  }
}

export default new SQLiteHelper();

```

### 5.3 领域模型层

**文件：`src/main/ets/model/NoteModel.ets`**

```tsx
/**
 * 笔记数据模型
 * 充血模型，包含数据实体与构造逻辑，适配 SQLite 查询结果的封装。
 */
export class NoteModel {
  id: number = 0;          // 本地数据库主键 (Local ID)
  server_id: number = 0;   // 服务端数据库主键 (Server ID)，0 表示该数据尚未同步到云端
  user_id: number = 0;     // 所属用户 ID
  title: string = '';      // 笔记标题
  content: string = '';    // 笔记内容

  constructor(content: string, title: string, userId: number, serverId: number = 0, id: number = 0) {
    this.content = content;
    this.title = title;
    this.user_id = userId;
    this.server_id = serverId;
    this.id = id;
  }
}

```

**文件：`src/main/ets/model/TodoModel.ets`**

```tsx
/**
 * 待办事项数据模型
 */
export class TodoModel {
  id: number = 0;          // 本地 SQLite 主键
  server_id: number = 0;   // 服务端 ID
  content: string = '';    // 待办内容
  status: number = 0;      // 状态：0=进行中，1=已完成

  constructor(content: string, serverId: number = 0, status: number = 0, id: number = 0) {
    this.content = content;
    this.server_id = serverId;
    this.status = status;
    this.id = id;
  }
}

```

### 5.4 核心业务逻辑与视图层

**文件：`src/main/ets/entryability/EntryAbility.ets`**

```tsx
import { AbilityConstant, ConfigurationConstant, UIAbility, Want } from '@kit.AbilityKit';
import { hilog } from '@kit.PerformanceAnalysisKit';
import { window } from '@kit.ArkUI';
import SQLiteHelper from '../common/database/SQLiteHelper';

const DOMAIN = 0x0000;

/**
 * 应用程序入口 Ability
 * 负责应用生命周期管理、数据库初始化以及窗口的系统级配置。
 */
export default class EntryAbility extends UIAbility {
  async onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): Promise<void> {
    this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET);

    // 初始化本地 SQLite 数据库，确保在页面加载前数据库就绪
    try {
      await SQLiteHelper.initDB(this.context);
    } catch (error) {
      hilog.error(DOMAIN, 'AppTag', 'SQLite Init Failed');
    }
  }

  async onWindowStageCreate(windowStage: window.WindowStage): Promise<void> {
    try {
      let windowClass: window.Window = await windowStage.getMainWindow();

      // 设置应用为全屏沉浸式体验
      await windowClass.setWindowLayoutFullScreen(true);

      // 计算系统状态栏和导航栏的避让区域高度，并存储到全局状态中
      let type = window.AvoidAreaType.TYPE_SYSTEM;
      let avoidArea = windowClass.getWindowAvoidArea(type);
      AppStorage.setOrCreate('topHeight', avoidArea.topRect.height);

    } catch (exception) {
      hilog.error(DOMAIN, 'AppTag', 'Failed to set full screen.');
    }

    // 加载首页面 (启动页)
    windowStage.loadContent('pages/SplashPage');
  }

  onDestroy(): void {}
  onWindowStageDestroy(): void {}
  onForeground(): void {}
  onBackground(): void {}
}

```

**文件：`src/main/ets/pages/SplashPage.ets`**

```tsx
import router from '@ohos.router';
import { HttpUtil } from '../common/utils/HttpUtil';
import { Api } from '../common/constants/Api';

interface ConfigData {
  splash_bg: string;
}

/**
 * 启动页组件
 * 展示品牌形象，并在后台预拉取全局配置信息。
 */
@Entry
@Component
struct SplashPage {
  @State count: number = 3; // 倒计时秒数
  @State bgUrl: string = ''; // 背景图 URL
  private timerId: number = -1;

  aboutToAppear() {
    this.fetchConfig();
    // 启动倒计时，结束后跳转登录页
    this.timerId = setInterval(() => {
      this.count--;
      if (this.count <= 0) {
        this.jumpToLogin();
      }
    }, 1000);
  }

  /**
   * 从服务端获取配置信息 (如背景图)
   */
  async fetchConfig() {
    const res = await HttpUtil.get(Api.GET_CONFIG);
    if (res && res.code === 200 && res.data) {
      // 严格模式：进行类型断言
      const data = res.data as Object as ConfigData;
      this.bgUrl = Api.SERVER_IP + data.splash_bg;
    }
  }

  jumpToLogin() {
    clearInterval(this.timerId);
    router.replaceUrl({ url: 'pages/LoginPage' });
  }

  build() {
    Stack({ alignContent: Alignment.Top }) {
      // 层级 1: 背景图片
      if (this.bgUrl) {
        Image(this.bgUrl).width('100%').height('100%').objectFit(ImageFit.Cover)
      } else {
        Column().width('100%').height('100%').backgroundColor(Color.White)
      }

      // 层级 2: 应用标题与标语
      Column() {
        Text("事记通")
          .fontSize(40).fontWeight(FontWeight.Bold)
          .fontColor(this.bgUrl ? Color.White : '#333')
          .margin({ top: 350 })

        Text("一站式事务管理")
          .fontSize(16).fontWeight(FontWeight.Medium)
          .fontColor(this.bgUrl ? Color.White : Color.Gray)
          .margin({ top: 15 }).letterSpacing(4)
      }.width('100%')

      // 层级 3: 跳过按钮
      Row({ space: 10 }) {
        Text(`${this.count}秒后跳过`)
          .fontSize(13).fontColor(Color.White)
          .backgroundColor('rgba(0, 0, 0, 0.4)').borderRadius(20)
          .padding({ left: 15, right: 15, top: 8, bottom: 8 })

        Text('跳过')
          .fontSize(13).fontColor(Color.White)
          .backgroundColor('rgba(0, 0, 0, 0.4)').borderRadius(20)
          .padding({ left: 20, right: 20, top: 8, bottom: 8 })
          .onClick(() => { this.jumpToLogin(); })
      }
      .width('100%').justifyContent(FlexAlign.End).padding({ top: 60, right: 20 })
    }.width('100%').height('100%')
  }
}

```

**文件：`src/main/ets/pages/LoginPage.ets`**

```tsx
import router from '@ohos.router';
import { HttpUtil } from '../common/utils/HttpUtil';
import { Api } from '../common/constants/Api';
import promptAction from '@ohos.promptAction';
import { PrivacyDialog } from '../view/PrivacyDialog';

/**
 * 登录页组件
 * 负责用户身份认证，并在登录成功后持久化 Session 信息。
 */
@Entry
@Component
struct LoginPage {
  @State phone: string = '';
  @State password: string = '';
  @State isShowPass: boolean = false;

  dialogController: CustomDialogController = new CustomDialogController({
    builder: PrivacyDialog(),
    autoCancel: true,
    alignment: DialogAlignment.Center
  });

  async handleLogin() {
    if (!this.phone || !this.password) {
      promptAction.showToast({ message: '请输入账号和密码' });
      return;
    }
    let params: Record<string, Object> = { 'phone': this.phone, 'password': this.password };
    const res = await HttpUtil.post(Api.LOGIN, params);

    if (res && res.code === 200 && res.data) {
      const userId = res.data['id'] as number;
      // 登录成功，将关键信息存入 AppStorage 全局状态中
      AppStorage.setOrCreate('phone', this.phone);
      AppStorage.setOrCreate('userId', userId);
      promptAction.showToast({ message: '登录成功' });
      router.replaceUrl({ url: 'pages/MainPage' });
    } else {
      promptAction.showToast({ message: res?.msg || '登录失败' });
    }
  }

  // 样式复用装饰器
  @Styles inputStyle() {
    .width('85%').height(55).backgroundColor(Color.Transparent)
    .borderRadius(30).border({ width: 1, color: '#8c9fad' })
    .padding({ left: 20, right: 20 })
  }

  build() {
    Column() {
      Text('欢迎').fontSize(40).fontWeight(FontWeight.Normal).fontColor('#333').margin({ top: 80, bottom: 60 })

      TextInput({ placeholder: '请输入手机号', text: this.phone })
        .inputStyle().onChange((val) => this.phone = val).margin({ bottom: 20 })

      Column() {
        TextInput({ placeholder: '请输入密码', text: this.password })
          .type(this.isShowPass ? InputType.Normal : InputType.Password)
          .inputStyle().onChange((val) => this.password = val)

        Row() {
          Text('显示密码').fontSize(14).fontColor('#666').onClick(() => { this.isShowPass = !this.isShowPass; })
        }.width('85%').justifyContent(FlexAlign.End).margin({ top: 8 })
      }.margin({ bottom: 40 })

      Button('登录').width('85%').height(50).fontSize(18).backgroundColor('#90a4ce')
        .type(ButtonType.Capsule).onClick(() => this.handleLogin())

      Row() {
        Text('忘记密码').fontSize(14).fontColor('#333').onClick(() => router.pushUrl({ url: 'pages/ForgetPage' }))
        Text('|').margin({ left: 10, right: 10 }).fontColor('#ccc')
        Text('注册新账号').fontSize(14).fontColor('#333').onClick(() => router.pushUrl({ url: 'pages/RegisterPage' }))
      }.width('85%').justifyContent(FlexAlign.SpaceBetween).padding({ left: 20, right: 20 }).margin({ top: 20 })

      Blank()

      Column() {
        Text('请注意,登录即代表您同意我们的').fontSize(12).fontColor('#666')
        Text('《隐私条款》').fontSize(12).fontColor(Color.Blue).fontWeight(FontWeight.Bold)
          .onClick(() => { this.dialogController.open(); })
      }.margin({ bottom: 40 })
    }
    .width('100%').height('100%').backgroundColor('#F5F8FA')
  }
}

```

**文件：`src/main/ets/pages/RegisterPage.ets`**

```tsx
import router from '@ohos.router';
import { HttpUtil } from '../common/utils/HttpUtil';
import { Api } from '../common/constants/Api';
import promptAction from '@ohos.promptAction';
import { PrivacyDialog } from '../view/PrivacyDialog';

@Entry
@Component
struct RegisterPage {
  @State phone: string = '';
  @State password: string = '';
  @State isShowPass: boolean = false;
  topHeight: number = AppStorage.get<number>('topHeight') || 0;

  dialogController: CustomDialogController = new CustomDialogController({
    builder: PrivacyDialog(),
    autoCancel: true,
    alignment: DialogAlignment.Center
  });

  async handleRegister() {
    if (!this.phone || !this.password) {
      promptAction.showToast({ message: '请完善信息' });
      return;
    }
    let params: Record<string, Object> = { 'phone': this.phone, 'password': this.password };
    const res = await HttpUtil.post(Api.REGISTER, params);

    if (res && res.code === 200) {
      promptAction.showToast({ message: '注册成功，请登录' });
      router.back();
    } else {
      promptAction.showToast({ message: res?.msg || '注册失败' });
    }
  }

  @Styles inputStyle() {
    .width('85%').height(55).borderRadius(30)
    .border({ width: 1, color: '#8c9fad' }).padding({ left: 20 })
  }

  build() {
    Column() {
      Row().height(50)
      Text('注册').fontSize(40).fontColor('#333').margin({ top: 50, bottom: 60 })

      TextInput({ placeholder: '请输入手机号', text: this.phone })
        .inputStyle().onChange((val) => this.phone = val).margin({ bottom: 20 })

      Column() {
        TextInput({ placeholder: '请输入密码', text: this.password })
          .type(this.isShowPass ? InputType.Normal : InputType.Password)
          .inputStyle().onChange((val) => this.password = val)

        Row() {
          Text('显示密码').fontSize(14).fontColor('#666').onClick(() => this.isShowPass = !this.isShowPass)
        }.width('85%').justifyContent(FlexAlign.End).margin({ top: 8 })
      }.margin({ bottom: 40 })

      Button('注册').width('85%').height(50).fontSize(18).backgroundColor('#90a4ce')
        .type(ButtonType.Capsule).onClick(() => this.handleRegister())

      Blank()

      Column() {
        Text('请注意,注册即代表您同意我们的').fontSize(12).fontColor('#666')
        Text('《隐私条款》').fontSize(12).fontColor(Color.Blue).fontWeight(FontWeight.Bold)
          .onClick(() => this.dialogController.open())
      }.margin({ bottom: 40 })
    }
    .width('100%').height('100%').backgroundColor('#F5F8FA')
    .padding({ top: px2vp(this.topHeight) })
  }
}

```

**文件：`src/main/ets/pages/ForgetPage.ets`**

```tsx
import router from '@ohos.router';
import { HttpUtil } from '../common/utils/HttpUtil';
import { Api } from '../common/constants/Api';
import promptAction from '@ohos.promptAction';

/**
 * 忘记密码页
 * 这是一个独立页面，用于未登录状态下的密码找回。
 */
@Entry
@Component
struct ForgetPage {
  @State userId: string = '';
  @State phone: string = '';
  @State newPass: string = '';
  @State confirmPass: string = '';
  @State isShowPass: boolean = false;
  topHeight: number = AppStorage.get<number>('topHeight') || 0;

  async handleReset() {
    if (!this.userId || !this.phone || !this.newPass || !this.confirmPass) {
      promptAction.showToast({ message: '请填写完整信息' });
      return;
    }
    if (this.newPass !== this.confirmPass) {
      promptAction.showToast({ message: '两次输入的密码不一致' });
      return;
    }
    let params: Record<string, Object> = {
      'userId': this.userId, 'phone': this.phone, 'newPass': this.newPass
    };
    const res = await HttpUtil.post(Api.FORGET_PASS, params);
    if (res && res.code === 200) {
      promptAction.showToast({ message: '密码重置成功，请重新登录' });
      router.back();
    } else {
      promptAction.showToast({ message: res?.msg || '重置失败' });
    }
  }

  @Styles inputStyle() {
    .width('85%').height(55).borderRadius(30).backgroundColor(Color.Transparent)
    .border({ width: 1, color: '#8c9fad' }).padding({ left: 20, right: 20 })
  }

  build() {
    Column() {
      Text('忘记密码').fontSize(36).fontColor('#333').fontWeight(FontWeight.Normal).margin({ top: 60, bottom: 40 })

      Column({ space: 20 }) {
        TextInput({ placeholder: '请输入用户ID', text: this.userId })
          .inputStyle().type(InputType.Number).onChange((val) => this.userId = val)

        TextInput({ placeholder: '请输入注册手机号', text: this.phone })
          .inputStyle().type(InputType.PhoneNumber).onChange((val) => this.phone = val)

        TextInput({ placeholder: '请输入新密码', text: this.newPass })
          .type(this.isShowPass ? InputType.Normal : InputType.Password)
          .inputStyle().onChange((val) => this.newPass = val)

        TextInput({ placeholder: '请再次输入新密码', text: this.confirmPass })
          .type(this.isShowPass ? InputType.Normal : InputType.Password)
          .inputStyle().onChange((val) => this.confirmPass = val)
      }

      Row() {
        Text('显示密码').fontSize(14).fontColor('#666').onClick(() => this.isShowPass = !this.isShowPass)
      }.width('85%').justifyContent(FlexAlign.End).margin({ top: 10, bottom: 40 })

      Button('确认修改').width('85%').height(50).fontSize(18).backgroundColor('#90a4ce')
        .type(ButtonType.Capsule).onClick(() => this.handleReset())
    }
    .width('100%').height('100%').backgroundColor('#F5F8FA')
    .padding({ top: px2vp(this.topHeight) })
  }
}

```

**文件：`src/main/ets/pages/MainPage.ets`**

```tsx
import router from '@ohos.router';
import { HttpUtil } from '../common/utils/HttpUtil';
import { Api } from '../common/constants/Api';
import { TodoPage } from './TodoPage';
import { NoteModel } from '../model/NoteModel';
import picker from '@ohos.file.picker';
import request from '@ohos.request';
import { promptAction } from '@kit.ArkUI';
import { BusinessError } from '@kit.BasicServicesKit';
import SQLiteHelper, { ServerNoteItem } from '../common/database/SQLiteHelper';

// 统计数据结构接口
interface StatsData {
  noteCount: number;
  todoTotal: number;
  todoDone: number;
  todoPending: number;
}
// 配置数据结构接口
interface ConfigData {
  splash_bg: string;
  ads: Array<string>;
}

// 内部组件：修改密码弹窗
@CustomDialog
struct ChangePassDialog {
  controller: CustomDialogController;
  @State newPass: string = '';
  @State confirmPass: string = '';
  confirm: (newPass: string) => void = () => {};

  build() {
    Column() {
      Text('修改密码').fontSize(20).fontWeight(FontWeight.Bold).margin({ top: 20, bottom: 20 })
      TextInput({ placeholder: '请输入新密码', text: this.newPass }).type(InputType.Password).height(50).width('90%').margin({ bottom: 15 }).borderRadius(10).backgroundColor('#F5F5F5').padding({ left: 15 }).onChange((val) => this.newPass = val)
      TextInput({ placeholder: '请确认新密码', text: this.confirmPass }).type(InputType.Password).height(50).width('90%').margin({ bottom: 20 }).borderRadius(10).backgroundColor('#F5F5F5').padding({ left: 15 }).onChange((val) => this.confirmPass = val)
      Row() {
        Button('取消').onClick(() => { this.controller.close(); }).backgroundColor('#F0F0F0').fontColor('#333').width('40%').margin({ right: 10 })
        Button('确认修改').onClick(() => {
          if (!this.newPass || !this.confirmPass) { promptAction.showToast({ message: '密码不能为空' }); return; }
          if (this.newPass !== this.confirmPass) { promptAction.showToast({ message: '两次密码输入不一致' }); return; }
          this.confirm(this.newPass);
          this.controller.close();
        }).backgroundColor('#007DFF').width('40%')
      }.margin({ bottom: 20 })
    }.width('85%').backgroundColor(Color.White).borderRadius(16)
  }
}

// 内部组件：修改昵称弹窗
@CustomDialog
struct ChangeNameDialog {
  controller: CustomDialogController;
  @State newName: string = '';
  confirm: (newName: string) => void = () => {};

  build() {
    Column() {
      Text('修改用户名').fontSize(20).fontWeight(FontWeight.Bold).margin({ top: 20, bottom: 20 })
      TextInput({ placeholder: '请输入新的用户名', text: this.newName }).height(50).width('90%').margin({ bottom: 20 }).borderRadius(10).backgroundColor('#F5F5F5').padding({ left: 15 }).onChange((val) => this.newName = val)
      Row() {
        Button('取消').onClick(() => { this.controller.close(); }).backgroundColor('#F0F0F0').fontColor('#333').width('40%').margin({ right: 10 })
        Button('保存').onClick(() => {
          if (this.newName.trim() === '') { promptAction.showToast({ message: '用户名不能为空' }); return; }
          this.confirm(this.newName);
          this.controller.close();
        }).backgroundColor('#007DFF').width('40%')
      }.margin({ bottom: 20 })
    }.width('85%').backgroundColor(Color.White).borderRadius(16)
  }
}

/**
 * 主页面 (Dashboard)
 * 采用 Tabs 布局，包含四个核心子模块：云笔记、待办事项、数据统计、个人中心。
 * 集成了新手引导遮罩层，基于 PersistentStorage 判断是否已读。
 */
@Entry
@Component
struct MainPage {
  @State currentIndex: number = 0;
  private controller: TabsController = new TabsController();
  userId: number = AppStorage.get<number>('userId') || 1;
  topHeight: number = AppStorage.get<number>('topHeight') || 0;

  @State stats: StatsData = { noteCount: 0, todoTotal: 0, todoDone: 0, todoPending: 0 };
  @State finishRate: number = 0;
  @State noteList: Array<NoteModel> = [];
  @State userAvatar: string = '';
  @State userName: string = `用户 ${this.userId}`;
  @State isShowGuide: boolean = false;
  @State guideImages: Array<string> = [];

  passDialogController: CustomDialogController = new CustomDialogController({
    builder: ChangePassDialog({ confirm: (newPass: string) => { this.handleChangePass(newPass); } }),
    autoCancel: true,
    alignment: DialogAlignment.Center
  });

  nameDialogController: CustomDialogController = new CustomDialogController({
    builder: ChangeNameDialog({ confirm: (newName: string) => { this.handleUpdateName(newName); } }),
    autoCancel: true,
    alignment: DialogAlignment.Center
  });

  onPageShow(): void {
    this.checkGuideStatus();
    // 每次显示页面时，根据当前 Tab 刷新数据
    if (this.currentIndex === 0) this.fetchNotes();
    if (this.currentIndex === 2) this.fetchStats();
    this.fetchUserInfo();
  }

  async fetchUserInfo(): Promise<void> {
    try {
      const res = await HttpUtil.get(`${Api.GET_USER_INFO}?userId=${this.userId}`);
      if (res && res.code === 200 && res.data) {
        const user = res.data as Record<string, Object>;
        const avatarPath = user['avatar'] as string;
        if (avatarPath) { this.userAvatar = Api.SERVER_IP + avatarPath; }
        const dbNickname = user['nickname'] as string;
        if (dbNickname) { this.userName = dbNickname; }
      }
    } catch (err) { console.error(`Fetch info failed: ${JSON.stringify(err as Object)}`); }
  }

  async handleUpdateName(newName: string): Promise<void> {
    try {
      let params: Record<string, Object> = { 'userId': this.userId, 'nickname': newName };
      const res = await HttpUtil.post(Api.UPDATE_NICKNAME, params);
      if (res && res.code === 200) {
        this.userName = newName;
        promptAction.showToast({ message: '用户名修改成功' });
      } else {
        promptAction.showToast({ message: '修改失败' });
      }
    } catch (err) { console.error(JSON.stringify(err as Object)); promptAction.showToast({ message: '网络异常' }); }
  }

  /**
   * 拉取笔记列表 (离线同步策略)
   * 1. 优先读取本地数据库 (Offline-First)。
   * 2. 上传本地未同步数据。
   * 3. 拉取全量云端数据覆盖本地。
   */
  async fetchNotes(): Promise<void> {
    try {
      this.noteList = await SQLiteHelper.queryAllNotes();
      const unsyncedNotes = await SQLiteHelper.getUnsyncedNotes();

      if (unsyncedNotes.length > 0) {
        for (let note of unsyncedNotes) {
          let params: Record<string, Object> = { 'userId': this.userId, 'title': note.title, 'content': note.content };
          await HttpUtil.post(Api.ADD_NOTE, params);
        }
      }

      const res = await HttpUtil.get(`${Api.GET_NOTES}?userId=${this.userId}`);
      if (res && res.code === 200 && res.data) {
        const serverList = res.data as Object as Array<ServerNoteItem>;
        await SQLiteHelper.syncOverwriteNotes(serverList);
        this.noteList = await SQLiteHelper.queryAllNotes();
      }
    } catch (err) { console.error(`Fetch notes failed: ${JSON.stringify(err as Object)}`); }
  }

  // 检查是否需要显示新手引导 (基于 PersistentStorage)
  async checkGuideStatus(): Promise<void> {
    const storageKey = `isGuideSeen_${this.userId}`;
    try {
      PersistentStorage.persistProp(storageKey, false);
      const hasSeen = AppStorage.get<boolean>(storageKey);
      if (!hasSeen) { await this.fetchConfig(); }
    } catch (err) { console.error(`Storage error: ${JSON.stringify(err as Object)}`); }
  }

  async fetchConfig(): Promise<void> {
    try {
      const res = await HttpUtil.get(Api.GET_CONFIG);
      if (res && res.code === 200 && res.data) {
        const config = res.data as Object as ConfigData;
        if (config.ads && config.ads.length > 0) {
          this.guideImages = config.ads.map((url: string) => Api.SERVER_IP + url);
          this.isShowGuide = true;
        }
      }
    } catch (err) { console.error(`Fetch config failed: ${JSON.stringify(err as Object)}`); }
  }

  handleFinishGuide() { this.isShowGuide = false; const storageKey = `isGuideSeen_${this.userId}`; AppStorage.setOrCreate(storageKey, true); }

  async handleAvatarClick(): Promise<void> {
    try {
      const photoSelectOptions = new picker.PhotoSelectOptions();
      photoSelectOptions.MIMEType = picker.PhotoViewMIMETypes.IMAGE_TYPE;
      photoSelectOptions.maxSelectNumber = 1;
      const photoViewPicker = new picker.PhotoViewPicker();
      const photoSelectResult = await photoViewPicker.select(photoSelectOptions);
      if (photoSelectResult.photoUris.length > 0) { const fileUri = photoSelectResult.photoUris[0]; this.uploadImage(fileUri); }
    } catch (err) { console.error(`Photo select failed: ${JSON.stringify(err as Object)}`); }
  }

  async uploadImage(fileUri: string): Promise<void> {
    try {
      const uploadConfig: request.UploadConfig = {
        url: Api.UPLOAD_AVATAR,
        header: { 'Content-Type': 'multipart/form-data' },
        method: 'POST',
        files: [ { filename: 'avatar.jpg', name: 'avatar', uri: fileUri, type: 'jpg' } ],
        data: [ { name: 'userId', value: this.userId.toString() } ]
      };
      const uploadTask = await request.uploadFile(getContext(this), uploadConfig);
      uploadTask.on('complete', () => { promptAction.showToast({ message: '上传成功' }); this.fetchUserInfo(); });
      uploadTask.on('fail', () => { promptAction.showToast({ message: '上传失败' }); });
    } catch (err) { console.error(`Upload failed: ${JSON.stringify(err as Object)}`); promptAction.showToast({ message: '上传出错' }); }
  }

  async fetchStats(): Promise<void> {
    try {
      const res = await HttpUtil.get(`${Api.GET_STATS}?userId=${this.userId}`);
      if (res && res.code === 200 && res.data) {
        const data = res.data as Object as StatsData;
        this.stats = data;
        if (this.stats.todoTotal > 0) { this.finishRate = Math.round((this.stats.todoDone / this.stats.todoTotal) * 100); } else { this.finishRate = 0; }
      }
    } catch (err) { console.error(`Fetch stats failed: ${JSON.stringify(err as Object)}`); }
  }

  async handleChangePass(newPass: string): Promise<void> {
    try {
      const userPhone = AppStorage.get<string>('phone') || '';
      let params: Record<string, Object> = { 'userId': this.userId, 'phone': userPhone, 'newPass': newPass };
      const res = await HttpUtil.post(Api.FORGET_PASS, params);
      if (res && res.code === 200) {
        promptAction.showToast({ message: '密码修改成功，请重新登录' });
        setTimeout(() => { this.handleLogout(); }, 1500);
      } else {
        promptAction.showToast({ message: res?.msg || '修改失败，请检查账号信息' });
      }
    } catch (err) { promptAction.showToast({ message: '网络请求异常' }); }
  }

  handleLogout(): void {
    AppStorage.setOrCreate('userId', 0);
    router.replaceUrl({ url: 'pages/LoginPage' }).catch((err: BusinessError) => { console.error(`Router error: ${JSON.stringify(err as Object)}`); });
  }

  @Builder TabBuilder(title: string, targetIndex: number) {
    Column() {
      Text(title).fontColor(this.currentIndex === targetIndex ? '#007DFF' : '#999').fontSize(14).fontWeight(this.currentIndex === targetIndex ? FontWeight.Medium : FontWeight.Normal).margin({ top: 5 })
    }.width('100%').height(50).justifyContent(FlexAlign.Center).onClick(() => { this.currentIndex = targetIndex; this.controller.changeIndex(targetIndex); })
  }

  @Builder MineCell(icon: Resource, title: string, onClick: () => void, isRed: boolean = false) {
    Row() {
      Image(icon).width(20).height(20).margin({ right: 15 });
      Text(title).fontSize(16).fontColor(isRed ? Color.Red : '#333').layoutWeight(1);
      Text('>').fontSize(14).fontColor('#ccc')
    }.width('100%').height(55).backgroundColor(Color.White).padding({ left: 20, right: 20 }).onClick(onClick)
  }

  build() {
    Stack() {
      Tabs({ barPosition: BarPosition.End, controller: this.controller }) {
        // Tab 0: 云笔记
        TabContent() {
          Stack({ alignContent: Alignment.BottomEnd }) {
            Column() {
              Row() { Text('蓝不蓝云笔记').fontSize(20).fontWeight(FontWeight.Medium).fontColor(Color.White) }.width('100%').height(56 + px2vp(this.topHeight)).backgroundColor('#007DFF').padding({ left: 20, top: px2vp(this.topHeight) }).alignItems(VerticalAlign.Center)
              List() {
                if (this.noteList.length === 0) { ListItem() { Text("暂无笔记，点击右下角添加").fontSize(14).fontColor(Color.Gray).width('100%').textAlign(TextAlign.Center).padding(20) } } else {
                  ForEach(this.noteList, (item: NoteModel, index: number) => {
                    ListItem() {
                      Column() { Text(`蓝不蓝云笔记${index + 1}`).fontSize(16).fontColor('#333').maxLines(1).textOverflow({ overflow: TextOverflow.Ellipsis }); Text(item.content).fontSize(12).fontColor(Color.Gray).maxLines(1).textOverflow({ overflow: TextOverflow.Ellipsis }).margin({ top: 5 }) }.width('100%').alignItems(HorizontalAlign.Start).padding(15).backgroundColor(Color.White).border({ width: { bottom: 1 }, color: '#f0f0f0' })
                    }.onClick(() => { router.pushUrl({ url: 'pages/NoteEditPage', params: { id: item.id, server_id: item.server_id, content: item.content } }).catch((err: BusinessError) => console.error(`Router error: ${JSON.stringify(err as Object)}`)); })
                  })
                }
              }.width('100%').layoutWeight(1).backgroundColor(Color.White)
            }.width('100%').height('100%')
            Button({ type: ButtonType.Circle, stateEffect: true }) { Text('+').fontSize(30).fontColor(Color.White).fontWeight(FontWeight.Lighter).margin({ bottom: 2 }) }.width(60).height(60).backgroundColor('#007DFF').shadow({ radius: 10, color: 'rgba(0,0,0,0.3)', offsetY: 5 }).margin({ right: 20, bottom: 30 }).onClick(() => { router.pushUrl({ url: 'pages/NoteEditPage' }).catch((err: BusinessError) => console.error(`Router error: ${JSON.stringify(err as Object)}`)); })
          }.width('100%').height('100%')
        }.tabBar(this.TabBuilder("云笔记", 0))

        // Tab 1: 待办事项
        TabContent() { Column() { TodoPage() }.padding({ top: px2vp(this.topHeight) }).backgroundColor('#F0F2F5') }.tabBar(this.TabBuilder("待办事项", 1))

        // Tab 2: 统计
        TabContent() {
          Column() {
            Text("数据统计").fontSize(24).fontWeight(FontWeight.Bold).margin({ top: 20, bottom: 20 }).width('90%')
            Row() { Column() { Text(this.stats.noteCount.toString()).fontSize(36).fontWeight(FontWeight.Bold).fontColor('#007DFF'); Text("累计笔记 (篇)").fontSize(14).fontColor(Color.Gray).margin({ top: 5 }) }.alignItems(HorizontalAlign.Start); Text("📝").fontSize(40) }.width('90%').padding(20).backgroundColor(Color.White).borderRadius(15).justifyContent(FlexAlign.SpaceBetween).margin({ bottom: 15 })
            Column() {
              Text("待办事项概览").fontSize(16).fontWeight(FontWeight.Bold).width('100%').margin({ bottom: 20 })
              Stack() { Circle({ width: 160, height: 160 }).fill('transparent').stroke('#F0F2F5').strokeWidth(15); Circle({ width: 160, height: 160 }).fill('transparent').stroke('#007DFF').strokeWidth(15).strokeDashArray([160 * 3.14 * this.finishRate / 100, 1000]).rotate({ angle: -90 }); Column() { Text(`${this.finishRate}%`).fontSize(36).fontWeight(FontWeight.Bold).fontColor('#007DFF'); Text("完成率").fontSize(14).fontColor(Color.Gray) } }.margin({ bottom: 30 })
              Row() { Column() { Text(this.stats.todoTotal.toString()).fontSize(20).fontWeight(FontWeight.Bold); Text("总计").fontSize(12).fontColor(Color.Gray) }.layoutWeight(1); Column() { Text(this.stats.todoDone.toString()).fontSize(20).fontWeight(FontWeight.Bold).fontColor(Color.Green); Text("已完成").fontSize(12).fontColor(Color.Gray) }.layoutWeight(1); Column() { Text(this.stats.todoPending.toString()).fontSize(20).fontWeight(FontWeight.Bold).fontColor(Color.Red); Text("未完成").fontSize(12).fontColor(Color.Gray) }.layoutWeight(1) }.width('100%')
            }.width('90%').padding(20).backgroundColor(Color.White).borderRadius(15)
          }.width('100%').height('100%').backgroundColor('#F0F2F5').padding({ top: px2vp(this.topHeight) })
        }.tabBar(this.TabBuilder("统计", 2))

        // Tab 3: 我的
        TabContent() {
          Column() {
            Column() {
              Stack({ alignContent: Alignment.BottomEnd }) {
                if (this.userAvatar && this.userAvatar.length > 0) { Image(this.userAvatar).width(80).height(80).borderRadius(40).objectFit(ImageFit.Cover) }
                else { Circle({ width: 80, height: 80 }).fill('#E0E0E0'); Text(this.userName.substring(0, 1)).fontSize(30).fontColor(Color.White).margin({ bottom: 25, right: 25 }) }
                Image($r('app.media.app_icon')).width(24).height(24).borderRadius(12).backgroundColor(Color.White)
              }.onClick(() => { this.handleAvatarClick().catch((err: BusinessError) => { console.error(`Avatar error: ${JSON.stringify(err as Object)}`); }); }).margin({ top: 40, bottom: 15 })
              Text(this.userName).fontSize(20).fontWeight(FontWeight.Bold).fontColor('#333')
              Text(`ID: ${this.userId}`).fontSize(14).fontColor('#999').margin({ top: 5 })
            }.width('100%').height(220).backgroundColor(Color.White).margin({ bottom: 15 }).justifyContent(FlexAlign.Center)

            Column() {
              this.MineCell($r('app.media.changeAvator'), '修改头像', () => { this.handleAvatarClick().catch((err: BusinessError) => console.error(JSON.stringify(err as Object))); })
              Divider().color('#F5F5F5').strokeWidth(1).padding({ left: 55 })

              this.MineCell($r('app.media.changeNickname'), '更换用户名', () => { this.nameDialogController.open(); })
              Divider().color('#F5F5F5').strokeWidth(1).padding({ left: 55 })

              this.MineCell($r('app.media.changePassword'), '修改密码', () => { this.passDialogController.open(); })
              Divider().color('#F5F5F5').strokeWidth(1).padding({ left: 55 })

              this.MineCell($r('app.media.aboutUs'), '关于我们', () => { promptAction.showToast({ message: 'ShiJiTong App v1.0.0' }) })
            }.backgroundColor(Color.White).margin({ bottom: 15 })

            Column() {
              this.MineCell($r('app.media.logout'), '退出登录', () => {
                try {
                  promptAction.showDialog({ title: '提示', message: '确定要退出登录吗？', buttons: [ { text: '取消', color: '#666666' }, { text: '确定', color: '#FF0000' } ] }).then((data) => { if (data.index === 1) { this.handleLogout(); } });
                } catch (err) { console.error(`Dialog error: ${JSON.stringify(err as Object)}`); }
              }, true)
            }.backgroundColor(Color.White)
          }.width('100%').height('100%').backgroundColor('#F0F2F5').padding({ top: px2vp(this.topHeight) })
        }.tabBar(this.TabBuilder("我的", 3))
      }
      .scrollable(false)
      .onChange((index) => {
        this.currentIndex = index;
        if (index === 0) { this.fetchNotes(); }
        if (index === 2) { this.fetchStats(); }
      })

      if (this.isShowGuide && this.guideImages.length > 0) {
        Stack() {
          Swiper() {
            ForEach(this.guideImages, (imgUrl: string, index: number) => {
              Stack({ alignContent: Alignment.Bottom }) {
                Image(imgUrl).width('100%').height('100%').objectFit(ImageFit.Cover)
                if (index === this.guideImages.length - 1) { Button('已知晓').width(150).height(50).fontSize(18).backgroundColor('#007DFF').fontColor(Color.White).margin({ bottom: 80 }).onClick(() => { this.handleFinishGuide(); }) }
              }.width('100%').height('100%')
            })
          }.width('100%').height('100%').loop(false).indicator(true)
        }.width('100%').height('100%').backgroundColor(Color.White).zIndex(999)
      }
    }.width('100%').height('100%')
  }
}

```

**文件：`src/main/ets/pages/TodoPage.ets`**

```tsx
import SQLiteHelper, { ServerTodoItem } from '../common/database/SQLiteHelper';
import { HttpUtil } from '../common/utils/HttpUtil';
import { Api } from '../common/constants/Api';
import { TodoModel } from '../model/TodoModel';
import promptAction from '@ohos.promptAction';

/**
 * 待办事项编辑弹窗
 */
@CustomDialog
struct EditDialog {
  controller: CustomDialogController
  @State content: string = ''
  confirm: (content: string) => void = () => {}
  build() {
    Column() {
      Text('修改待办').fontSize(20).fontWeight(FontWeight.Bold).margin({ top: 20, bottom: 20 })
      TextInput({ text: this.content }).onChange((val) => this.content = val).margin({ bottom: 20 }).width('90%')
      Row() {
        Button('取消').onClick(() => this.controller.close()).backgroundColor(Color.Gray).margin({ right: 20 })
        Button('确定').onClick(() => { this.confirm(this.content); this.controller.close() })
      }.margin({ bottom: 20 })
    }.backgroundColor(Color.White).borderRadius(20).width('80%')
  }
}

/**
 * 待办事项页面组件
 * 包含列表展示、添加、修改、删除及云端同步逻辑。
 */
@Component
export struct TodoPage {
  @State todoList: Array<TodoModel> = [];
  @State newContent: string = '';
  userId: number = AppStorage.get<number>('userId') || 1;
  @State editId: number = 0;
  @State editContent: string = '';

  dialogController: CustomDialogController = new CustomDialogController({
    builder: EditDialog({ content: this.editContent, confirm: (newVal: string) => { this.handleUpdateContent(newVal); } }),
    autoCancel: true
  })

  async aboutToAppear(): Promise<void> {
    try {
      // 优先加载本地数据以提升首屏速度
      this.todoList = await SQLiteHelper.queryAll();
      await this.syncFromCloud();
    } catch (err) { console.error('Data load error:', JSON.stringify(err as Object)); }
  }

  // 同步策略：先推（本地新增），再拉（云端全量），最后覆盖本地
  async syncFromCloud(): Promise<void> {
    try {
      const unsyncedList = await SQLiteHelper.getUnsyncedTodos();
      if (unsyncedList.length > 0) {
        for (let item of unsyncedList) {
           let params: Record<string, Object> = { 'userId': this.userId, 'content': item.content };
           await HttpUtil.post(Api.ADD_TODO, params);
        }
      }
      const res = await HttpUtil.get(`${Api.GET_TODOS}?userId=${this.userId}`);
      if (res && res.code === 200 && res.data) {
        const serverListData = res.data as Object as Array<ServerTodoItem>;
        await SQLiteHelper.syncOverwrite(serverListData);
        this.todoList = await SQLiteHelper.queryAll();
      }
    } catch (err) { console.error('Sync error:', JSON.stringify(err as Object)); }
  }

  async handleAdd(): Promise<void> {
    if (this.newContent.trim() === '') { promptAction.showToast({ message: '请输入内容' }); return; }
    try {
      let params: Record<string, Object> = { 'userId': this.userId, 'content': this.newContent };
      const res = await HttpUtil.post(Api.ADD_TODO, params);
      if (res && res.code === 200 && res.data) {
        // 有网状态：保存服务端 ID
        const serverId = res.data['id'] as number;
        const newTodo = new TodoModel(this.newContent, serverId, 0);
        await SQLiteHelper.insert(newTodo);
      } else { throw new Error('API Error'); }
    } catch (err) {
      // 离线状态：保存本地，server_id = 0
      const newTodo = new TodoModel(this.newContent, 0, 0);
      await SQLiteHelper.insert(newTodo);
      promptAction.showToast({ message: '已离线保存' });
    }
    this.todoList = await SQLiteHelper.queryAll();
    this.newContent = '';
  }

  async handleStatusChange(item: TodoModel, newStatus: number): Promise<void> {
    try {
      const index = this.todoList.findIndex(i => i.id === item.id);
      if (index !== -1) { this.todoList[index].status = newStatus; this.todoList = [...this.todoList]; }
      let params: Record<string, Object> = { 'id': item.server_id, 'status': newStatus };
      await HttpUtil.post(Api.UPDATE_TODO, params);
      await this.syncFromCloud();
    } catch (err) { await this.syncFromCloud(); }
  }

  async handleDelete(item: TodoModel): Promise<void> {
    try {
      if (item.server_id !== 0) {
        let params: Record<string, Object> = { 'id': item.server_id };
        await HttpUtil.post(Api.DELETE_TODO, params);
      }
      this.todoList = this.todoList.filter(i => i.id !== item.id);
      await this.syncFromCloud();
      promptAction.showToast({ message: '已删除' });
    } catch (err) { console.error('Delete error:', JSON.stringify(err as Object)); }
  }

  openEditDialog(item: TodoModel): void { this.editId = item.id; this.editContent = item.content; this.dialogController.open(); }

  async handleUpdateContent(newVal: string): Promise<void> {
    try {
      const index = this.todoList.findIndex(i => i.id === this.editId);
      if (index !== -1) {
        this.todoList[index].content = newVal;
        this.todoList = [...this.todoList];
        await SQLiteHelper.updateTodo(this.todoList[index]);
      }
      const targetItem = this.todoList[index];
      if (targetItem.server_id !== 0) {
         let params: Record<string, Object> = { 'id': targetItem.server_id, 'content': newVal };
         await HttpUtil.post(Api.UPDATE_TODO_CONTENT, params);
      }
      promptAction.showToast({ message: '修改成功' });
    } catch (err) { console.error('Update content error:', JSON.stringify(err as Object)); promptAction.showToast({ message: '已离线修改' }); }
  }

  @Builder TitleArea() { Column() { Text('TodoList').fontSize(36).fontWeight(FontWeight.Bold).fontColor('#333').margin({ top: 30, bottom: 20 }).shadow({ radius: 2, color: 'rgba(0,0,0,0.2)', offsetY: 2 }) }.width('100%').alignItems(HorizontalAlign.Center) }

  @Builder InputArea() {
    Row() {
      TextInput({ placeholder: '请输入代办事项', text: this.newContent }).height(50).layoutWeight(1).backgroundColor(Color.White).borderRadius(25).padding({ left: 20 }).onChange((val) => this.newContent = val).shadow({ radius: 10, color: 'rgba(0,0,0,0.1)', offsetY: 5 })
      Button('添加').height(50).width(80).margin({ left: 15 }).type(ButtonType.Capsule).backgroundColor(this.newContent.trim().length > 0 ? '#007DFF' : '#E0E0E0').fontColor(this.newContent.trim().length > 0 ? Color.White : '#333').fontSize(16).shadow({ radius: 5, color: 'rgba(0,0,0,0.1)', offsetY: 2 }).onClick(() => { this.handleAdd(); })
    }.width('90%').margin({ bottom: 30 })
  }

  @Builder SectionHeader(title: string, count: number) {
    Row() { Text(title).fontSize(24).fontWeight(FontWeight.Bold).fontColor('#333'); Blank(); Stack({ alignContent: Alignment.Center }) { Circle({ width: 24, height: 24 }).fill(Color.White); Text(count.toString()).fontSize(14).fontColor('#333') } }.width('90%').margin({ bottom: 15, top: 10 })
  }

  @Builder TodoItemCard(item: TodoModel, isFinished: boolean) {
    Row() {
      Text(item.content).fontSize(16).fontColor('#333').layoutWeight(1).maxLines(1).textOverflow({ overflow: TextOverflow.Ellipsis }).decoration({ type: isFinished ? TextDecorationType.LineThrough : TextDecorationType.None }).opacity(isFinished ? 0.6 : 1)
      Row({ space: 8 }) {
        Button(isFinished ? '取消完成' : '确认完成').fontSize(10).height(28).backgroundColor('#F0F0F0').fontColor('#333').border({ width: 1, color: '#999' }).type(ButtonType.Capsule).onClick(() => { this.handleStatusChange(item, isFinished ? 0 : 1); })
        Button('删除').fontSize(10).height(28).backgroundColor('#FF6B22').fontColor(Color.White).type(ButtonType.Capsule).onClick(() => { this.handleDelete(item); })
        Button('修改').fontSize(10).height(28).backgroundColor('#4876FF').fontColor(Color.White).type(ButtonType.Capsule).onClick(() => { this.openEditDialog(item); })
      }
    }.width('90%').height(60).backgroundColor(Color.White).borderRadius(30).padding({ left: 20, right: 10 }).margin({ bottom: 12 }).shadow({ radius: 8, color: 'rgba(0,0,0,0.1)', offsetY: 4 })
  }

  build() {
    Column() {
      Scroll() {
        Column() {
          this.TitleArea()
          this.InputArea()
          this.SectionHeader('未完成', this.todoList.filter(i => i.status === 0).length)
          ForEach(this.todoList.filter(i => i.status === 0), (item: TodoModel) => { this.TodoItemCard(item, false) }, (item: TodoModel) => item.id.toString() + item.content)
          this.SectionHeader('已完成', this.todoList.filter(i => i.status === 1).length)
          ForEach(this.todoList.filter(i => i.status === 1), (item: TodoModel) => { this.TodoItemCard(item, true) }, (item: TodoModel) => item.id.toString() + item.content)
          Blank().height(50)
        }.width('100%')
      }.scrollBar(BarState.Off).edgeEffect(EdgeEffect.Spring)
    }.width('100%').height('100%').backgroundColor('#CDE2F6')
  }
}

```

**文件：`src/main/ets/pages/NoteEditPage.ets`**

```tsx
import router from '@ohos.router';
import { HttpUtil } from '../common/utils/HttpUtil';
import { Api } from '../common/constants/Api';
import promptAction from '@ohos.promptAction';
import SQLiteHelper from '../common/database/SQLiteHelper';
import { NoteModel } from '../model/NoteModel';

@Entry
@Component
struct NoteEditPage {
  @State content: string = '';
  @State noteId: number = 0;
  @State serverId: number = 0;
  topHeight: number = AppStorage.get<number>('topHeight') || 0;
  userId: number = AppStorage.get<number>('userId') || 1;

  aboutToAppear() {
    const params = router.getParams() as Record<string, Object>;
    if (params && params['id']) {
      this.noteId = params['id'] as number;
      this.content = params['content'] as string;
      this.serverId = (params['server_id'] as number) || 0;
    }
  }

  async saveNote() {
    if (this.content.trim() === '') { promptAction.showToast({ message: '内容不能为空' }); return; }
    const title = this.content.substring(0, 10);
    try {
      let url = Api.ADD_NOTE;
      let params: Record<string, Object> = { 'userId': this.userId, 'title': title, 'content': this.content };
      if (this.serverId !== 0) { url = Api.UPDATE_NOTE; params['id'] = this.serverId; }
      const res = await HttpUtil.post(url, params);
      if (res && res.code === 200) {
        promptAction.showToast({ message: '保存成功' });
        if (this.noteId !== 0) {
           const note = new NoteModel(this.content, title, this.userId, this.serverId, this.noteId);
           await SQLiteHelper.updateNote(note);
        }
        router.back();
      } else { throw new Error('API Fail'); }
    } catch (err) {
      if (this.noteId === 0) {
        const note = new NoteModel(this.content, title, this.userId, 0, 0);
        await SQLiteHelper.insertNote(note);
      } else {
        const note = new NoteModel(this.content, title, this.userId, this.serverId, this.noteId);
        await SQLiteHelper.updateNote(note);
      }
      promptAction.showToast({ message: '已离线保存' });
      router.back();
    }
  }

  async deleteNote() {
    if (this.noteId === 0) return;
    AlertDialog.show({
      title: '提示', message: '确定要删除这条笔记吗？',
      primaryButton: { value: '取消', action: () => {} },
      secondaryButton: {
        value: '删除', fontColor: Color.Red,
        action: async () => {
          try {
             if (this.serverId !== 0) { await HttpUtil.post(Api.DELETE_NOTE, { 'id': this.serverId }); }
             await SQLiteHelper.deleteNote(this.noteId);
             promptAction.showToast({ message: '已删除' }); router.back();
          } catch (err) {
             await SQLiteHelper.deleteNote(this.noteId);
             promptAction.showToast({ message: '已离线删除' }); router.back();
          }
        }
      }
    });
  }

  build() {
    Column() {
      Row() {
        Row() { Text('<').fontSize(24).fontWeight(FontWeight.Bold).margin({ right: 5 }); Text(this.noteId === 0 ? '新建笔记' : '编辑笔记').fontSize(18).fontWeight(FontWeight.Bold) }.onClick(() => router.back())
        Row({ space: 15 }) {
          if (this.noteId !== 0) { Text('删除').fontSize(16).fontColor(Color.Red).onClick(() => this.deleteNote()) }
          Text('保存').fontSize(16).fontColor('#007DFF').onClick(() => this.saveNote())
        }
      }.width('100%').height(50 + px2vp(this.topHeight)).padding({ left: 15, right: 15, top: px2vp(this.topHeight) }).justifyContent(FlexAlign.SpaceBetween).backgroundColor(Color.White).border({ width: { bottom: 1 }, color: '#eee' })
      TextArea({ placeholder: '在此输入笔记内容...', text: this.content }).width('100%').layoutWeight(1).backgroundColor(Color.White).fontSize(16).padding(15).textAlign(TextAlign.Start).onChange((value) => { this.content = value; })
    }.width('100%').height('100%').backgroundColor(Color.White)
  }
}

```

### 5.5 视图组件 (`view`)

**文件：`src/main/ets/view/PrivacyDialog.ets`**

```tsx
/**
 * 隐私政策弹窗组件
 * 强制用户阅读并同意协议后方可使用 App。
 */
@CustomDialog
export struct PrivacyDialog {
  controller: CustomDialogController;
  confirm: () => void = () => {};

  build() {
    Column() {
      Text('隐私政策').fontSize(22).fontWeight(FontWeight.Bold).fontColor('#333333').margin({ top: 25, bottom: 15 })
      Divider().color('#DCDCDC').strokeWidth(1).width('90%').margin({ bottom: 15 })
      Scroll() {
        Text(`本应用尊重并保护所有使用服务用户的个人隐私权...`).fontSize(15).lineHeight(24).fontColor('#333333')
      }.height(300).width('90%').backgroundColor('#E6F0FF').padding(15).margin({ bottom: 20 }).align(Alignment.TopStart)
      Text('已知晓').fontSize(18).fontColor(Color.Red).fontWeight(FontWeight.Medium).onClick(() => { this.controller.close(); this.confirm(); }).margin({ bottom: 25 })
    }.backgroundColor(Color.White).borderRadius(16).width('85%')
  }
}

```

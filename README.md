# 🧰 NocoBase 工具箱管理平台 — 企业内部部署指南

> 基于 [NocoBase](https://www.nocobase.com/) 开源无代码/低代码平台，为公司内部搭建可本地部署的工具箱管理系统。

---

## 目录

1. [NocoBase 是什么？](#1-nocobase-是什么)
2. [核心能力一览](#2-核心能力一览)
3. [本地部署指南（Docker）](#3-本地部署指南docker)
4. [用 NocoBase 搭建工具箱管理平台](#4-用-nocobase-搭建工具箱管理平台)
5. [插件生态](#5-插件生态)
6. [工作流自动化](#6-工作流自动化)
7. [权限与访问控制](#7-权限与访问控制)
8. [同类开源平台对比](#8-同类开源平台对比)
9. [常见问题 & 最佳实践](#9-常见问题--最佳实践)
10. [参考资源](#10-参考资源)

---

## 1. NocoBase 是什么？

NocoBase 是一款 **AI 驱动、高度可扩展的开源无代码/低代码平台**，专为构建企业内部业务系统而设计。它的核心理念是"数据模型驱动 UI"，让数据结构与界面展示完全解耦，开发者与非技术人员都可以快速搭建复杂的业务应用。

**关键亮点：**
- 完全开源，可 100% 自托管（私有部署）
- 微内核 + 插件架构，功能模块均可按需加载
- 内置 AI 能力（AI 员工、智能工作流）
- 企业级细粒度权限管理（行级 / 字段级）
- 支持 MySQL、PostgreSQL、MariaDB 等多种数据库

---

## 2. 核心能力一览

| 能力模块 | 说明 |
|---|---|
| **可视化应用构建器** | 拖拽式 WYSIWYG 界面编辑，支持表格、表单、看板、日历、图表、仪表盘等多种视图 |
| **数据模型管理** | 可视化定义数据集合（Collection）、字段类型及关联关系，自动生成 CRUD API |
| **工作流引擎** | 可视化节点编排，支持数据事件触发、定时任务、审批流、HTTP 调用、SQL 操作等 20+ 节点类型 |
| **插件系统** | 一切皆插件，支持用 TypeScript/React 自定义插件，无需改动核心代码 |
| **权限管理** | 基于角色（RBAC），支持菜单级 / 表级 / 字段级 / 行级权限控制 |
| **多数据源接入** | 可连接已有数据库、REST API、外部第三方系统 |
| **AI 员工** | 可嵌入工作流的 AI 助手，用于数据分析、翻译、汇总等场景 |
| **通知系统** | 支持邮件、短信、站内信等多渠道通知 |
| **多应用 / 多租户** | 单一部署下支持多个相互隔离的应用实例 |

---

## 3. 本地部署指南（Docker）

### 3.1 前置条件

- 安装 Docker 及 Docker Compose（推荐 Docker 20.x+）
- 服务器配置建议：2 核 CPU，4 GB 内存，20 GB 磁盘空间
- 开放端口：`13000`（或自定义）

```bash
# Ubuntu / Debian 安装 Docker
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
```

### 3.2 创建 `docker-compose.yml`

```bash
mkdir my-nocobase && cd my-nocobase
```

将以下内容保存为 `docker-compose.yml`（推荐使用 PostgreSQL）：

> **安全警告：** 以下配置中所有标注 `CHANGE_THIS_*` 的占位符**必须**在启动前替换为强密码/密钥，切勿使用示例值部署到生产环境！

```yaml
version: "3"

networks:
  nocobase:
    driver: bridge

services:
  app:
    image: nocobase/nocobase:latest
    restart: always
    networks:
      - nocobase
    depends_on:
      - postgres
    environment:
      - APP_KEY=CHANGE_THIS_APP_KEY          # 必须修改，可用 openssl rand -hex 32 生成
      - DB_DIALECT=postgres
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_DATABASE=nocobase
      - DB_USER=nocobase
      - DB_PASSWORD=CHANGE_THIS_DB_PASSWORD  # 必须修改，使用强密码
      - TZ=Asia/Shanghai
    volumes:
      - ./storage:/app/nocobase/storage
    ports:
      - "13000:80"

  postgres:
    image: postgres:16
    restart: always
    command: postgres -c wal_level=logical
    environment:
      POSTGRES_USER: nocobase
      POSTGRES_DB: nocobase
      POSTGRES_PASSWORD: CHANGE_THIS_DB_PASSWORD  # 必须与上方 DB_PASSWORD 保持一致
    volumes:
      - ./storage/db/postgres:/var/lib/postgresql/data
    networks:
      - nocobase
```

生成安全密钥的方式：

```bash
# 生成 APP_KEY
openssl rand -hex 32

# 生成强数据库密码（示例）
openssl rand -base64 24
```

### 3.3 启动服务

```bash
docker compose pull        # 拉取最新镜像
docker compose up -d       # 后台启动
docker compose logs -f app # 查看启动日志
```

### 3.4 访问系统

浏览器打开 `http://localhost:13000`（服务器部署则替换为对应 IP）。

**初始账号：**
- 邮箱：`admin@nocobase.com`
- 密码：`admin123`

> **安全提示：** 首次登录后请立即修改默认密码！

### 3.5 升级与备份

```bash
# 升级前备份数据
cp -r ./storage ./storage.bak

# 拉取最新镜像并重启
docker compose pull
docker compose up -d
```

---

## 4. 用 NocoBase 搭建工具箱管理平台

以下是一个典型的**企业内部工具箱管理平台**设计方案，覆盖常见需求：

### 4.1 推荐数据模型

| 集合名称 | 核心字段 | 说明 |
|---|---|---|
| `tools`（工具库） | 名称、类别、描述、访问链接、负责人、状态 | 所有内部工具的注册目录 |
| `categories`（分类） | 名称、图标、排序 | 工具分类管理 |
| `tool_requests`（申请记录） | 申请人、工具、申请理由、状态、审批人 | 工具访问权限申请流程 |
| `users`（用户）| 姓名、部门、角色 | 关联 NocoBase 内置用户系统 |
| `announcements`（公告） | 标题、内容、发布时间、发布人 | 系统通知与公告 |

### 4.2 推荐视图布局

- **首页仪表盘**：工具分类卡片 + 最近使用 + 公告栏
- **工具目录**：表格视图（支持搜索、筛选、按分类浏览）
- **我的申请**：当前用户的工具申请状态追踪
- **审批中心**（管理员）：待审批工单列表 + 一键审批
- **工具管理**（管理员）：工具增删改查、批量导入

### 4.3 建设步骤

1. **定义数据模型** — 在「数据表管理」中创建以上集合及字段
2. **配置界面** — 新建页面，拖拽添加「表格区块」「表单区块」「图表区块」
3. **设置工作流** — 配置工具申请 → 通知审批人 → 审批通过 → 自动开通权限的审批流
4. **配置权限** — 按角色（普通用户 / 部门管理员 / 超级管理员）分配数据及菜单访问权限
5. **测试上线** — 内部测试后通知全员使用

---

## 5. 插件生态

NocoBase 采用"一切皆插件"架构，以下是工具箱管理平台中常用的官方插件：

| 插件 | 用途 |
|---|---|
| `plugin-workflow` | 工作流引擎，用于审批流、自动化通知 |
| `plugin-acl` | 角色权限管理，字段 / 行级细粒度控制 |
| `plugin-action-import` | Excel 批量导入工具数据 |
| `plugin-charts` | 数据可视化，展示工具使用统计 |
| `plugin-file-manager` | 工具图标、附件文件管理 |
| `plugin-auth-sms` | 短信验证码登录（企业内网可替换为 LDAP/SSO） |
| `plugin-calendar` | 日历视图，用于排班、维护计划等 |
| `plugin-notifications` | 站内信 / 邮件通知 |

**自定义插件开发：**

```bash
# 创建自定义插件
yarn nocobase create-plugin my-toolbox-plugin

# 插件目录结构
packages/plugins/my-toolbox-plugin/
├── src/
│   ├── client/    # 前端（React 组件）
│   └── server/    # 后端（Node.js 接口）
└── package.json
```

---

## 6. 工作流自动化

NocoBase 工作流支持可视化节点编排，典型的工具申请审批流如下：

```
[触发器：提交申请表单]
        |
[通知节点：发送邮件给部门管理员]
        |
[人工审批节点：管理员在系统内审批]
        |
[条件判断：审批结果]
    /         \
[通过]        [拒绝]
  |              |
[更新记录：  [更新记录：
 状态=已通过]  状态=已拒绝]
  |              |
[通知申请人]  [通知申请人]
```

**支持的触发器类型：**
- 数据事件（新建 / 更新 / 删除记录时触发）
- 定时任务（Cron 表达式）
- 表单提交（手动触发）
- Webhook（外部系统回调）

---

## 7. 权限与访问控制

NocoBase 提供企业级 RBAC（基于角色的访问控制），可实现：

| 权限维度 | 说明 |
|---|---|
| **菜单权限** | 控制哪些角色可见哪些菜单页面 |
| **数据表权限** | 控制角色对特定数据表的增 / 删 / 改 / 查权限 |
| **字段权限** | 针对敏感字段（如成本、密码）设置只读或隐藏 |
| **数据行权限** | 用户只能查看/编辑自己创建的记录（"仅自己的数据"） |
| **操作权限** | 控制导出、导入、批量删除等高危操作 |

**推荐角色设计（工具箱平台）：**

```
超级管理员   → 全部权限
部门管理员   → 管理本部门工具申请 + 审批权限
普通用户     → 查看工具目录 + 提交申请 + 查看自己的申请记录
访客         → 仅查看工具目录（只读）
```

---

## 8. 同类开源平台对比

如果你在选型阶段，以下是几个常见开源内部工具平台的对比：

| 平台 | 类型 | 上手难度 | 可扩展性 | 最适合场景 | GitHub Stars |
|---|---|---|---|---|---|
| **NocoBase** | 无代码/低代码 | 中等 | 极强 | 复杂业务系统、权限精细、长期迭代 | 15k+ |
| **Appsmith** | 低代码 | 较低 | 强 | 仪表盘、管理后台、API 集成 | 35k+ |
| **Budibase** | 低代码 | 低 | 中等 | 快速 CRUD、表单流程、简单内部工具 | 23k+ |
| **ToolJet** | 低代码 | 较低 | 强 | 开发者团队、多数据源集成 | 34k+ |

**选型建议：**
- 需要**复杂权限、审批流程、长期维护**的业务系统 → **NocoBase**（推荐）
- 需要**快速搭建仪表盘 + API 对接**，团队有前端能力 → **Appsmith**
- 追求**最简单上手**，主要是表单和简单流程 → **Budibase**
- 团队**开发者居多**，需要灵活 JS 逻辑 → **ToolJet**

> 对于企业内部**工具箱管理平台**这一场景，NocoBase 的插件扩展能力和细粒度权限控制是核心优势，推荐优先选用。

---

## 9. 常见问题 & 最佳实践

**Q: 数据库应该选 PostgreSQL 还是 MySQL？**

推荐 **PostgreSQL**。NocoBase 对 PostgreSQL 的支持最为完整，部分高级功能（如逻辑复制、全文搜索）依赖 PostgreSQL 特性。MySQL 在高并发下偶有兼容性问题。

**Q: 如何对外开放并保证安全？**

建议：① 在反向代理（Nginx）层配置 HTTPS；② 修改默认密码和 `APP_KEY`；③ 限制数据库端口仅本机访问；④ 定期备份 `storage` 目录。

**Q: 能否对接公司已有的 LDAP / AD 域账号？**

NocoBase 支持通过插件扩展认证方式（`plugin-auth-ldap`），可对接企业 LDAP/Active Directory，实现单点登录（SSO）。

**Q: NocoBase 支持移动端吗？**

当前版本主要针对桌面端浏览器，移动端响应式支持在持续完善中。如有强移动端需求，可结合微信小程序/企业微信等方式封装访问入口。

**Q: 如何做数据迁移或备份？**

备份 `storage` 目录（文件附件）加数据库 dump（PostgreSQL 用 `pg_dump`）即可完整保留所有数据。

---

## 10. 参考资源

- [NocoBase 官网（中文）](https://www.nocobase.com/cn)
- [官方文档](https://docs.nocobase.com/cn)
- [GitHub 仓库](https://github.com/nocobase/nocobase)
- [Docker 安装文档](https://docs.nocobase.com/cn/get-started/installation/docker)
- [官方教程](https://www.nocobase.com/cn/tutorials)
- [中文社区论坛](https://forum.nocobase.com/)
- [插件市场](https://www.nocobase.com/cn/plugins)

---

> 本文档持续更新，欢迎贡献改进建议。如有问题请提 Issue 或联系 @bbsleon。

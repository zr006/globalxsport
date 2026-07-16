# Pickleball Zone — 腾讯云 CloudBase 云托管部署指南

> 本指南面向零基础用户，重点讲解 Dockerfile 原理和 CloudBase 云托管部署流程。

---

## 前置准备

你需要以下账号：

1. **GitHub 账号** — 存放代码（[github.com](https://github.com)）
2. **腾讯云账号** — 使用 CloudBase 云托管（[cloud.tencent.com](https://cloud.tencent.com)）
3. **（可选）Docker Desktop** — 本地测试用（[docker.com](https://www.docker.com)）

---

## Dockerfile 详解

项目根目录下的 `Dockerfile` 是整个部署的核心。它告诉云平台"如何把你的代码打包成一个可运行的容器"。

### 逐行解读

```dockerfile
# 基础镜像：使用 Node.js 22 的 Alpine 精简版（约 50MB）
# Alpine 是一个极小的 Linux 系统，比普通 Ubuntu 镜像小 10 倍以上
FROM node:22-alpine
```

```dockerfile
# 设置时区。默认是 UTC（比北京时间慢 8 小时）
# 设置为 Asia/Shanghai 后，数据库的时间、日志的时间都是北京时间
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    apk del tzdata
```

```dockerfile
# better-sqlite3 是 C++ 编写的模块，安装时需要编译
# python3 + make + g++ 是编译工具链
RUN apk add --no-cache python3 make g++
```

```dockerfile
# 安全最佳实践：不用 root 用户运行应用
# 创建一个普通用户 appuser，后续用这个用户运行服务
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
```

```dockerfile
# 所有后续操作都在 /app 目录下进行
WORKDIR /app
```

```dockerfile
# 先只复制 package.json（不复制代码）
# 这样只要 package.json 不变，Docker 就会使用缓存的依赖层
# 大幅加速重复构建的速度
COPY server/package*.json ./server/
```

```dockerfile
# 只安装生产依赖（跳过 devDependencies）
# --omit=dev 等同于旧版的 --only=production
RUN cd server && npm install --omit=dev && npm cache clean --force
```

```dockerfile
# 复制后端代码
COPY server/ ./server/

# 复制前端 HTML 页面
COPY *.html ./

# 复制图片资源
COPY assets/ ./assets/

# 复制字体文件
COPY _shared/ ./_shared/
```

```dockerfile
# 创建上传目录并赋予 appuser 权限
RUN mkdir -p server/uploads && chown -R appuser:appgroup /app
```

```dockerfile
# 切换到普通用户（安全）
USER appuser
```

```dockerfile
# 声明服务监听的端口
EXPOSE 3000
```

```dockerfile
# 容器启动时执行的命令
# 先运行 db.js 初始化数据库，再启动服务器
CMD ["sh", "-c", "cd server && node db.js && node index.js"]
```

### .dockerignore 的作用

类似 `.gitignore`，告诉 Docker 打包时**排除**哪些文件：

```
node_modules/     # 不打包依赖（容器内会重新安装）
server/data.db    # 不打包数据库（每次部署时重新创建）
*.tar.gz          # 不打包压缩包
DEPLOY.md         # 不打包部署文档
```

这样可以大幅减小镜像体积，加快上传和部署速度。

---

## 方式一：CloudBase 控制台部署（推荐小白）

### 第 1 步：把代码推送到 GitHub

和之前 Railway/Render 的流程一样，先在 GitHub 创建仓库并推送代码。

1. 创建 GitHub 仓库 `pickleball-zone`
2. 把整个 `pickleball-club` 文件夹推送到仓库
3. 确认仓库里包含 `Dockerfile` 和 `.dockerignore`

### 第 2 步：开通 CloudBase 云托管

1. 登录 [腾讯云控制台](https://console.cloud.tencent.com)
2. 搜索 **"云开发 CloudBase"** 或直接访问 [cloudbase.net](https://docs.cloudbase.net/run/quick-start/introduce)
3. 点击 **"新建环境"**
4. 选择 **"按量计费"**（免费额度内不用花钱）
5. 填写：
   - **环境名称**: `pickleball-zone`
   - **地域**: 选 **"上海"**（CloudBase 云托管目前只支持上海）
6. 点击 **"确定"**，等待环境创建完成（约 1-2 分钟）

### 第 3 步：创建云托管服务

1. 进入 CloudBase 控制台 → 左侧菜单选 **"云托管"**
2. 点击 **"新建服务"**
3. 填写配置：
   - **服务名称**: `pickleball-zone`
   - **网络**: 使用默认
   - **镜像仓库**: 选择 **"自动创建同名仓库"**（最简单）
4. 点击 **"确定"**

### 第 4 步：上传代码部署

#### 方法 A：上传代码包（最简单，不用学 Git）

1. 在本机把 `pickleball-club` 文件夹压缩成 **zip** 文件
   - Mac：右键 → 压缩
   - Windows：右键 → 发送到 → 压缩(zipped)文件夹
2. 回到 CloudBase 云托管页面
3. 点击你的服务 → **"部署"**
4. 选择 **"上传代码包"**
5. 选择 **"Dockerfile 部署"**
6. 上传你刚才压缩的 zip 文件
7. 点击 **"确定"**，等待构建和部署（约 3-5 分钟）

#### 方法 B：从 GitHub 拉取（需要绑定 GitHub）

1. 在 CloudBase 控制台 → **"设置"** → **"代码源"**
2. 绑定你的 GitHub 账号，选择 `pickleball-zone` 仓库
3. 回到服务页面 → **"部署"** → 选择 **"从代码仓库拉取"**
4. 选择分支 `main`，点击 **"确定"**

#### 方法 C：用 CLI 命令行部署

```bash
# 1. 安装 CloudBase CLI
npm install -g @cloudbase/cli

# 2. 登录
tcb login

# 3. 在项目根目录执行部署
cd pickleball-club
tcb cloudrun deploy
```

### 第 5 步：配置监听端口

1. 进入 CloudBase 控制台 → 云托管 → 你的服务
2. 点击 **"服务配置"** → **"基础配置"**
3. **监听端口** 设置为 **3000**（和代码中 Express 监听的端口一致）
4. 点击 **"保存"**，服务会自动重新部署

### 第 6 步：获取访问地址

1. 在服务页面，找到 **"访问配置"** 或 **"公网访问"**
2. 如果没有自动生成，点击 **"新建"** → 选择 **"HTTP"**
3. 你会获得一个类似这样的地址：
   ```
   https://pickleball-zone-xxxx.ap-shanghai.run.cloudbaseapp.com
   ```
4. 点击即可在浏览器中打开你的网站！

---

## 方式二：本地 Docker 测试（可选）

如果你想在本地先验证 Docker 镜像能正常工作：

```bash
# 1. 进入项目目录
cd pickleball-club

# 2. 构建 Docker 镜像
docker build -t pickleball-zone .

# 3. 运行容器（把本机 3000 端口映射到容器的 3000 端口）
docker run -p 3000:3000 pickleball-zone

# 4. 浏览器打开 http://localhost:3000 验证
```

如果看到 `✓ Pickleball Zone 服务器已启动: http://localhost:3000` 说明镜像构建成功。

```bash
# 停止容器：按 Ctrl+C

# 查看镜像大小
docker images | grep pickleball-zone
# 预期：约 200-300MB
```

---

## 数据持久化（重要！）

CloudBase 云托存的容器是**无状态**的，意味着：
- 每次重新部署，容器内的文件会被重置
- 数据库 `data.db` 会在每次部署时重新创建（`db.js` 里有自动初始化逻辑）
- **用户上传的图片会丢失**

### 解决方案

#### 数据库：推荐升级为腾讯云 MySQL/TDSQL

1. CloudBase 控制台 → **"数据库"** → **"新建 MySQL 实例"**
2. 选择免费套餐（如可用）
3. 修改代码中的数据库连接为 MySQL
4. 这样数据就不会因为重新部署而丢失

#### 上传图片：使用腾讯云 COS（对象存储）

1. CloudBase 控制台 → **"存储"** → 开通云存储
2. 修改上传接口，将文件上传到 COS 而非本地
3. COS 中的文件是持久化的，不会丢失

> **如果暂时不需要数据持久化**：当前方案也可以正常使用。每次重新部署会恢复默认的测试数据（admin 账号、demo 会员、示例活动等）。

---

## 费用说明

CloudBase 云托管采用**按量计费**：

| 资源 | 免费额度 | 超出后价格 |
|------|---------|-----------|
| CPU | 每月 40000 核秒 | 约 ¥0.000111/核秒 |
| 内存 | 每月 80000 GB秒 | 约 ¥0.000028/GB秒 |
| 流量 | 每月 1 GB | 约 ¥0.8/GB |
| 构建时长 | 每月 60 分钟 | 约 ¥0.08/分钟 |

**对俱乐部网站来说**：如果每天访问量在几百次以内，免费额度基本够用。超出部分每月大约 ¥10-30。

### 省钱技巧

- 在 CloudBase 控制台设置 **"缩容到 0"**：没有访问时不运行容器，不产生费用
- 访问时会自动扩容，冷启动约 5-10 秒
- 适合访问量不大的俱乐部网站

---

## 常见问题

**Q: 构建失败，日志显示 `better-sqlite3` 编译错误**
A: Dockerfile 里已经包含了 `python3 make g++` 编译工具。如果仍然失败，尝试：
   - 检查 `.dockerignore` 是否排除了 `server/node_modules`
   - 确认 `server/package.json` 存在

**Q: 部署成功但访问不了**
A: 检查以下几项：
   1. **监听端口**是否设置为 `3000`（服务配置 → 基础配置）
   2. 是否开启了 **公网访问**（访问配置 → 新建 HTTP 访问）
   3. 查看服务日志是否有报错

**Q: 如何绑定自定义域名？**
A:
   1. 在腾讯云解析（DNSPod）添加域名
   2. CloudBase 控制台 → 你的环境 → **"静态网站"** 或 **"云托管"** → **"域名管理"**
   3. 添加自定义域名，按提示配置 CNAME 记录

**Q: 和 Railway / Render 相比，CloudBase 有什么优势？**
A:
   - **国内访问速度快**（服务器在上海）
   - **不需要翻墙**部署和管理
   - **微信生态集成**（如果以后要做小程序版）
   - **按量缩容到 0** 更省费用

---

## 文件清单

部署到 CloudBase 需要确保以下文件都在仓库里：

```
pickleball-club/
├── Dockerfile              ← 新增：Docker 构建配置
├── .dockerignore           ← 新增：Docker 打包排除规则
├── render.yaml             ← Render 配置（不影响 CloudBase）
├── railway.json            ← Railway 配置（不影响 CloudBase）
├── package.json
├── *.html                  ← 所有前端页面
├── assets/                 ← 图片资源
├── _shared/fonts/          ← 字体文件
└── server/                 ← 后端代码
    ├── package.json
    ├── index.js
    ├── db.js
    ├── middleware/
    └── routes/
```

所有平台共用同一份代码，互不冲突。

---

## 需要帮助？

把 CloudBase 控制台的构建日志截图或报错信息发给我，我帮你排查！
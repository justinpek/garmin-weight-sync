# 小米体重秤自动同步到佳明 (Garmin) 工具

本工具可以将小米运动健康 (Mi Fitness) 中的体重数据，自动生成 Garmin 兼容的 FIT 文件并同步到佳明 Connect 账户。目前支持小米体脂秤 S400 等主流型号。

---

## 📖 目录
1. [新手环境准备](#1-新手环境准备)
2. [下载与安装](#2-下载与安装)
3. [快速配置](#3-快速配置)
4. [核心使用流程 (Python 方式)](#4-核心使用流程-python-方式)
5. [进阶功能说明](#5-进阶功能说明)
6. [Docker 使用指南 (推荐)](#6-docker-使用指南-推荐)
7. [常见问题 (FAQ)](#7-常见问题-faq)

---

## 1. 新手环境准备

### 项目结构说明
```text
garmin-weight-sync/
├── src/                # 源代码文件夹
│   ├── xiaomi/         # 小米登录与数据获取模块
│   ├── garmin/         # 佳明上传与文件生成模块
│   └── main.py         # 一键同步主程序
├── users.json          # 您的核心配置文件 (存账号密码)
├── requirements.txt    # 必须安装的程序组件包
├── Dockerfile          # Docker 构建文件
├── docker-compose.yml  # Docker 编排文件
└── README.md           # 您正在看的这份文档
```

在开始使用之前，您需要确保电脑上安装了 Python（这是运行此程序的程序包）。**如果您使用 Docker，可以跳过此步骤。**

### 第一步：安装 Python
1. 访问 [Python 官网](https://www.python.org/downloads/)。
2. 下载并安装 **Python 3.12.6** **(注意：Python 3.14 可能导致兼容性问题)**。
3. **特别注意**：安装过程中，一定要勾选 **"Add Python to PATH"**（将 Python 添加到系统变量）。

### 第二步：确认安装成功
打开您的终端（Windows 用户按 `Win+R` 输入 `cmd`；Mac 用户打开 `Terminal`），输入以下命令并按回车：
```bash
python --version
```
如果显示 `Python 3.12.x`，说明安装成功。

---

## 2. 下载与安装

### 第一步：建立虚拟环境（推荐）
虚拟环境可以防止项目依赖与您的系统环境产生冲突。
```bash
# 进入项目目录（请根据您的实际路径操作）
cd garmin-weight-sync

# 创建虚拟环境
python -m venv .venv

# 激活虚拟环境
# Windows 用户:
.venv\Scripts\activate
# Mac/Linux 用户:
source .venv/bin/activate
```

### 第二步：一键安装依赖
```bash
pip install -r requirements.txt
```

---

## 3. 快速配置

在项目文件夹中，找到 `users.json` 文件（如果没有，请手动新建一个）。**这是程序唯一的配置文件。**

### 配置文件模板
请填入您的小米和佳明账户信息：

```json
{
    "users": [
        {
            "username": "您的手机号/邮箱",   
            "password": "小米账号密码",      
            "model": "yunmai.scales.ms103",
            "token": {
                "userId": "",
                "passToken": "",
                "ssecurity": ""
            },
            "garmin": {
                "email": "您的佳明账号",     
                "password": "佳明账号密码",  
                "domain": "CN"              
            }
        }
    ]
}
```

### 关键参数说明：
- **model**: 设备型号。如果您使用的是 **小米体脂秤 S400**，请填 `yunmai.scales.ms103`。
- **domain**: 佳明服务器区域。**中国区**填 `CN`，**国际区**（如台湾、香港、美国）填 `COM`。
- **token**: 初始留空即可。程序运行成功后会自动保存登录凭证，下次无需重复输入。

---

## 4. 核心使用流程 (Python 方式)

### 第一阶段：获取小米授权 (仅需执行一次)
因为小米账号需要处理图形验证码或短信验证，我们需要手动运行登录工具：

```bash
python src/xiaomi/login.py --config users.json
```

1. **图形验证码**：程序会自动在您的网页浏览器中打开一张图片。看清验证码后，回到终端输入并回车。
2. **2FA 验证**：如果您的账号开启了二次验证，程序会提示已发送验证码到您的手机，输入收到的 6 位数字即可。
3. **成功提示**：看到 `✅ Login SUCCESS!` 后，程序会自动更新 `users.json`。

### 第二阶段：同步数据到佳明
授权成功后，运行主程序开始同步：

```bash
# 获取数据、生成 FIT 文件、并全自动同步到佳明
python src/main.py --config users.json --sync
```

**程序运行时会做什么：**
1. 自动登录您的小米账户（使用之前获取的 Token）。
2. 从小米服务器拉取您的历史体重记录（默认显示最近 10 条）。
3. 自动在本地 `weight_data_{账户}.json` 备份数据。
4. 在 `garmin-fit/` 文件夹下生成佳明专用的数据文件。
5. 自动登录佳明系统并将数据同步上去。

---

## 5. 进阶功能说明

### 常用命令参数
- `--limit N`: 指定显示多少条最近的体重记录（默认 10）。
- `--fit`: 仅生成本地 FIT 文件，不上传。
- `--sync`: 同时执行生成和上传（一键同步模式）。

### 定时自动同步 (长期使用)
您可以设置定时任务（如 Linux 的 `cron` 或 Windows 的任务计划程序），每天自动运行：
```bash
# 示例：每天凌晨 2 点执行同步
0 2 * * * cd /您的项目路径 && .venv/bin/python src/main.py --sync
```

---

## 6. Docker 使用指南 (推荐)

如果您不想在本地安装 Python 环境，可以使用 Docker 运行本工具。

### 准备工作
1. 确保已安装 Docker 和 Docker Compose。
2. 在项目根目录下创建或修改 `users.json`，填入您的账号信息。

### 步骤一：构建镜像
```bash
docker-compose build
```

### 步骤二：首次登录 (交互式)
首次使用需要手动登录以获取 Token。请运行以下命令进入交互模式：

```bash
# Windows (PowerShell)
docker run --rm -it -v ${PWD}/users.json:/app/users.json -v ${PWD}:/app/host_mount garmin-weight-sync python src/xiaomi/login.py

# Linux / macOS
docker run --rm -it -v $(pwd)/users.json:/app/users.json -v $(pwd):/app/host_mount garmin-weight-sync python src/xiaomi/login.py
```

**注意**：
- 如果出现图形验证码，程序会在当前目录下生成 `captcha.png`。
- 请直接在文件夹中打开查看，然后在终端输入验证码。
- 登录成功后，Token 会自动保存到您的 `users.json` 文件中。

### 步骤三：一键同步
登录成功后，即可运行同步任务：

```bash
# 使用 Docker Compose 运行 (推荐)
docker-compose up
```

这将启动容器，执行同步任务，完成后容器会自动退出。

如果您希望它在后台定时运行，建议结合系统的定时任务（如 crontab）来调用上述命令，或者编写一个简单的 shell 脚本。

```bash
# 或者使用 Docker Run 手动运行
docker run --rm -v ${PWD}/users.json:/app/users.json -v ${PWD}/garmin-fit:/app/garmin-fit -v ${PWD}/.garth:/app/.garth garmin-weight-sync
```

---

## 7. 常见问题 (FAQ)

### Q: 提示 `ModuleNotFoundError: No module named 'requests'` 怎么办？
A: 确保您已激活虚拟环境并运行了 `pip install -r requirements.txt`。

### Q: 佳明上传一直提示 `Duplicate` 怎么办？
A: 说明佳明服务器已经存在这一份记录。佳明会自动识别重复数据并跳过，这不是错误，无需处理。

### Q: 换了新电脑/账号变动怎么办？
A: 删除 `users.json` 中的 `token` 部分，重新运行 `python src/xiaomi/login.py` 即可。

### Q: 支持哪些小米秤？
A: 理论上支持小米运动健康里绑定的所有体脂秤。如果默认 model 无法获取数据，请尝试在 App 内查看设备的插件信息获取对应 ID。

---

## 🛡️ 安全提示
- **users.json** 包含您的明文密码和敏感 Token，**请通过任何方式分享该文件**。
- 建议将 `.venv/`、`.garth/` 和 `users.json` 添加到 `.gitignore` 以防意外提交到公开 GitHub。

---

## ✨ 许可证
MIT License. 开发者：Leslie & Gemini Pair.
参考项目：[XiaomiGateway3](https://github.com/AlexxIT/XiaomiGateway3), [garth](https://github.com/matin/garth)

### Q: 在海外服务器上登录失败 / 提示 Connection aborted？
A: 小米账号对异地登录（尤其是海外 IP）有严格的风控。
1. 请先在**本地电脑**（同 IP 环境或常用环境）登录一次小米账号，处理完所有的“跨境数据传输”同意弹窗。
2. 如果仍然失败，建议在本地电脑运行此工具获取 `users.json` (包含 Token)，然后将生成的 `users.json` 上传到服务器覆盖即可。Token 一旦生成，通常可以在不同 IP 下使用。

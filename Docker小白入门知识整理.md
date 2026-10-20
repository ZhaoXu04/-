
---

## 目录

- [一、全局路线图](#一全局路线图)
- [二、环境准备（Windows + WSL2 + Docker Desktop）](#二环境准备windows--wsl2--docker-desktop)
- [三、镜像管理（Image）](#三镜像管理image)
- [四、容器运行与运维（Container）](#四容器运行与运维container)
- [五、数据持久化（挂载卷 Volume）](#五数据持久化挂载卷-volume)
- [六、Docker Compose 容器编排](#六docker-compose-容器编排)
- [七、实战：部署前端项目](#七实战部署前端项目)
- [八、命令速查表](#八命令速查表)
- [九、踩坑与 FAQ](#九踩坑与-faq)
- [十、原文遗留待补点](#原文十遗留待补点)

---

## 一、全局路线图

原文的学习主线其实是五个阶段，串联起来就是一条完整的 Docker 上手路径：

| 阶段      | 目标                        | 关键产物 / 命令                        |
| ------- | ------------------------- | -------------------------------- |
| ① 环境准备  | 让 Windows 具备 Linux 容器运行能力 | WSL2 + Docker Desktop            |
| ② 镜像    | 把「装好软件的只读模板」拉到本地          | `docker pull` / `images` / `rmi` |
| ③ 容器    | 用镜像跑起可访问的服务               | `docker run -d -p 8080:80`       |
| ④ 数据持久化 | 让数据在容器删除后仍然存在             | `-v`（绑定挂载 / 命名卷）                 |
| ⑤ 编排与部署 | 多容器协同、项目上线                | `docker compose up -d`           |

**一句话理解三个核心概念的关系**：

```
Dockerfile --build--> 镜像 (Image) --run--> 容器 (Container) --挂载--> 数据卷 (Volume)
```

- **镜像**：只读模板，类比「安装光盘 / 类（Class）」。
- **容器**：镜像的运行实例，可读写、可启停，类比「运行中的程序 / 对象（Object）」。
- **数据卷**：独立于容器生命周期的存储，类比「外挂硬盘」。

---

## 二、环境准备（Windows + WSL2 + Docker Desktop）

### 2.1 启用 Windows 功能

在「控制面板 → 程序和功能 → 启用或关闭 Windows 功能」中勾选：

- **适用于 Linux 的 Windows 子系统**（Windows Subsystem for Linux）
- **虚拟机平台**（Virtual Machine Platform）

两项都启用后按提示重启。

### 2.2 设置 WSL 版本为 WSL2

```bash
wsl --set-default-version 2
```

> 必须以 WSL2 为默认版本：Docker Desktop 在 Windows 上的 Linux 容器实际跑在 WSL2 虚拟机里，WSL1 不支持。

### 2.3 安装 / 升级 WSL

```bash
wsl --update --web-download
```

使用 `--web-download` 走网络下载，绕开 Microsoft Store 的常见故障。

**遇到 403 报错时**：

```
C:\Windows\System32>wsl --update
正在检查更新。
已禁止(403)。
错误代码: Wsl/UpdatePackage/0x80190193
```

> 处理方式：**暂时禁用 VPN / 代理软件**（如 Clash、Shadowsocks 等）后重试。代理会把 WSL 的更新请求劫持到不可用的线路，导致 403。

### 2.4 下载 Docker Desktop

| 资源 | 链接 |
|---|---|
| 官网 | https://www.docker.com/ |
| 官方文档 | https://docs.docker.com/get-started/ |
| 下载指引页 | https://docs.docker.com/get-started/get-docker/ |
| Windows 安装包直链 | `https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe` |

### 2.5 指定安装目录（避免占满 C 盘）

Docker Desktop 默认把程序、WSL 数据盘、Windows 容器数据都放在 C 盘，动辄几十 GB。安装前在**安装包所在目录**执行带参数的命令，把三处路径一次性挪到其他盘：

```bash
& "O:\chromeDownload\Docker Desktop Installer.exe" install --quiet --accept-license `
  --installation-dir="O:\Docker" `
  --wsl-default-data-root="O:\Docker\data" `
  --windows-containers-default-data-root="O:\Docker\windows-data"
```

| 参数 | 含义 |
|---|---|
| `--installation-dir` | 程序本体安装目录 |
| `--wsl-default-data-root` | Linux 容器镜像 / 数据卷的存储根目录（**最占空间的地方**） |
| `--windows-containers-default-data-root` | Windows 容器数据根目录 |
| `--quiet` / `--accept-license` | 静默安装、自动接受协议，便于脚本化 |

参考教程：https://www.cnblogs.com/shengmingboai/p/18884882

---

## 三、镜像管理（Image）

### 3.1 查看版本

```bash
docker --version
```

### 3.2 拉取镜像

```bash
docker pull nginx:1.0.0
```

语法：`docker pull <镜像名>:<标签>`，省略标签时默认 `latest`。

> **【补充】** 若拉取报 `manifest unknown / not found`，多半是标签不存在。`nginx` 官方镜像请用 `latest`、`stable`、`mainline` 或 `1.27` 这类真实版本号。

### 3.3 列出本地镜像

```bash
docker images
```

输出包含 `REPOSITORY`、`TAG`、`IMAGE ID`、`CREATED`、`SIZE` 五列。

### 3.4 删除镜像

```bash
docker rmi <镜像ID>/<镜像名>
```

> 有容器（含已停止容器）正在引用该镜像时无法删除，需先 `docker rm` 相关容器，或加 `-f` 强制删除。

### 3.5 拉取指定 CPU 架构的镜像

```bash
docker pull --platform=XXXXXX nginx
```

典型取值：`linux/amd64`、`linux/arm64`、`linux/arm/v7`。用于 Apple Silicon / ARM 服务器上运行 x86 镜像，或反之。

### 3.6 查看镜像漏洞与建议摘要

```bash
docker scout quickview nginx
```

Docker Scout 会给出该镜像的 CVE 数量分级和基础镜像升级建议，是上线前的安全检查动作。

---

## 四、容器运行与运维（Container）

### 4.1 用镜像创建容器

```bash
docker run nginx
```

前台模式：命令会占用当前终端，日志持续输出，**无法再做后续操作**（直接关掉终端或 `Ctrl+C` 容器即停止）。所以这种方式只适合临时验证。

### 4.2 后台模式运行（实际常用）

```bash
docker run -d nginx
```

- `-d` = **detached mode**，后台（分离）模式：容器在后台运行，终端立即返回容器 ID。
- 用 `docker ps` 确认状态，用 `docker logs` 看输出。

### 4.3 查看进程状态

`ps` 即 **process status**（进程状态）。

```bash
docker ps          # 正在运行的容器
docker ps -a       # 所有容器，含已停止的
```

### 4.4 端口映射

```bash
docker run -d -p <宿主机端口>:<容器端口> nginx
```

实例：

```bash
docker run -d -p 8080:80 nginx
```

宿主机浏览器访问 <http://localhost:8080/> 即可看到 Nginx 默认欢迎页。

> **注意**：冒号必须是**半角英文冒号 `:`**。中文全角冒号 `：` 会让命令直接报错或参数被当成镜像名。
>
> 方向记忆：`-p 宿主机:容器`，左边是「你在浏览器里敲的端口」，右边是「容器里程序真正监听的端口」。

### 4.5 自定义容器名称

```bash
docker run -d --name kz_nginx nginx
```

不指定时 Docker 会随机生成名字（如 `nifty_lalande`、`sweet_jepsen`），不利于后续 `exec` / `logs` / `rm` 操作。

### 4.6 进入容器 / 查看容器内部

```bash
docker exec -it <容器ID或名称> bash     # 进入运行中的容器（最常用）
docker exec <容器ID> ps -ef             # 查看容器内进程
docker exec -it <容器ID> ls -R <目录>   # 递归查看目录内容
```

- `-i` 保持标准输入打开，`-t` 分配伪终端，二者合用才是可交互的 shell。
- 容器里没有 `bash`（如 alpine）时改用 `sh`。
- **【补充】** `exec` 只能进「正在运行」的容器；容器已停止时用 `docker start` 后再进。

### 4.7 一次性调试容器

```bash
docker run -it --rm alpine
```

- `--rm`：退出时自动删除容器，不留垃圾。适合临时跑个命令、验证网络或系统行为。
- `alpine` 是一个体积极小的 Linux 发行版（约 5MB），常被当作调试用「系统」。

### 4.8 容器内安装软件前先确认发行版

```bash
cat /etc/os-release
```

不同基础镜像的包管理器不同：Debian/Ubuntu 用 `apt`，Alpine 用 `apk`，CentOS/RHEL 用 `yum`/`dnf`。装错包管理器会直接找不到命令。

### 4.9 删除容器

```bash
docker rm <容器ID>            # 删除已停止的容器
docker rm -f <容器ID>         # 强制删除，运行中的也能删
```

### 4.10 容器与宿主机之间传文件（docker cp）

`docker cp` 的语法模仿 Linux 的 `scp` / `cp`，是宿主机与容器之间复制文件的标准工具。

**宿主机 → 容器**

```bash
docker cp <宿主机路径> <容器ID或名称>:<容器内目标路径>

# 复制单个文件
docker cp /home/user/index.html my_nginx:/usr/share/nginx/html/index.html

# 复制整个文件夹（不需要 -r，docker cp 自动递归）
docker cp /home/user/project/dist my_nginx:/var/www/html
```

**容器 → 宿主机**

```bash
docker cp <容器ID或名称>:<容器内源路径> <宿主机目标路径>

docker cp my_nginx:/etc/nginx/nginx.conf ./nginx.conf
docker cp my_nginx:/var/log/nginx /home/user/logs_backup
```

**注意事项**

| 事项 | 说明 |
|---|---|
| 无需进入容器 | 容器存在即可，不要求运行中，也不需要 `exec` |
| 自动建目录 | 目标路径的父目录不存在时 Docker 会自动创建 |
| 覆盖风险 | 目标已存在同名文件会被**直接覆盖，无确认提示** |
| 权限问题 | 文件会保留原 UID/GID 或变成 root 所有，容器内可能读不到，需进容器 `chown` |
| 特殊字符 | 路径含空格时务必加双引号，如 `"C:/My Folder/file.txt"` |

---

## 五、数据持久化（挂载卷 Volume）

### 5.1 为什么需要挂载

容器删除后，容器内的写入随之消失。挂载（Mount）把**宿主机上的目录**或**Docker 管理的卷**接到容器内的路径上，从而实现**数据持久化**，并让宿主机与容器共享同一份文件。

Docker 提供两种挂载方式：

| | 绑定挂载（Bind Mount） | 命名卷（Named Volume） |
|---|---|---|
| 语法 | `-v <宿主机绝对路径>:<容器内路径>` | `-v <卷名>:<容器内路径>` |
| 数据存放位置 | 你指定的宿主机目录，路径完全可控 | Docker 数据根目录下（`volumes/<卷名>/_data`） |
| 是否需先创建 | 目录不存在时 Docker 会自动创建 | 可用 `docker volume create` 显式创建 |
| 适用场景 | 开发调试、共享代码 / 静态文件 | 数据库数据、生产环境持久化 |
| 备份迁移 | 直接拷目录即可 | 用 `docker volume` 系列命令管理 |

### 5.2 绑定挂载（Bind Mount）

```bash
docker run -d -p 8080:80 -v <宿主机目录>:<容器内目录> nginx
```

原文示例（Linux 云服务器，博主用的是上海云主机，所以路径是 `/website`）：

```bash
sudo docker run -d -p 8080:80 -v /website/html:/usr/share/nginx/html nginx
```

### 5.3 Windows 用户的关键差异 ⚠️

在 Windows 上用 Docker Desktop 时，容器跑在 **WSL2 虚拟机** 内，所以「宿主机路径」指的是 **WSL2 虚拟机里的路径**，而不是 Windows 的 `D:\...`。

**① 通过 WSL 共享路径访问**

先查发行版名称：

```bash
wsl -l -v
```

再拼出访问地址（示例发行版为 `docker-desktop`）：

```
\\wsl$\docker-desktop\website\html
```

**② 直接用 Windows 盘符（推荐写法）**

Windows 的 `O:\` 在 Docker 参数里写作 `/o/`（盘符小写、正斜杠）：

```bash
docker run -d -p 8080:80 -v /o/Docker/learnJyd/website/html:/usr/share/nginx/html nginx
```

> 结论：**绑定挂载的源目录最好先在宿主机上手动创建好并放入文件**，否则会踩下面 5.4 的「目录变空」问题。

### 5.4 踩坑：挂载后容器内目录是空的

**现象**：执行 `docker exec -it <id> ls -R /usr/share/nginx/html`，输出只有一个空的目录名：

```
/usr/share/nginx/html:
```

**原因：绑定挂载的「覆盖原则」**

- Nginx 官方镜像构建时，`/usr/share/nginx/html` 里本来有 `index.html` 等默认页面。
- 当你 `-v <宿主机目录>:/usr/share/nginx/html`，Docker 把宿主机目录**整个覆盖**到该位置。
- 宿主机那个目录是刚建的空文件夹 → 容器内对应目录也就空了。

**解决**：往宿主机目录里放入文件即可。例如在 `O:\Docker\learnJyd\website\html` 下新建 `index.html`，写入任意测试内容，刷新 <http://localhost:8080/> 就能看到页面。

> 原文给出了一段完整的 HTML 测试页（BMW 风格的 Docker 命令速查卡片，含 CSS），本文不重复贴出，按需自己写几行 HTML 即可。

**小结**：截图证明挂载路径配置是对的、容器也正常运行，问题只在于**源文件夹是空的**。

### 5.5 命名卷（Named Volume）

命名卷的数据位置由 Docker 管理，与宿主机具体目录解耦，更适合生产环境。

```bash
# 1. 创建卷
docker volume create nginx_html

# 2. 用卷启动容器
docker run -d -p 8080:80 -v nginx_html:/usr/share/nginx/html nginx

# 3. 查看卷的真实存放路径
docker volume inspect nginx_html
```

`docker volume inspect` 输出里的 `Mountpoint` 就是数据真实位置。**Linux 下需要切到 root 才能访问该目录**。

**Windows 下怎么访问？** 该路径位于 WSL2 内部，需要走 `\\wsl$` 或 `file://wsl.localhost` 访问，例如：

```
file://wsl.localhost/docker-desktop/mnt/docker-desktop-disk/data/docker/volumes/nginx_html/_data/index.html
```

> 原文记录的困惑：按 `docker volume inspect` 给出的路径去资源管理器里找会失败，因为那是**容器/虚拟机视角的路径**，不是 Windows 视角；换成上面的 `wsl.localhost` 地址即可。访问 <http://localhost:8080/> 正常则说明数据本身没问题。

**卷的日常管理**

```bash
docker volume list              # 列出所有创建过的卷
docker volume rm nginx_html     # 删除指定卷
docker volume prune -a          # 删除所有「没有任何容器引用」的卷
```

### 5.6 踩坑：`docker volume prune -a` 删不掉卷

**现象**：执行 `docker volume prune -a` 后，`nginx_html` 依然存在。

**原因**：`prune -a` 只删除「未被任何容器（含已停止容器）引用」的卷。只要还有容器挂载着它，即使容器已停止，也视为「被引用」，不会被清理——这是防止误删数据的保护机制。

**解决**

方法一（推荐）：先删引用它的容器

```bash
docker ps -a --filter "volume=nginx_html"   # 找到使用该卷的容器
docker rm <容器ID>                          # 删除容器
docker volume rm nginx_html                 # 再删卷
```

方法二：强制删除（⚠️ 数据不可恢复）

```bash
docker volume rm -f nginx_html
```

> ⚠️ 强制删除会连同卷内所有数据一起删除；若容器仍在运行，可能导致容器报错或数据丢失。

### 5.7 两个容器共用同一个卷

Docker 允许、也推荐多个容器同时挂载同一个卷，这种机制叫**数据卷共享**。

**为什么是合理的**

- **生命周期解耦**：卷独立于容器存在，删掉容器（不删卷）数据仍在。
- **多容器协同**的典型场景：
  - 一个 Nginx 容器对外提供静态页面，另一个容器负责更新这些页面文件；
  - 应用容器写日志，日志收集容器读取并转发；
  - 多个 Web 实例共享同一份代码 / 配置，实现负载均衡。
- **数据实时同步**：所有挂载同一卷的容器对文件的修改彼此立即可见，比网络传输或数据库快得多。

**注意事项**

| 风险 | 说明 |
|---|---|
| 并发写入冲突 | 多容器同时写同一文件可能损坏内容；Nginx 这类只读场景通常没问题，数据库等高频写入场景要格外小心 |
| 权限问题 | 不同容器内用户 UID 可能不同，导致读不到文件 |
| 清理策略 | 不再需要时先删容器、再删卷，避免留下占空间的「孤儿卷」 |

---

## 六、Docker Compose 容器编排

### 6.1 是什么

Compose 用**一个 YAML 文件**描述「容器之间如何创建、如何协同工作」。

可以把它理解成：**把一条或多条 `docker run` 命令，按特定格式写进一个文件里**，然后用一条命令整体起停。

它解决的核心问题：手工 `docker run` 时，端口、卷、网络、环境变量、容器间依赖都得逐个记、逐个敲，多容器场景根本无法维护。

### 6.2 基本工作流

```bash
vi docker-compose.yaml     # 1. 编写编排文件
docker compose up -d       # 2. 按文件创建并后台启动所有服务
```

> **【补充】** 原文此处只给了这两个命令，没有给出 compose 文件的具体内容与字段说明，属于待补章节（见第十节）。

### 6.3 常用命令

```bash
docker compose up -d        # 后台启动（并创建）所有服务
docker compose down         # 停止并移除容器、网络
docker compose logs -f      # 聚合查看所有服务日志
docker compose ps           # 查看本项目下容器的状态
```

> 新旧写法差异：`docker-compose`（v1，带横杠）与 `docker compose`（v2，作为 docker 子命令）。Docker Desktop 现在自带 v2，**建议统一用 `docker compose`**。

---

## 七、实战：部署前端项目

原文的「docker 部署前端项目」章节**以流程截图为主，未附文字步骤**。结合前文（镜像 → 容器 → 挂载）可还原出的完整链路是：

1. **构建产物**：前端项目 `npm run build`，得到 `dist/` 目录。
2. **准备 Nginx 镜像**：`docker pull nginx`。
3. **把产物送进容器**，两种方式：
   - 开发/调试：绑定挂载 `-v /o/.../dist:/usr/share/nginx/html`；
   - 生产：`docker cp dist/. <容器>:/usr/share/nginx/html/`，或写 Dockerfile 用 `COPY dist /usr/share/nginx/html` 构建自有镜像。
4. **端口映射并验证**：`docker run -d -p 8080:80 ...`，浏览器访问 `http://localhost:8080/`。
5. **多服务场景**（前端 + 后端 + 数据库）改用 `docker compose up -d` 统一编排。

> **【补充】** 上面第 1–4 步是整理者依据原文前几章推演的通用流程，原文该章节具体截图内容本文未收录，请以原文为准。

---

## 八、命令速查表

### 8.1 镜像管理（Images）

| 命令 | 说明 |
|---|---|
| `docker pull [image]:[tag]` | 从仓库拉取镜像，例 `docker pull nginx:latest` |
| `docker images` | 列出本地所有已下载镜像 |
| `docker rmi [image_id]` | 删除指定镜像 |
| `docker build -t [name] .` | 按当前目录的 Dockerfile 构建镜像 |
| `docker tag [source] [target]` | 给镜像打标签（常用于推送私有仓库） |
| `docker pull --platform=[arch] [image]` | 拉取指定 CPU 架构镜像 |
| `docker scout quickview [image]` | 查看镜像漏洞与升级建议 |

### 8.2 容器运行与生命周期（Run & Lifecycle）

| 命令 | 说明 |
|---|---|
| `docker run -d -p [host]:[container] [image]` | 后台运行并映射端口，例 `-p 8080:80` |
| `docker run -it [image] /bin/bash` | 以交互式终端进入容器 |
| `docker run -it --rm [image]` | 退出即自动删除的一次性容器 |
| `docker run -d --name [name] [image]` | 指定容器名称 |
| `docker start/stop/restart [container]` | 启动 / 停止 / 重启容器 |
| `docker kill [container]` | 强制杀掉容器进程 |
| `docker rm [container]` | 删除已停止容器，`-f` 可删运行中的 |

### 8.3 状态与日志（Status & Logs）

| 命令 | 说明 |
|---|---|
| `docker ps` | 查看运行中的容器，`-a` 查看全部 |
| `docker logs -f [container]` | 实时查看日志（类似 `tail -f`） |
| `docker top [container]` | 查看容器内运行的进程 |
| `docker stats` | 实时监控所有容器 CPU / 内存占用 |
| `docker inspect [container]` | 查看容器底层详细信息（JSON） |

### 8.4 交互与调试（Interaction）

| 命令 | 说明 |
|---|---|
| `docker exec -it [container] bash` | 进入运行中的容器（最常用） |
| `docker exec [container] ps -ef` | 查看容器内进程 |
| `docker cp [src] [container]:[dest]` | 宿主机文件拷贝到容器内 |
| `docker cp [container]:[src] [dest]` | 容器内文件拷贝到宿主机 |
| `docker diff [container]` | 查看容器文件系统发生的变更 |
| `cat /etc/os-release` | 容器内查询 Linux 发行版（决定用 apt/apk/yum） |

### 8.5 数据卷（Volume）

| 命令 | 说明 |
|---|---|
| `docker volume create [name]` | 创建命名卷 |
| `docker volume list` | 列出所有卷 |
| `docker volume inspect [name]` | 查看卷的详细信息（含挂载点） |
| `docker volume rm [name]` | 删除指定卷，`-f` 强制 |
| `docker volume prune -a` | 清理无容器引用的卷 |

### 8.6 清理与维护（System）

| 命令 | 说明 |
|---|---|
| `docker system df` | 查看 Docker 磁盘占用 |
| `docker system prune` | 清理所有未使用资源（**慎用**） |
| `docker network prune` | 清理所有未使用网络 |

### 8.7 Docker Compose

| 命令 | 说明 |
|---|---|
| `docker compose up -d` | 后台启动 compose 中定义的所有服务 |
| `docker compose down` | 停止并移除容器、网络 |
| `docker compose logs -f` | 查看所有服务聚合日志 |
| `docker compose ps` | 查看 compose 项目下容器状态 |

---

## 九、踩坑与 FAQ

### Q1：`wsl --update` 报 `已禁止(403)`、错误代码 `Wsl/UpdatePackage/0x80190193`

**原因**：VPN / 代理软件（Clash、Shadowsocks 等）劫持了 WSL 的更新下载请求。
**解决**：暂时关闭代理后重试；仍失败可加 `--web-download` 强制走网络下载。

### Q2：Docker Desktop 装完 C 盘爆满

**原因**：默认把程序、WSL 数据盘（镜像 + 卷）、Windows 容器数据都放 C 盘，镜像层动辄几十 GB。
**解决**：安装前用命令行参数 `--installation-dir` / `--wsl-default-data-root` / `--windows-containers-default-data-root` 指定到其他盘（见 2.5）。

### Q3：`docker run nginx` 之后终端卡住，没法做别的操作

**原因**：前台模式，容器日志独占终端。
**解决**：加 `-d` 后台运行，再用 `docker ps` / `docker logs` 观察。

### Q4：浏览器打开 `localhost:8080` 打不开

排查顺序：
1. `docker ps` 确认容器是 `Up` 状态且端口映射列为 `0.0.0.0:8080->80/tcp`；
2. 检查 `-p` 用的是不是**半角冒号**、方向是不是「宿主机:容器」；
3. 检查宿主端口是否被其他程序占用；
4. 看 `docker logs <容器>` 里应用是否真的监听在容器端口（如 80）上。

### Q5：`-v` 挂载后，容器里原来的文件都没了 / 目录是空的

**原因**：绑定挂载是**覆盖**，宿主目录内容会盖掉镜像里该路径的原内容；宿主目录为空 → 容器内也为空。
**解决**：先往宿主目录里放文件（如 `index.html`）。这就是「绑定挂载前要先自己创建并放好文件」的原因。

### Q6：容器内 `ls` 看不到宿主机的真实路径

**原因**：Docker 的安全隔离机制——容器只能看到**挂载点**（如 `/usr/share/nginx/html`），感知不到它背后对应的宿主机路径。
**解决**：从外部查询：`docker inspect <容器>` 看 `Mounts` 字段；或在 Docker Desktop 图形界面点容器 → **Inspect** 标签 → 搜 `Mounts`。

### Q7：Windows 上 `docker volume inspect` 给的路径在资源管理器里打不开

**原因**：那是 WSL2 虚拟机内部路径，不是 Windows 路径。
**解决**：改用
```
\\wsl$\docker-desktop\...          # 或
file://wsl.localhost/docker-desktop/mnt/docker-desktop-disk/data/docker/volumes/<卷名>/_data/
```
Linux 下则需 root 权限访问 `/var/lib/docker/volumes/...`。

### Q8：`docker volume prune -a` 删不掉某个卷

**原因**：仍有（哪怕已停止的）容器引用它，prune 出于保护不会删。
**解决**：`docker ps -a --filter "volume=<卷名>"` 找到容器 → `docker rm` → 再删卷；或 `docker volume rm -f` 强删（数据不可恢复）。

### Q9：`docker rmi` 删不掉镜像 / `docker rm` 删不掉容器

- 镜像被容器引用 → 先删容器，或 `docker rmi -f`；
- 容器在运行 → `docker rm -f`。

### Q10：多个容器挂同一个卷，数据会乱吗

不会「自动乱」，但要预防**并发写冲突**（多容器同时写同一文件）、**权限不一致**（UID 不同）、**孤儿卷**（先删容器再删卷）。

### Q11：容器里装软件提示找不到 `apt`

**原因**：基础镜像不是 Debian 系。
**解决**：先 `cat /etc/os-release` 确认发行版，Alpine 用 `apk add`，CentOS/RHEL 用 `yum`/`dnf`。

---

# 十、原文遗留待补点（Deepseek整理时生成）

整理时发现原文尚有缺口，若要把它做成一份完整知识库，建议补齐：

| # | 缺失内容 | 影响 | 建议补充 |
|---|---|---|---|
| 1 | **Docker Compose 的 YAML 文件示例与字段说明**（services / image / ports / volumes / depends_on / networks） | 该章只有两个命令，读者无法真正上手编排 | 补一个「Nginx + 后端 + MySQL」的最小可运行 `compose.yaml` |
| 2 | **「部署前端项目」章节全是截图，无文字步骤** | 无法检索、无法复现 | 补 Dockerfile + build + 运行 + 验证的完整命令清单 |
| 3 | **镜像构建（Dockerfile）章节缺失** | 路线从「用别人的镜像」直接跳到部署，缺「做自己的镜像」 | 补 Dockerfile 基础指令（FROM/WORKDIR/COPY/RUN/EXPOSE/CMD）与多阶段构建 |
| 4 | **容器网络（bridge / host / 自定义网络、容器互访）未涉及** | 多容器编排必然要用网络 | 补 `docker network` 与 Compose 中的服务名解析 |
| 5 | **`docker compose` 与 `docker-compose` 版本差异**未说明 | 照抄旧命令可能报 `command not found` | 统一到 v2 写法 |
| 6 | 示例 `nginx:1.0.0` 标签在 Docker Hub 上并不存在 | 直接照抄会拉取失败 | 改用 `nginx:latest` / `stable` / 实际版本号 |
| 7 | `-p {宿主机端口号}：{容器端口号}` 用了全角冒号 | 复制粘贴会报错 | 统一为半角 `:` |

---

# 附：外部知识链接汇总

| 类型 | 链接 |
|---|---|
| 视频教程（40 分钟 Docker 实战） | https://www.bilibili.com/video/BV1THKyzBER6 |
| Docker 官网 | https://www.docker.com/ |
| Docker 官方入门文档 | https://docs.docker.com/get-started/ |
| Docker Desktop 下载指引 | https://docs.docker.com/get-started/get-docker/ |
| 指定安装目录参考教程 | https://www.cnblogs.com/shengmingboai/p/18884882 |
| 关于命名卷路径的讨论 | https://www.qianwen.com/share/chat/6c2b3e814ab74db8b439370a24ee8a55 |
| 命名卷路径视频 | https://www.bilibili.com/video/BV1KjyPBPEVx |

---



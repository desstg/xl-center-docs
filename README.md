# XL Center

一个 **Emby 风格的本地媒体管理中心**，专为「已经用 NFO 刮削好的视频库」设计。扫描本地目录，把影片（含海报、简介、演员、标签、番号等元数据）入库，提供海报墙浏览、详情页、多条件搜索和浏览器播放。

- 后端：Node.js + TypeScript + Express + SQLite（better-sqlite3）
- 前端：Vue 3 + Vite
- 部署：Docker（适配群晖 NAS 等任何支持 Docker 的设备）

---

## ✨ 功能特性

- **海报墙 + 详情页**：竖版海报墙、横版缩略图两种视图，详情页展示海报/画报/剧照、简介、演员、标签、类型、番号、编码等信息
- **多对多搜索**：按标题 / 番号 / 演员 / 标签 / 类型 / 文件名 / 年份 / 媒体库过滤，支持「全部匹配」和「任意匹配」
- **浏览器播放**：H264 直连播放、HEVC 转码播放（HLS）、STRM 流代理播放
- **演员归一化**：通过 `mapping_actor.xml` 把同一演员的多个别名归并到统一主名
- **媒体库管理**：网页里添加 / 删除媒体库、手动扫描、自动增量扫描
- **NFO 编辑 / 海报裁剪**：网页里直接编辑影片元数据并写回 NFO 文件、裁剪海报
- **码率 / 网速显示**：播放器实时显示视频码率和当前下载速度
- **可选登录**：默认无密码直接使用，也可通过环境变量开启账号密码登录

---

## 🚀 快速开始（Docker）

### 1. 准备媒体库目录

确保你的视频库是「每部影片一个文件夹」的结构（详见下方 [媒体库准备](#媒体库准备)）。

### 2. 编写 `docker-compose.yml`

```yaml
services:
  xl-center:
    image: desstg/xl-center:latest
    container_name: xl-center
    ports:
      - "8899:8899"
    volumes:
      - ./data:/app/data                # 数据持久化（数据库 + 缩略图缓存）
      - /volume1/媒体库:/media          # ← 改成你自己的媒体库目录（左侧宿主机路径）
    environment:
      # 登录账号密码（可选）：两个都填才启用登录；不设置则无登录，直接进主界面
      # - AUTH_USERNAME=admin
      # - AUTH_PASSWORD=admin123
    devices:
      # 核显透传，用于 VAAPI 硬件转码（可选，没有 Intel 核显就删掉下面两行）
      - /dev/dri:/dev/dri
    restart: unless-stopped
```

### 3. 启动

```bash
docker compose up -d
```

### 4. 打开网页

浏览器访问 `http://你的设备IP:8899`

### 5. 添加媒体库并扫描

1. 进入顶部导航「管理」→「媒体库」
2. 点「添加媒体库」，名称随意（如「电影」），**路径填容器内路径 `/media`**
3. 保存后点「扫描」，等待扫描完成即可看到海报墙

> ⚠️ **重要**：网页里填的是**容器内的路径 `/media`**，不是宿主机上的真实路径。宿主机路径通过 `docker-compose.yml` 的 `volumes` 左侧那一项映射进来，右侧 `/media` 固定不变。

---

## 📁 媒体库准备

扫描器会递归查找「包含视频文件的文件夹」，每个这样的文件夹视为一部影片。建议结构如下：

```
媒体库/
├── 影片A/
│   ├── 影片A.mkv              # 视频文件（也支持 mp4 / avi / ts / m2ts / webm / mov / m4v / flv / wmv / strm）
│   ├── 影片A.nfo              # Kodi 标准的 movie NFO（或 movie.nfo）
│   ├── poster.jpg             # 海报（竖版）
│   ├── fanart.jpg             # 画报（横版背景，也支持 backdrop.jpg）
│   ├── thumb.jpg              # 缩略图
│   ├── logo.png               # Logo（可选）
│   ├── 字幕.srt               # 字幕（srt / ass / ssa / sub / vtt）
│   └── extrafanart/           # 剧照目录（可选）
│       ├── 1.jpg
│       └── 2.jpg
├── 影片B/
│   └── ...
```

- **视频格式**：`.mp4` `.mkv` `.avi` `.ts` `.m2ts` `.webm` `.mov` `.m4v` `.flv` `.wmv` `.strm`
- **字幕格式**：`.srt` `.ass` `.ssa` `.sub` `.vtt`
- **图片格式**：`.jpg` `.jpeg` `.png` `.webp`
- **NFO**：标准 Kodi `movie.nfo`（根节点 `<movie>`），支持 `title`、`num`（番号）、`originaltitle`、`year`、`plot`、`outline`、`rating`、`votes`、`runtime`、`mpaa`、`premiered`、`trailer`、`genre`、`tag`、`country`、`studio`、`director`、`credits`、`actor` 等字段

如果影片文件夹里没有任何 NFO，影片仍会入库，标题取文件夹名，其他元数据为空。

---

## 🌐 STRM 流播放

`.strm` 文件是一个纯文本文件，内容是一行流地址（HTTP/HTTPS URL）。播放时后端会读取该地址并代理转发给浏览器。

```
http://192.168.1.100:5244/d/xxx.mp4   # 例如 alist / cms 的直链
```

**注意事项**：

- STRM 里写的地址必须是**容器能访问到的**（公网地址、或与容器同网段的局域网地址）
- ⚠️ 不要写 `localhost` / `127.0.0.1`——在 Docker 容器里它指向容器自身，而不是宿主机
- 后端会自动跟随 302 跳转（alist 直链常见的跳转），外网浏览器无需直连内网

---

## 🎬 播放与转码

| 视频类型 | 播放方式 |
|---------|---------|
| H264（含 4K） | 直接播放，无需转码 |
| HEVC 1080p | VAAPI 硬件转码（有 Intel 核显）或软编转码 |
| HEVC 4K | 需要较强 Intel 核显（如 UHD 630 / Iris Xe）才能硬解转码；否则降级直连（可能无法播放） |

转码依赖 **Intel 核显 VAAPI**，需要：

1. 宿主机有 Intel 核显
2. `docker-compose.yml` 里透传 `/dev/dri`
3. 镜像内已内置 ffmpeg + Intel iHD 驱动

没有核显也能正常使用（H264 直连、HEVC 1080p 软编），只是 HEVC 4K 播不了。程序启动时会自动检测 VAAPI 设备，有就用、没有自动降级。

---

## 🔐 登录（可选）

默认**没有登录**，打开网页直接使用。如需开启账号密码登录，在 `docker-compose.yml` 里设置两个环境变量：

```yaml
environment:
  - AUTH_USERNAME=admin
  - AUTH_PASSWORD=admin123
```

- 两个都设置 → 启用登录，访问时需要输入账号密码
- 留空 / 不设置 → 无登录，直接进主界面

登录使用内存 token（重启失效），登录状态保存在浏览器 localStorage。

---

## 📦 获取方式

- **Docker Hub 镜像**：`docker pull desstg/xl-center:latest`
- **源码仓库**：当前为私有仓库，暂不公开源码。如需源码或有定制需求，欢迎通过 [Issues](https://github.com/desstg/xl-center-docs/issues) 联系。

---

## ❓ 常见问题

**Q：扫描后海报墙是空的？**
A：检查媒体库路径是否填了容器内路径 `/media`（而不是宿主机路径），且媒体库目录结构正确（每部影片一个文件夹）。

**Q：HEVC 4K 播不了？**
A：这是硬件限制。HEVC 4K 转码需要较强 Intel 核显，你的设备核显不够时无法硬解转码。H264 不受影响。

**Q：STRM 播放不了？**
A：确认 STRM 文件里写的是容器能访问到的地址（不是 `localhost`），且该地址有效。

**Q：如何更新到新版本？**
A：`docker compose pull && docker compose up -d`（数据在 `/app/data` 挂载卷里，更新不会丢失）。

**Q：剧集（电视剧）支持吗？**
A：当前版本只支持电影库，剧集扫描仍在开发中。

---

## 📄 许可证

本软件通过 Docker Hub 公开分发，供个人学习与使用。源码仓库暂不公开。

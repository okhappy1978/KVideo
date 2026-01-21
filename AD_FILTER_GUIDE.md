# 🛡️ 广告过滤功能使用指南

KVideo 新增了强大的 M3U8 广告过滤系统，支持多种过滤模式，可有效去除视频中插入的切片广告。

## ✨ 功能特性

- **多模式选择**：支持关闭、关键词过滤、智能启发式过滤(Beta)和激进模式。
- **UI 集成**：在播放器设置菜单中直接切换模式，实时生效。
- **自定义关键词**：支持通过环境变量扩展过滤关键词。
- **高性能**：基于流式处理，对播放加载速度几乎无影响。

## 🎮 使用说明

1. 打开任意视频播放。

2. 点击播放器左上角的 **(···)** 按钮。

3. 在 **广告过滤** 下拉菜单中选择模式：
   - **关闭**：不过滤任何内容（默认）。
   - **关键词**：仅过滤 URL 中包含特定广告关键词（如 `adjump`, `pre-roll`）的片段。安全稳定，误杀率极低。
   - **智能(Beta)**：推荐使用。结合关键词、文件名特征、时间戳跳变标记 (DISCONTINUITY) 和 HLS 标签进行综合评分。能识别大多数无明显关键词的广告。
   - **激进**：降低判定阈值，过滤更多疑似广告，但可能会误伤正片。仅在智能模式无效时尝试。
   
   > ⚠️重要提示：添加关键词时，请勿使用类似`ad` `ads` `video` `20260118` 这类简短相对通用的词汇，会造成误杀率极高。

---

## 🛠️ 部署配置指南

你可以通过环境变量 `NEXT_PUBLIC_AD_KEYWORDS` 自定义额外的广告关键词。

### 1. 本地开发 (Local Development)

在项目根目录的 `.env.local` 文件中添加。支持多行格式（推荐使用引号包裹）：

```env
# 单行模式
NEXT_PUBLIC_AD_KEYWORDS=spam,promo,intro-ad

# 多行模式 (更清晰)
NEXT_PUBLIC_AD_KEYWORDS="
spam
promo
intro-ad
/unwanted-segment/
"
```

### 2. Docker 部署 (运行时热加载)

我们支持三种方式注入关键词，优先级从高到低：

1. **文件加载 (推荐)**：通过挂载文件并在运行时读取。
2. **运行时环境变量**：直接通过 `-e` 传入 `AD_KEYWORDS`。
3. **构建时环境变量**：构建镜像时打入的 `NEXT_PUBLIC_AD_KEYWORDS`。

#### 方案 A：挂载配置文件 (支持运行时修改)

创建一个关键词文件 `ad_keywords.txt`：
```text
spam
promo
/ad-segment/
intro
```

运行 Docker 时挂载该文件并设置 `AD_KEYWORDS_FILE` 环境变量：

```bash
docker run -d -p 3000:3000 \
  -v $(pwd)/ad_keywords.txt:/app/ad_keywords.txt \
  -e AD_KEYWORDS_FILE=/app/ad_keywords.txt \
  --name kvideo kuekhaoyang/kvideo:latest
```
*注：修改宿主机文件后重启容器即可生效，无需重新构建镜像。*

#### 方案 B：直接注入环境变量

```bash
docker run -d -p 3000:3000 \
  -e AD_KEYWORDS="spam,promo,intro-ad" \
  --name kvideo kuekhaoyang/kvideo:latest
```

### 3. Vercel 部署

1. 进入 Vercel 项目 Dashboard。
2. 点击 **Settings** -> **Environment Variables**。
3. 添加新变量：
   - **Key**: `NEXT_PUBLIC_AD_KEYWORDS`
   - **Value**: `spam,promo,intro-ad`
4. 重新部署项目以生效。

### 4. Cloudflare Pages 部署

1. 进入 Cloudflare Pages 项目 Dashboard。
2. 点击 **Settings** -> **Environment variables**。
3. 添加变量：
   - **Variable name**: `NEXT_PUBLIC_AD_KEYWORDS`
   - **Value**: `spam,promo,intro-ad`
4. 重新部署项目 (Retry deployment) 以生效。

---

## 🔍 技术实现细节

广告检测逻辑位于 `lib/utils/m3u8-ad-detector.ts`，采用分块评分机制：

1. **Block 解析**：将 M3U8 播放列表按 `#EXT-X-DISCONTINUITY` 分割成多个块。
2. **特征学习**：分析最长的块（通常是正片）提取特征，如文件命名模式、路径前缀等。
3. **评分系统**：
   - **HLS 标签**：`#EXT-X-CUE-OUT`/`IN` (+10分，确认为广告)
   - **路径前缀**：与正片路径不一致 (+5分，高度疑似)
   - **关键词**：URL 包含已知广告词 (+2.5分)
   - **文件名模式**：与正片命名规则不符 (+1.5分)
4. **决策过滤**：
   - **智能模式**：移除评分 >= 5.0 的块。
   - **激进模式**：移除评分 >= 3.0 的块。

# MusicDatabase

### 这是什么

这是我的音乐收集库：自动把**音频 + 封面 + 歌词**下载下来，按 `【歌名-歌手】` 文件夹归档进 `main` 分支。

> [!important]
>
> **Note：这不是 GitHub 压力测试，也不是什么 commit 农场。**
> 
> commit 数量高，是因为**每首歌都会产生 3~4 个 commit** —— commit 是流水线的运行日志，不是产物。产物是音乐库本身。
>
> 这里面的歌并不全是我听过的 —— 还有我的朋友们。

### 它实际在做什么

```
报歌（Collector Worker，Cloudflare）
  │  先查 main/list.txt，已收集过的直接跳过
  └─ 把 <歌名>-<歌手>.json 推到 goodurl 分支           → commit "collector: <歌名> - <歌手>"
       └─ push 触发 .github/workflows/archive.yml
            ├─ 从 QQ 音乐 / 网易云 CDN 下载 音频 / 封面 / 歌词（自动重试 + 内容校验）
            ├─ 封面嵌入音频 tag（同时保留独立封面文件），写入标题 / 艺术家
            ├─ 存入 main 分支 <歌名>-<歌手>/ 文件夹     → commit "archive: 归档 1 首歌曲 <歌名>"
            │    （song.mp3 + cover.jpg + info.json [+ 歌词]）
            └─ 硬失败（下载失败 / 过小 / 时长过短）→ 整首丢弃，JSON 留在 goodurl，下轮重试

每天 03:50 (UTC+8)，.github/workflows/cleanup.yml：
  ├─ 删除 goodurl 里已归档的 JSON                      → commit "cleanup: 删除已归档的 JSON [skip archive]"
  ├─ 重建 main/list.txt（已收集目录，报歌去重用）        → commit "chore: 更新已收集目录 (N 首)"
  └─ 扫描 info.json，补抓缺失的封面 / 歌词              → commit "refetch: 补抓缺失的封面/歌词 [skip archive]"
```

所以一首歌 ≈ **collector 1 个 + archive 1 个 + cleanup 1 个 commit**，外加每天几笔维护 commit。

### 为什么一首歌一个 commit，不攒一批一起提交？

1. **每首歌独立成败。** 下载失败、文件过小、时长过短 → 只有那一首被丢弃，JSON 留在 `goodurl` 下轮自动重试，不连累其他歌。
2. **软失败留档。** 封面 / 歌词链接坏了不挡归档：歌曲照常入库，缺什么写进 `info.json`（`picOk` / `lrcOk` / `coverError` / `lyricError`）。`info.json` 是唯一的"缺失状态"来源，重试责任全在每天一次的 refetch job（每条最多重试 5 次，超了就在 `missing.txt` 留档）。
3. **可追溯。** 每首歌一个 commit，坏在哪首、什么时候坏的，翻历史就能看到；`info.json` 里还有 `reportedAt` / `archivedAt` 时间戳。回滚和排查都按"首"为单位。

### 为什么用 goodurl / main 两个分支？

- `goodurl` = **收件箱**：只有 JSON，很轻。collector 只写这里。
- `main` = **成品库**：只有归档完的 `【歌名-歌手】` 文件夹和 `list.txt`。
- 两边解耦：collector 不碰 main，archive 不重写 goodurl（只有 cleanup 删），避免两边抢同一个文件。
- cleanup 的 commit 带 `[skip archive]` 标记——删 JSON 的那次 push 不会反过来又空跑一遍 archive。

### 为什么把音乐库放在 GitHub 上？

- 免费，而且自带版本历史：每首歌一个 commit，等于自动 changelog。
- 重活（下载、校验、打 tag、补抓）全在 GitHub Actions 上跑，我不需要养一台 7×24 的服务器。
- 没有额外 secret：两个 workflow 都只用仓库自带的 `GITHUB_TOKEN`（`contents:write`）。

### 给后来的你 / 路人

如果你是因为看到"一个月 300+ commit"皱着眉头点进来的：抱歉，也谢谢。上面就是全部真相。

想验证？打开 `.github/workflows/archive.yml` —— 触发器是 goodurl 的 JSON push，写进 main 的只有歌曲文件夹和 `list.txt`；打开 `goodurl` 分支看看，除了一两个待归档的 JSON，什么二进制都没有。

### 另请参阅

[Collector Worker](https://github.com/lingyicute/MusicCollector)

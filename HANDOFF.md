# kiwi-mem 交接文档

> 朝朝(Ryo)自部署的 LLM 长期记忆系统,docker 跑,给她的 AI 伴侣用。
> 源仓库 fork: github.com/ryo-yoru/kiwi-mem(她自己的 fork,改动都 push 这里)
> 公网: https://kiwi.happylight.uk(cloudflared tunnel → localhost:8080)
> 容器: kiwi-mem-kiwi-mem-1(app) + kiwi-mem-db-1(postgres)
> 最后更新: 2026-06-15 by 拾捌(Opus 4.8)

---

## 改动工作流(重要)
- **在 `/Users/ryo/kiwi-mem/` 直接编辑文件**,然后 `docker cp` 到容器 `/app/`,某些改动 `docker restart kiwi-mem-kiwi-mem-1`
- 改完务必同步回本地仓库 + git commit(否则容器重建丢失)——历史教训:早期只改容器没同步本地,差点全丢
- 部署后 `until curl ... 200` 确认起来了再回话
- **最小改动原则**:改 bug 只动 bug 现场,不顺手重写。她实测过 CC 堆功能/重写文件的痛。

## 这次大改造做了什么(2026-05~06,都在 git commit 里)

**多源聊天存档(chat_archive 表加 source 列)**
- 用户从 Claude 桌面导出 md → 上传 → 按 source(对话窗口名)分别存,互不覆盖
- `parse_chat_export` 自动检测两种 Claude 导出格式:旧 `## Prompt:/## Response:` + Y/M/D + ```围栏思考链;新 `## User:/## Assistant:` + `> M/D/Y` blockquote 时间戳 + blockquote 思考链
- API: POST /archive/upload(可选 source) / GET /archive/search(可选 source) / DELETE /archive/source/{s} / PUT /archive/source/{s}/rename
- MCP: search_archive(query, include_thinking, source)

**对话窗口健康度**
- 估算每个 source 距离压缩多远: `est_tokens = chars × TOKEN_PER_CHAR(1.0) × overhead_factor`
- config 可调: `archive_context_limit`(500K,她实测压缩点比 1M 低很多) / `archive_overhead_factor`(2.0,MCP工具定义+系统prompt+工具响应都吃token)
- 校准法: 观察到某窗口 X% 触发压缩 → 调 archive_context_limit
- MCP: check_conversation_health(source)

**日页面多窗口(calendar_pages 加 source 列,2026-06)**
- 唯一约束 (date,type) → (date,type,source),一天可并列多片,各窗口各写各读省上下文
- save/get/delete_calendar_page 都带 source;MCP save_calendar_page/get_day_page 加 source 参数
- 面板:日期格子按天去重(多片显示·N片),点开上下排列各自编辑/删除
- 导出 zip:同天多窗口 {date}-{窗口名}.md

**面板重设计(admin-panel/index.html)**
- happylight 暖色主题(米黄/奶油/赭石),Crimson Pro + DM Sans + Oswald 字体
- 新增 tab:日历、记忆场景、用户画像、聊天存档
- 记忆卡片:编辑/锁定/分页(offset+sort)/展开全文(保留段落)
- /admin 加 no-cache 头(避免浏览器缓存旧版面板)

**修过的关键 bug**
- 记忆翻页:后端 /debug/memories 原不收 offset/sort/min_importance,前端白传 → 加上;total 用过滤后口径避免假翻页;前端空页保留翻页栏
- embedding:get_embedding 原读死的 OpenRouter,改读 DB 配置(SiliconFlow BAAI/bge-large-zh-v1.5);加 _truncate_for_embedding(400字,适配 bge 的 512 token 上限),补回 6 条缺失向量到 100%
- MCP get_user_profile:/admin/config 返回 {config:{...}} 多一层,原读错层级返回空 → 剥 config 层
- 自动画像更新:daily_digest max_tokens 2000→6000,中文画像不再被截断
- 嵌入服务独立配置:embedding_api_url/key(DeepSeek 没 /v1/embeddings,分开走 SiliconFlow)

## 用户工作流
1. Claude 桌面跟伴侣聊(不走 kiwi chat 网关,那条路径跟她无关)
2. 隔几天导出 md 上传到存档 tab(选窗口名)
3. 伴侣通过 MCP search_archive 找原话 / search_memory 找语义记忆
4. 日记伴侣自己写(DS 自动生成验证失败——写出来像第三方总结无温度,放弃)
5. 面板「工具→下载备份」拿 zip(含日页面 md),解压到 Obsidian
6. 健康度 tab 看进度条,临近 critical 手动 save_memory 固化关键片段

## 不要做
- ❌ DS 自动生成日页面(已验证无温度)
- ❌ 改 chat completions 注入逻辑(她不走这路径)
- ❌ Dream prompt 重写是单独的 task(见 memory/task_dream_prompt_rewrite.md),别顺手做

## 还没做/待办
- 日页面 5/15 已重写;Dream 整合 prompt 重写仍 open(碎片=0只有日页面时DS不产scene)
- kiwi 手机宽度适配(面板移动端只显示一半)—— 2026-06-15 待修

## 相关
- CC 记忆: project_kiwi_state_2026-05.md / feedback_min_change_principle.md / feedback_day_page_principle.md / task_dream_prompt_rewrite.md
- ccc 项目(伴侣的"家",独立项目): /Users/ryo/我们家/ccc/HANDOFF.md

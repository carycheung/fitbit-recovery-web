# Fitbit Air → Recovery（自算 WHOOP 式恢复分）

纯前端网页。用 Google Health API 拉取 Fitbit Air 数据，在浏览器里算出对标 WHOOP 的四组指标：

- **Recovery 恢复度**（0–100%）— HRV/静息心率/睡眠/呼吸率相对个人基线；绿≥67 / 黄34–66 / 红<34
- **Sleep 睡眠表现**（0–100%）— 睡够度(占60%) + 睡眠效率 + 深睡/REM 修复占比；85+ 优秀 / 70–84 尚可 / <70 不足
- **Strain 当日负荷**（0–21 对数刻度）— 各心率区间停留时间加权的心血管消耗；<10 轻松 / 10–14 适中 / 14–18 偏高 / 18+ 极限
- **Body Age 身体年龄** — VO₂max(或静息心率+年龄反推) + 静息心率 + 日均步数，近30天滚动、周尺度缓变，带「变年轻/变老」pace 指示

每个指标都有中文解读，还有可展开的「指标怎么看」总说明。
**没有后端，数据只在你浏览器内存里流转，不写进这个仓库、不上传任何服务器。**

UI 对标 WHOOP / goose：黑底 + 三圆环（Recovery 大 + Sleep/Strain 小）+ 身体年龄 gauge + 三色词汇，移动端优先。

## 打开即用（缓存机制）

Recovery 一天只出一次分（睡醒后），所以不需要频繁联网：
- 拉过的数据缓存在浏览器 `localStorage`，**再次打开先秒显缓存里的「今天」**，离线也能看。
- **登录态也记住了**：授权拿到的 access token 存在浏览器（约 1 小时有效），刷新页面直接复用、不再弹授权；过期后自动静默续期一次，无需手动点登录。
- 只有当缓存里最新一天还不是今天时，才**后台静默更新**一次。
- 想强制刷新：点右上角 `⚙ → 重新拉取`。
- 首次访问会自动拉最近 60 天（用来建立 HRV/RHR 的个人基线，否则算不出今天的分），hero 卡永远显示最新一天=今天，趋势图默认近 30 天、可切 7 天 / 全部。

## 一次性设置（用 Fitbit 绑定的那个 Google 账号：carycheung321@gmail.com）

1. **建 GCP 项目并启用 API**
   - https://console.cloud.google.com/projectcreate 建项目
   - 启用 Google Health API：https://console.cloud.google.com/apis/api/health.googleapis.com

2. **配置 OAuth 同意屏幕**（新版在「Google Auth 平台」下）
   - https://console.cloud.google.com/auth/overview → 开始配置
   - App 名随意、支持邮箱选自己、**受众类型 External**、开发者邮箱填自己
   - 「受众 / Audience」→ **Test users 里把 carycheung321@gmail.com 加进去**
   - 发布状态留在 **Testing** 即可（无需 Google 审核）

3. **创建 Web 客户端**
   - 「Google Auth 平台 → 客户端 / Clients」→ 创建客户端
   - Application type = **Web application**
   - **Authorized JavaScript origins** 填部署地址：`https://carycheung.github.io`
     （本地测试再加一条 `http://localhost:8000`）
   - 创建后复制 **Client ID**（形如 `xxx.apps.googleusercontent.com`）

4. **部署 & 打开**
   - 把 `index.html` push 到 GitHub 仓库，开启 Pages（Settings → Pages → 从 main 分支）
   - 打开 `https://carycheung.github.io/<repo>/`
   - 在页面「连接」里粘贴 Client ID（会记住）→ 选日期区间 → 「用 Google 登录并拉取」
   - 浏览器弹 Google 授权 → 选 carycheung321 → 同意（会有“未验证应用”警告，点继续）
   - 自动出恢复分 + 趋势图

## 本地先测（可选）

```bash
cd fitbit-recovery-web
python3 -m http.server 8000
# 浏览器开 http://localhost:8000  （记得把 http://localhost:8000 加进 Authorized JS origins）
```

## 需要的权限（只读）
- `googlehealth.health_metrics_and_measurements.readonly`（HRV / 静息心率 / 呼吸率）
- `googlehealth.sleep.readonly`（睡眠）
- `googlehealth.activity_and_fitness.readonly`（活动区间→Strain / 步数 / VO₂max→身体年龄）

> 新增了第三个 scope。老用户第一次打开会自动弹一次授权补上「活动数据」权限；
> 若没弹或被拒，点 ⚙ 里的「授权活动数据」按钮即可。没授权时 Recovery/Sleep 仍照常出，
> 只是 Strain / 身体年龄显示无数据。

## 算法
每项指标对个人滚动基线（默认30天）求 z 分：HRV 用 ln(rmssd)、RHR/呼吸取反（越低越好），
加权（HRV .55 / RHR .25 / 睡眠 .12 / 呼吸 .08）后经 logistic 映射到 0–100%。参数页面可调。
需至少 7 天历史建基线。HRV 默认取「深睡 rMSSD」（最接近 WHOOP 口径），可切全天平均。

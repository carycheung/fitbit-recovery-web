# Fitbit Air → Recovery（自算 WHOOP 式恢复分）

纯前端网页。用 Google Health API 拉取 Fitbit Air 的 HRV / 静息心率 / 呼吸率 / 睡眠，
在浏览器里算出 0–100% 的恢复分（绿≥67 / 黄34–66 / 红<34）。
**没有后端，数据只在你浏览器内存里流转，不写进这个仓库、不上传任何服务器。**

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

## 算法
每项指标对个人滚动基线（默认30天）求 z 分：HRV 用 ln(rmssd)、RHR/呼吸取反（越低越好），
加权（HRV .55 / RHR .25 / 睡眠 .12 / 呼吸 .08）后经 logistic 映射到 0–100%。参数页面可调。
需至少 7 天历史建基线。HRV 默认取「深睡 rMSSD」（最接近 WHOOP 口径），可切全天平均。

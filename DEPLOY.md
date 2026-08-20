# VPS 应用部署任务书

> 本文档自包含: 所有资源都在本仓库里, 按步骤执行即可。
> 目标 VPS: 107.172.208.174 (登录方式用户会提供)

## 任务总览

1. 部署"趋势漏斗"工作台到 VPS, nginx 反代到 `/funnel/`
2. 给"NextDay 台账"(github.com/asjio/nextday)做移动端卡片化适配 + PWA
3. 给"VPS 管理门户"页面加 PWA
4. 配置 Server酱 微信通知(定时任务跑完推送结果到个人微信)
5. 三个页面都能手机"添加到主屏幕"当 APP 用

## 资源清单(本仓库内)

| 资源 | 路径 | 用途 |
|------|------|------|
| 门户图标 | `icons/portal-192.png` / `icons/portal-512.png` | 门户 PWA 图标 |
| NextDay图标 | `icons/nextday-192.png` / `icons/nextday-512.png` | NextDay PWA 图标(深蓝底红金K线, 已按A股红涨绿跌配色) |
| 通用SW | `pwa/sw.js` | Service Worker 模板(网络优先+缓存兜底) |
| 通知模块 | `code/notifier.py` | Server酱微信通知函数 |

下载方式: `wget https://raw.githubusercontent.com/asjio/vps-app-deploy/main/<路径>`

漏斗图标不用下载, 已在 trend-funnel 仓库的 `funnel/static/` 里。

## 需要用户提供

1. **Server酱 SendKey**: 让用户打开 https://sct.ftqq.com 微信扫码登录, 复制 SendKey(格式 `SCT...`)

---

## 任务一: 部署 trend-funnel

代码已做好移动端卡片化适配 + PWA, 直接拉取部署, 不用改代码。

```bash
cd /root
git clone https://github.com/asjio/trend-funnel.git
cd trend-funnel
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### systemd 服务

创建 `/etc/systemd/system/trend-funnel.service`:

```ini
[Unit]
Description=trend-funnel workbench
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/trend-funnel
ExecStart=/root/trend-funnel/.venv/bin/python -m funnel.web
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now trend-funnel
```

### nginx 反代

`/etc/nginx/sites-available/default` 的 server block 内加:

```nginx
location /funnel/ {
    proxy_pass http://127.0.0.1:8768/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

`nginx -t && systemctl reload nginx`

注意: 代码里 fetch/manifest/sw 全用相对路径, 反代子路径不用改代码。

### cron 定时

```
30 15 * * 1-5 cd /root/trend-funnel && .venv/bin/python -m funnel.main >> data/cron.log 2>&1
```

---

## 任务二: NextDay 台账移动端适配 + PWA

代码仓库: `github.com/asjio/nextday` (公开), clone 到 VPS 后改 `nextday_v2/web.py`。

### 2.1 改造清单(照搬 trend-funnel 的成熟模式)

1. head 区加 PWA 标签:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<meta name="theme-color" content="#2f6fed">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-title" content="NextDay">
<link rel="manifest" href="manifest.json">
<link rel="apple-touch-icon" href="static/icon-192.png">
```

2. 新建 `nextday_v2/static/` 目录, 放入:
   - `icon-192.png` / `icon-512.png` (从本仓库 icons/ 下载)
   - `manifest.json` (参考下方模板)
   - `sw.js` (从本仓库 pwa/sw.js 下载)

```json
{
  "name": "NextDay v2 动量选股台账",
  "short_name": "NextDay",
  "start_url": ".",
  "scope": ".",
  "display": "standalone",
  "background_color": "#f7f6f4",
  "theme_color": "#2f6fed",
  "icons": [
    {"src": "icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable"},
    {"src": "icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable"}
  ]
}
```

3. web.py 后端加三个路由:
   - `app.mount("/static", StaticFiles(directory=static_dir))` 提供图标
   - `@app.get("/manifest.json")` 返回 manifest 内容
   - `@app.get("/sw.js")` 返回 sw 内容(必须根路径, 否则 scope 不对)

4. **窄屏卡片化**(核心, 参考 trend-funnel web.py 的 `@media (max-width:720px)` 段):
   - `.scrollbox` / 宽表格在窄屏 `display:none`
   - 每个表格旁加 `.m-cards` 容器, JS 渲染函数里同步输出卡片版
   - 卡片结构: 股票名+代码+现价一行, 指标用 `grid-template-columns:repeat(3,1fr)`, 触发原因单独一行
   - 预测台账表 / 对账历史表 / 回测表 全部做卡片版
   - 涨红跌绿底色继承现有 `.row-fall` 配色语义

5. JS 里所有 `fetch("/api/...")` 改相对路径 `fetch("api/...")`, 兼容 nginx 子路径反代

6. nginx 反代 `/nextday/` -> 8767 (端口以实际 web.py 为准), 配置同任务一

7. 改完推到 GitHub: `git push`(用户账号), 保持仓库与 VPS 同步

### 2.2 微信通知接入

把本仓库 `code/notifier.py` 复制到 `nextday_v2/notifier.py`, 填入用户给的 SendKey。
在 `nextday_v2/main.py` 的 `main()` 末尾调用:

```python
from .notifier import send, NOTIFY_SENDKEY
# main() 末尾, result 生成后:
if "xxxxxxxx" not in NOTIFY_SENDKEY:
    lines = [f"**大盘** 指数{m['index_close']:.0f} 闸门:{'开' if m['gate_open'] else '关'}"]
    if result["gate_open"]:
        lines.append(f"**信号** TOP{len(result['predictions'])} 买入{result['buy_date']} 卖出{result['sell_date']}")
        for p in result["predictions"][:5]:
            lines.append(f"{p['rank']}. {p['name']}({p['code'][-6:]}) 动量{p['momentum20']:+.1f}%")
    else:
        lines.append(f"**空仓** {result['gate_closed_reason']}")
    send(f"NextDay v2 信号报告 {today}", "\n".join(lines))
```

---

## 任务三: VPS 管理门户 PWA

门户页面在 VPS 上(nginx 根下的 /portal/), 直接改那个 HTML:

1. 下载本仓库 `icons/portal-192.png` / `portal-512.png` 到门户目录
2. head 加 PWA 标签(manifest/sw 引用, theme_color 用 `#6366f1`)
3. 创建门户的 manifest.json(short_name: "VPS门户", start_url 指门户路径)
4. 复制本仓库 `pwa/sw.js` 到门户根路径
5. 门户页面已有 `@media(max-width:640px)` 移动端适配, 不用大改

---

## 验收标准

1. 手机浏览器打开 `/funnel/` `/nextday/` `/portal/` 三个页面, 都能"添加到主屏幕"出现图标
2. 手机(窄屏)打开漏斗和 NextDay, 是卡片流而不是挤在一起的表格, 无横向溢出
3. 桌面(宽屏)打开, 还是原来的表格样式, 功能不变
4. 手动跑一次 `python -m nextday_v2.main`, 微信收到 Server酱 推送
5. cron 配置生效(`crontab -l` 能看到), systemd 三个服务状态 active

## 注意事项

- 数据源走腾讯/新浪公开 API, VPS 在美国也能访问
- pip 用清华镜像 `-i https://pypi.tuna.tsinghua.edu.cn/simple`
- 改 nginx 后必须 `nginx -t` 验证再 reload
- 所有密码/SendKey 不要写进公开仓库

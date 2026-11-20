# Chrome 关闭代理后仍报 ERR_PROXY_CONNECTION_FAILED 的排查记录

## 现象

- 在 Linux（Ubuntu 24.04）上关闭本地代理软件（sing-box，mixed 入站监听 `127.0.0.1:7893`）后，Chrome 打开任何网页都报：

  ```
  未连接到互联网
  ERR_PROXY_CONNECTION_FAILED
  ```

- 同一时间：
  - `curl` 直连正常（`https://www.baidu.com` 200，`https://www.tongyi.com` 301）
  - 系统代理 `gsettings get org.gnome.system.proxy mode` = `'none'`
  - Firefox / 其他程序正常

也就是说：**只有 Chrome 还在坚持使用已经关闭的 7893 代理。**

## 环境

- Ubuntu 24.04 / GNOME
- Google Chrome stable（`/opt/google/chrome`）
- sing-box：mixed 入站 `127.0.0.1:7893`

## 排查过程（走过的弯路）

### 1. 启动器参数

Chrome 的 desktop 文件里三处 `Exec` 都写死了代理：

```
Exec=/usr/bin/google-chrome-stable --proxy-server=http://127.0.0.1:7893 %U
```

去掉后以为解决，但错误依旧。

### 2. 系统代理

```bash
gsettings get org.gnome.system.proxy mode   # none
```

系统层没有代理，排除。

### 3. Chrome 用户配置

`~/.config/google-chrome/Default/Preferences` 中没有 `proxy` 字段；扩展（含 FanVPN）也不在代理状态。排除。

### 4. 找到策略文件

```bash
grep -rn '7893' /etc/opt/chrome/policies/
```

发现：

```json
// /etc/opt/chrome/policies/managed/local-proxy-7893.json
{"ProxySettings":{"ProxyMode":"fixed_servers","ProxyServer":"http://127.0.0.1:7893"}}
```

这是 Chrome 的强制策略（Policy），优先级高于启动参数。

### 5. 关键坑：改名成 `.disabled` 没用

先把文件改名为 `local-proxy-7893.json.disabled`，Chrome 依然连 7893。

**原因：Linux 版 Chrome 会读取 `policies/managed/` 目录下的所有普通文件，不按扩展名过滤。** 改扩展名不等于禁用，必须把文件移出该目录（或删除）。

### 6. 如何确认“到底是谁在连 7893”

在 7893 上起一个临时监听，保持连接，然后用 `sudo ss` 看客户端进程：

```bash
# 临时监听 7893，接受后保持 25 秒
python3 - <<'EOF'
import socket, threading, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('127.0.0.1', 7893))
s.listen(50)
held = []
def hold(c):
    held.append(c); time.sleep(25); c.close()
while True:
    c, _ = s.accept()
    threading.Thread(target=hold, args=(c,), daemon=True).start()
EOF

# 另一个终端：启动/刷新 Chrome 后执行
sudo ss -tnp state established '( dport = :7893 )'
```

输出直接指向 Chrome 的 network service 进程：

```
users:(("chrome",pid=747806,fd=36))
```

这一步是整次排查的转折点：不再猜配置，而是直接抓现行。

## 根因

两个独立来源叠加，且第二层很容易被误判为“已处理”：

| 层 | 位置 | 说明 |
| --- | --- | --- |
| 启动参数 | `~/.local/share/applications/google-chrome.desktop` | `--proxy-server=http://127.0.0.1:7893`，只影响从该启动器启动的实例 |
| 强制策略 | `/etc/opt/chrome/policies/managed/local-proxy-7893.json` | `ProxySettings: fixed_servers`，对所有启动方式生效，优先级更高 |

**最大的坑**：策略文件改成 `.json.disabled` 后仍在生效——Chrome 在 Linux 上加载 managed 目录内的全部文件，不看后缀。

## 正确处置

1. 去掉启动器参数：

```bash
sed -i 's# --proxy-server=http://127\.0\.0\.1:7893##g' \
  ~/.local/share/applications/google-chrome.desktop
```

2. 把策略文件**移出** managed 目录（不要只改名）：

```bash
sudo mkdir -p /etc/opt/chrome/policies-backup
sudo mv /etc/opt/chrome/policies/managed/local-proxy-7893.json \
        /etc/opt/chrome/policies-backup/
```

3. 完全退出 Chrome 再启动（策略只在启动时读取）。

## 验证

- `chrome://policy`：确认 `ProxySettings` 不再出现
- `chrome://net-internals/#proxy`：查看 Effective proxy settings
- 复测：在 7893 上再起监听，启动 Chrome 打开网页，确认没有任何连接打到 7893
- `curl --noproxy '*' https://www.baidu.com` 应正常返回 200

## 经验总结

1. Chrome 的代理来源不止一处：启动参数、**企业策略**、系统代理、扩展。逐层排除时，策略层最容易被忽略。
2. **Linux Chrome 禁用策略不能靠改扩展名**，必须移出 `policies/managed/` 目录。
3. `ERR_PROXY_CONNECTION_FAILED` 出现时，先确认“当前到底有没有代理在生效”，而不是反复重启浏览器。
4. 最有效的定位方式：在目标端口起监听 + `sudo ss -tnp` 抓客户端 PID，直接锁定进程。
5. Chrome 会话恢复的报错标签页不会自动重试，修好配置后仍需手动刷新，容易造成“改了也没用”的错觉。

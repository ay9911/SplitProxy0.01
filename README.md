# SplitProxy Server v1.0.0

## 第一步　准备 VPS、域名和证书

在 Debian 12/13 的 SSH 窗口以 root 操作。下行域名指向这台 VPS。选用独立上行域名时，也让它指向同一台 VPS。建议为 SplitProxy 使用独立子域名。

先安装预检工具：

```bash
apt-get update && apt-get install -y curl iproute2 openssl ca-certificates certbot
```

每个用到的域名都需要同名的 Certbot 证书目录。若已签发，运行 `certbot certificates` 查看证书名称和路径。脚本读取 `/etc/letsencrypt/live/实际域名/fullchain.pem` 和同目录的 `privkey.pem`。

DNS 已指向 VPS、暂不开启 Cloudflare 代理、且 TCP 80 可从公网访问并未被占用时，可用以下命令签发。使用两个域名就执行两遍，各输入对应的真实域名。

```bash
read -rp '请输入指向这台 VPS 的真实域名: ' domain
certbot certonly --standalone --cert-name "$domain" -d "$domain"
```

若已有站点占用 80，或不方便临时关闭 Cloudflare 代理，可通过 DNS 验证签发。使用 `--manual` 且未配置认证钩子的证书需要手动续期。续期后重载 Nginx；启用了直连 TLS 时，还要重启 `codex-split` 以重新读取证书。

## 第二步　运行单文件安装脚本

GitHub Release 发布前，用 FinalShell 把 `install-one-click.sh` 上传到 `/root/`，然后运行：

```bash
bash /root/install-one-click.sh --version
bash /root/install-one-click.sh
```

版本应显示 `SplitProxy Server v1.0.0`。脚本内含安装器、服务端和依赖版本声明，会安装 Debian 软件包及 Python 虚拟环境依赖。

1. 输入下行域名。
2. 选 `ws` 时，上行和下行都经 Cloudflare。上行域名留空则沿用下行域名，填写则配置独立上行域名。
3. 选 `direct` 时，上行走 VPS 的直连 TLS 端口，下行仍经 Cloudflare。选 `both` 时，两种上行方式都启用，可由客户端选择。
4. 安装完成后，将用于 WS 的域名开启 Cloudflare 代理，并将 SSL/TLS 模式设为 **Full (strict)**。VPS 的 443 需允许 Cloudflare 回源。使用直连上行时，还需允许客户端访问所选的直连 TLS 端口。默认内部 WS 端口 18765 仅监听回环地址。

## 第三步　填入 Windows 客户端

运行 `codex-split-menu`，选 `6` 查看参数：

| 服务端参数 | Windows 客户端 |
| --- | --- |
| `SPLIT_DOMAIN` | TLS / Cloudflare 域名 |
| `SPLIT_PATH` | WS 路径 |
| `SPLIT_TOKEN` | 代理 Token |
| `SPLIT_UP_DOMAIN` | 上行域名 |
| `SPLIT_UP_PATH` | 上行 WS 路径，留空使用直连上行 |
| `SPLIT_DIRECT_PORT` | 直连 TLS 端口，为 `0` 时留空 |

下行候选可填 `DNS`。直连上行时，在「VPS 公网 IP」填写 VPS 地址。本地 SOCKS5 端口可保留默认值 `19108`，或换成电脑上未被占用的端口。

## 已装旧版或安装中断

- 脚本会列出检测到的旧配置路径并停止。正在用的服务请先检查 `systemctl status codex-split --no-pager -l`，避免直接删除配置。
- 服务正常、只缺菜单时，运行 `bash /root/install-one-click.sh --menu-only`。
- 若启动失败，运行 `journalctl -u codex-split -n 50 --no-pager` 和 `nginx -t`。日志可用于排查，分享前请遮盖凭据。

## GitHub Release 在线安装

在你控制的 GitHub 仓库创建标签 `v1.0.0` 的 Release，上传本次生成的 `install-one-click.sh`。资产的 SHA-256 为：

```text
e8d885bb705977c13829dec3eea18d56591d707e0c41ff91910d9cb04b440499
```

以下命令用于公开 Release。仓库为私有时，需先通过已登录的 GitHub 账号下载附件，再上传到 VPS。在线下载成功且通过哈希校验后才会执行脚本，安装仍会在 SSH 终端提示填写参数。

```bash
(
  set -e
  installer=$(mktemp)
  trap 'rm -f -- "$installer"' EXIT
  curl -fsSL --retry 3 -o "$installer" 'https://github.com/ay9911/SplitProxy0.01/releases/download/v1.0.0/install-one-click.sh'
  printf '%s  %s\n' 'e8d885bb705977c13829dec3eea18d56591d707e0c41ff91910d9cb04b440499' "$installer" | sha256sum -c -
  bash "$installer"
)
```

公开发布资产前，请确认发布目录里只有要发布的文件。节点 JSON、VPS 地址清单、证书私钥和实际 Token 属于部署数据。

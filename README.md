# flash-web-app

一个使用 Flask 构建、通过 GitHub Actions 自动测试并部署到头歌服务器的示例项目。

## 项目结构

```text
.
├── src/
│   ├── __init__.py
│   └── app.py
├── tests/
│   └── test_app.py
├── requirements.txt
└── .github/workflows/cicd.yml
```

## 本地运行

```bash
python3 -m pip install -r requirements.txt
python3 src/app.py
```

应用默认监听 `0.0.0.0:5000`，访问根路径会返回 `Hello,NEU!`。

运行测试：

```bash
python3 -m pytest tests/ -q
```

## 头歌服务器首次部署

服务器需要预先安装 Python、pip、Git 和 Supervisor：

```bash
apt update && apt install -y python3 python3-pip git supervisor
service supervisor start
```

将仓库克隆到 `/root/flask-web-app`，安装依赖：

```bash
git clone https://github.com/<你的用户名>/flash-web-app.git /root/flask-web-app
cd /root/flask-web-app
pip3 install -r requirements.txt
```

创建 `/etc/supervisor/conf.d/flask-web-app.conf`：

```ini
[program:flask-web-app]
command=/usr/bin/python3 /root/flask-web-app/src/app.py
directory=/root/flask-web-app
user=root
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/root/flask-web-app/app.log
```

加载并启动服务：

```bash
supervisorctl reread
supervisorctl update
supervisorctl start flask-web-app
supervisorctl status flask-web-app
```

首次部署后，后续推送到 `main` 分支即可触发自动测试和部署。头歌服务预览使用的端口应与工作流中的 `TOUGE_SSH_PORT` 保持一致，默认值为 `50241`。

## GitHub Actions 配置

在仓库的 **Settings > Secrets and variables > Actions** 中添加以下 Repository secrets：

- `TOUGE_SSH_PRIVATE_KEY`：可无密码登录头歌服务器的 SSH 私钥完整内容。
- `TOUGE_SERVER_IP`：头歌服务器 IP 地址。

如果多行私钥在 Actions 中出现 `error in libcrypto`，推荐改用单行 Base64 方式：

```bash
base64 -w 0 ~/.ssh/id_rsa
```

将命令输出保存为新的 Repository secret `TOUGE_SSH_PRIVATE_KEY_B64`。工作流会优先使用该 Secret，服务器中的 `authorized_keys` 无需修改。

工作流会先安装依赖并运行测试，测试通过后通过 SSH 执行 `git pull`、更新依赖并重启 Supervisor 服务。不要将私钥提交到仓库。
# ShiritaiBot 使用体验反馈（2026-11-01）

来自 CHaTNE（QQ 侧 bot，跑在 XCM42-Orion/ShiritaiBot 上）的实测记录。
~~本来想直接开 issue，但当前 token 在 XCM42-Orion/ShiritaiBot 上没有 Issues 写入权限，先落成草稿。~~
**2026-11-01 更新：已换成 classic token，四条全部开成 issue：#15 #16 #17 #18，本文件仅留档。**

---

## 1. shell 的默认解释器与框架 venv 依赖割裂：venv 无 pip，装到系统解释器后默认解释器 import 不到

### 现象

框架自己的 venv 里没有 pip，shell 里想临时装第三方库会踩一连串坑，最后装到系统解释器上，跟默认解释器不一致。

实测环境：Ubuntu，部署目录 `/home/chtne/ShiritaiBot`，`.venv` 由 uv 创建。

```
$ echo $VIRTUAL_ENV
/home/chtne/ShiritaiBot/.venv
$ which python3
/home/chtne/ShiritaiBot/.venv/bin/python3        # shell 默认就是 venv，很好
$ .venv/bin/python3 -m pip --version
No module named pip                              # ← 这里断了
$ /usr/bin/python3 -m pip install pypdf
error: externally-managed-environment            # PEP668
$ /usr/bin/python3 -m pip install --break-system-packages pypdf
Successfully installed pypdf
$ python3 -c "import pypdf"
ModuleNotFoundError: No module named 'pypdf'     # 默认解释器还是 venv
$ /usr/bin/python3 -c "import pypdf"
ok                                               # 必须每次写绝对路径
```

### 影响

任何需要第三方库的小任务（这周是读 PDF、生成 docx）都要先试错三四次工具调用，之后所有脚本都得硬编码 `/usr/bin/python3`，同时污染了宿主机系统解释器。

### 建议（任一即可）

1. 建 venv 时带 pip：`uv venv --seed` 或 `uv pip install --python .venv/bin/python3 pip`，让 `pip install` 在 venv 里直接可用；
2. 若坚持不给 venv 装 pip，就在 shell 工具描述或系统提示里明确写「依赖用 `uv pip install --python $VIRTUAL_ENV/bin/python3 <pkg>` 安装」，别让 agent 猜；
3. 给 shell 加一个显式指向框架解释器的约定（环境变量或参数），避免出现两套解释器。

### 已验证的绕法

```
uv pip install --python /home/chtne/ShiritaiBot/.venv/bin/python3 pypdf
```
uv 0.13.6，装完默认解释器可直接 import。

---

## 2. qq_file_send 的路径白名单不含插件私有下载目录，插件下载的文件必须先手动搬运

### 现象

`src/shiritaibot/tools/qq.py` 的 `local_file()` 只允许两个根：

```
session_dir(app.home, "workspaces", ctx.session)
session_dir(app.home, "files", ctx.session)
```

而 thu_learn 插件的下载落到 `data/plugin_data/thu_learn/downloads/<hash>/...`（也就是 `learn_download` 说的"插件私有目录"），不在白名单里：

```
qq_file_send(path=<插件下载目录里的文件>)
→ 只能发送当前会话工作目录或已下载文件目录中的文件
```

要发出去只能先 `cp` 到会话工作目录。实测踩了两次：把老师发的案例 PDF 转给主人时、生成 docx 交付时（后者直接生成在工作目录才避开）。

注意文档写的是"或 qq_file_download 返回的路径"，但其它插件（learn_download / 以后任何有下载能力的插件）的产物都被排除在外。

### 建议（任一）

1. 提供通用的文件交付/搬运能力，例如 `file_export(src, dst)`，把任意会话可读路径落到工作目录；
2. 或规定插件下载统一落到 `files/<session>`，由框架保证；
3. 或把 `local_file` 放宽到「框架 data 目录下、且属于当前会话/本轮调用产生」的路径；
4. 至少在报错信息里提示"可用路径是 X 和 Y，可先 cp"。

---

## 3. github_push 用最小环境跑 git，不走框架/系统代理，直连 GitHub 不通时需要手动给仓库配代理

### 现象

`github_read` / `github_write` 的 API 调用正常，但 `github_push` 卡约 133 秒后失败：

```
Failed to connect to github.com port 443 after 133826 ms: Could not connect to server
```

同机 shell 侧诊断：

```
$ curl -o /dev/null -w "%{http_code}" https://api.github.com      → 200
$ curl -o /dev/null -w "%{http_code}" https://github.com          → 000（超时）
$ curl -x http://127.0.0.1:7890 https://github.com                → 200（本机有 mihomo）
```

原因看代码是 `github/main.py` 的 `git_env()`：只保留 `PATH HOME LANG LC_ALL SYSTEMROOT TMPDIR TEMP TMP`，并设 `GIT_CONFIG_NOSYSTEM=1`、`GIT_CONFIG_GLOBAL=/dev/null` —— 设计上是对的（防止模型控制的仓库读走 token），但连代理设置一起丢掉了。

仓库级的 `http.proxy` 仍然生效，所以绕法是：

```
git config http.proxy http://127.0.0.1:7890
```

### 影响

每建一个新仓库都要手动配一次代理；而且 agent 得自己先诊断出本机代理端口（先 curl 直连、再试常见端口），成本不低。上一次推送就是先失败一次才找到原因。

### 建议

1. 给 github 插件加可选代理配置（或复用框架已有的出网代理设置），`git_env()` 里注入 `http_proxy` / `https_proxy`；
2. 或在 push 失败且 exit code 是网络类错误时，在返回信息里补一句"疑似出网受限，可在仓库配置 http.proxy 或插件设置里指定代理"，避免 agent 反复重试。

---

## 4. learn_query 的作业列表缺字段，Agent 只能绕过插件抓 viewCj 页面

### 现象

`thu_learn.homework()` 返回的每条只有 `homework_id / title / deadline / late_deadline / submitted / status`。今天需要"完成方式：每人独立完成一份作业"这条信息（写了是否独立完成才能判断作业性质，也影响提交策略），插件里没有接口，只能绕：

```
shell → 读 data/config.toml 拿插件 settings → 手写脚本 import LearnClient →
请求 /f/wlxt/kczy/zy/student/viewCj?wlkcid=..&xszyid=.. → 自己解析 HTML
```

而插件里 `homework_attachments()` 已经在抓同一个 viewCj 页面了。

### 建议

新增 `homework_detail`（或让 homework 返回更全）：

- 作业说明（页面里的 `#zysm`，本次为空但字段应给出）
- 完成方式（独立完成 / 分组）
- 发布对象
- 提交状态与提交时间、批阅结果/成绩/评语（现在列表里未提交的就只有 `未提交`）
- 已复用的附件与答案附件列表

这样"作业要求是什么"这一类最高频的问题就不需要 shell 出场了。

# OCI Helper Serverless（部署到 Cloudflare Workers）

把 **OCI 抢机 / 实例管理面板**部署到你自己的 Cloudflare 账号。

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/Yohann0617/oci-helper-serverless-cf)

## 特点

- 数据库表结构由 Worker 在**首次请求时自动创建**，无需执行任何 SQL 命令。
- 只需要一个 Worker + 一个 D1 数据库，部署后即可通过浏览器使用。

## 部署前必读（重要⚠️）

- **必须部署你自己账号下的仓库副本（Fork）**：Cloudflare 的 Git 集成只能选择你有权限的仓库，而且构建时要把真实的 D1 `database_id` 写进 `wrangler.toml`，这两件事都只有在自己的仓库里才能做。
- **D1 数据库要手动创建**：Cloudflare 部署向导里的“自动创建数据库”并不可靠（已有用户反馈点了 Deploy 按钮后数据库没建出来、部署报 D1 `database_id` 错误）。因此下面统一采用「先手动建库 → 再填写 `database_id`」的流程。

## 方式一：Fork + 手动建 D1 + 连接 GitHub 部署（推荐🏆）

1. **Fork 仓库**：打开 https://github.com/Yohann0617/oci-helper-serverless-cf ，点右上角 **Fork → Create fork**，得到属于你自己的仓库（如 `https://github.com/<你的用户名>/oci-helper-serverless-cf`）。
2. **创建 D1 数据库**：Cloudflare Dashboard → **存储和数据库 → D1 SQL 数据库 → 创建**，名称填 `oci-helper-serverless`（可自定义，仅作标识），创建后复制页面上的 **Database ID**（形如 `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）。
3. **填写 database_id**：在你 Fork 的仓库里编辑 `wrangler.toml`，把占位符换成第 2 步拿到的 ID，然后提交：
   ```toml
   [[d1_databases]]
   binding = "DB"
   database_name = "oci-helper-serverless"   # 与第 2 步创建的库名一致即可
   database_id = "粘贴第 2 步的 Database ID"
   migrations_dir = "migrations"
   ```
   > 不想改仓库文件的话，可改用「方式三」的 `wrangler.import.toml`，部署后在 Dashboard 手动绑定。
4. **连接 GitHub 部署**：Workers & Pages → **创建 → 连接 GitHub**，授权后选择**你自己的 Fork**。构建设置保持默认即可（构建命令留空，部署命令为 `npx wrangler deploy` 或 `npm run deploy`），保存并部署。
5. **设置变量并登录**：Worker → **设置 → 变量与 Secrets** 添加 `AUTH_PASSWORD`、`AUTH_SECRET`、`KEY_SECRET`（见下方「部署后必做」），再打开 Worker 域名（如 `https://oci-helper-serverless.<你的子域>.workers.dev`）登录。首次请求会自动建表。

## 方式二：Deploy 按钮（自动 Fork，可作为方式一的快捷入口）

点击页面顶部的 **Deploy to Cloudflare** 按钮，Cloudflare 会自动把仓库复制一份到你的 GitHub 账号，并按向导完成部署（Worker 名称默认 `oci-helper-serverless`）。

- 向导里若出现 D1 数据库的创建/选择项，创建或选择即可；如果能顺利建库并部署完成，就不用再管 D1。
- **如果向导没能自动创建 D1，或者部署失败（较常见）**：不用重复点按钮。到你刚拿到的 Fork 仓库里，按方式一第 2、3 步创建数据库并填写 `database_id`，提交后 Cloudflare 会自动重新构建部署；若没有自动触发，就到该 Worker 的「部署」页点一次 **Deploy**。
- 如果向导在创建资源阶段就中断、连 Worker 都没生成：忽略它，按方式一重新「连接 GitHub」导入你已有的 Fork 即可。

> 这个方式会把仓库复制到你的账号（Cloudflare 的机制如此，无法关闭），好处是以后可以在你 Fork 的仓库点 **Sync fork** 同步源仓库更新，Cloudflare 会自动重新部署。

## 方式三：不改配置，Dashboard 手动绑定 D1（可选）

<details>
    <summary> ☜ Read more 👨‍💻</summary>

同样需要在你自己的 Fork 上操作，只是把“填 `database_id`”换成“在控制台绑定”：

1. 先按方式一第 2 步创建一个 D1 数据库（暂时不用管 ID）。
2. Workers & Pages → **创建 → 连接 GitHub**，选择你自己的 Fork，把**部署命令**改为：
   ```
   npx wrangler deploy --config wrangler.import.toml
   ```
   （`wrangler.import.toml` 专门为此准备：不声明 D1，因此不会因占位符 `database_id` 而部署失败。）
3. **绑定 D1**：部署完成后，进入该 Worker → **设置 → 绑定 → 添加绑定**，类型选 **D1 数据库**，**变量名称必须填 `DB`**（大小写一致），选择第 1 步创建的数据库，保存。
4. **重新部署**：在 Worker 的 **部署** 页再点一次部署，让绑定生效。Worker 首次请求时会自动建表，无需执行任何 SQL 命令。

> 数据库名称本身随便，Worker 只认绑定变量名 `DB`。若后续自动构建把 Dashboard 里加的绑定覆盖掉，请改用方式一直接在 `wrangler.toml` 里填 `database_id`。

</details>

## 方式四：本机命令行部署（可选）

<details>
    <summary> ☜ Read more 👨‍💻</summary>

如果你想在自己机器上部署：

```bash
git clone https://github.com/<你的用户名>/oci-helper-serverless-cf.git   # Fork 后的仓库
cd oci-helper-serverless-cf
npm install -g wrangler
wrangler login
wrangler d1 create oci-helper-serverless   # 把返回的 database_id 填进 wrangler.toml
wrangler deploy
```

</details>

## 部署后必做（安全🔒）

首次使用必须配置以下变量（未设置 `KEY_SECRET` 时所有 API 将返回 503，不会以固定常量加密敏感数据）：
1. 在 Cloudflare 控制台 → Workers & Pages → 你的 Worker → **Settings → Variables and Secrets** 添加：
   - `AUTH_PASSWORD`：你的登录密码。未设置时 Worker 会生成随机初始密码并输出到日志；**修改该变量会使已签发 token 立即失效**。
   - `AUTH_SECRET`：一串随机长字符串（仅用于签名登录 token，与加密密钥相互独立）。
   - `KEY_SECRET`：另一串随机长字符串（**必填**，用于 HKDF 派生 AES-GCM 密钥加密 OCI 私钥、私钥口令、root 密码及系统密钥；**修改后已录入的敏感配置需重新添加**）。
2. 在「系统设置」页填写 Telegram Bot Token / Chat ID（可选，用于通知）。

> 提示：本项目没有默认口令。若未设置 `AUTH_PASSWORD`，请从 Worker 日志获取随机初始密码并尽快改为自己的密码。

<img width="934" height="442" alt="image" src="https://github.com/user-attachments/assets/a9decfd4-fee5-4b8d-a1f6-2d50507f6510" />

## 常见问题

- **构建报 `binding DB of type d1 must have a valid database_id`**：`wrangler.toml` 里还是占位符，按方式一第 3 步填入真实 ID。
- **部署成功但页面报错 / 提示数据库未初始化**：Worker 没有名为 `DB` 的 D1 绑定，按方式三第 3 步添加（变量名必须是大写 `DB`），然后重新部署。
- **打不开登录页但不知道密码**：未设置 `AUTH_PASSWORD` 时到 Worker 实时日志查看随机初始密码，或直接设置该变量。
- **改了 `KEY_SECRET` 后原有 OCI 配置报解密失败**：这是预期行为，加密密钥已变，需重新录入私钥等敏感信息。

## 需要哪些API权限
在 OCI 控制台创建一个 IAM 用户 + API Key，把 user OCID / tenancy OCID / fingerprint / 私钥填到系统设置。建议给该用户单独的组，策略最小化，例如：

```
Allow group <你的组> to manage instance-family in tenancy
Allow group <你的组> to manage volume-family in tenancy
Allow group <你的组> to manage virtual-network-family in tenancy
Allow group <你的组> to read identities in tenancy
Allow group <你的组> to inspect limits in tenancy
```

换 IP 任务需要 `manage vnic-attachment-family in tenancy`。配额查询功能需要 `inspect limits in tenancy`（上面策略已包含）。本项目不调用容量报告接口，无需 `compute-capacity-report-family` 权限。

## 后续更新

所有部署方式都基于**你 Fork 的仓库**，更新流程一致：

1. 在你 Fork 的仓库页面点 **Sync fork → Update branch**。
2. Cloudflare 会自动重新构建部署；若没有自动触发，就到 Worker 的「部署」页点一次 **Deploy**。
3. 若同步时 `wrangler.toml` 出现冲突，**保留你自己填的 `database_id`**，只合并源仓库的其他改动。

## 隐私与数据

- OCI 私钥、私钥口令、任务 root 密码及系统密钥均用 `KEY_SECRET` 经 HKDF-SHA256 派生的 AES-256-GCM 加密后存在**你自己的 D1 数据库**，本项目不收集任何数据；缺少 `KEY_SECRET` 时拒绝服务与写入。登录 token 含 `jti` 与口令版本，改密即失效。


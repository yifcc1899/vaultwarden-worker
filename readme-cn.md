简易部署教程
该项目的部署方法与二叉树树的教程一致: 你可曾想过，直接将 BitWarden 部署到 Cloudflare Worker？
因此我在此只做简述并强调关键步骤，详细图文流程请参考上面的博客，记得要 fork 我的项目就行。

前期准备
你需要准备：

一个 Cloudflare 账户
一个 Github 账户
一个没有被阻断的域名
第一步：设置 Github 项目
首先 fork 我的仓库到你的账户（要是能顺手点个星星就好啦），进入 Action 页面，启用 Action。
点击 settings - Secrets and variables - Actions，准备添加 Repository secrets，接下来我们一共要添加三个 secrets。

然后进入 Cloudflare 控制台，https://dash.cloudflare.com/
在浏览器地址栏中复制你的 account id
image

回到 github 页面，点击 New repository secret，名称写 CLOUDFLARE_ACCOUNT_ID，值就写刚才复制的 id。

接着访问 https://dash.cloudflare.com/profile/api-tokens
选择编辑 Worker 的模板，然后给它再添加 D1 的编辑权限，再创建 api 令牌，回到 github 页面添加为 CLOUDFLARE_API_TOKEN​。

最后去 Cloudflare 控制台的存储与数据库 - D1 sql 数据库，创建一个数据库，并复制它的 id。

image
image
635×124 4.57 KB

回到 github，添加为 D1_DATABASE_ID。
至此在 github 上的配置就完毕了，点击上方的 Actions 选项卡，选到 Build 工作流点运行即可。

第二步：建表
在 Action 运行的时候也不用干等着，可以先去 CF 控制台里把表建了。

打开刚才创建的数据库，点 Explore Data，然后在 Query 窗口中分别粘贴并运行以下三条 sql 语句 (定义在 sql/schema.sql 中)：

CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY NOT NULL,
    name TEXT,
    email TEXT NOT NULL UNIQUE,
    email_verified BOOLEAN NOT NULL DEFAULT 0,
    master_password_hash TEXT NOT NULL,
    master_password_hint TEXT,
    password_salt TEXT, -- Salt for server-side PBKDF2 hashing (NULL for legacy users pending migration)
    key TEXT NOT NULL, -- The encrypted symmetric key
    private_key TEXT NOT NULL, -- encrypted asymmetric private_key
    public_key TEXT NOT NULL, -- asymmetric public_key
    kdf_type INTEGER NOT NULL DEFAULT 0, -- 0 for PBKDF2
    kdf_iterations INTEGER NOT NULL DEFAULT 600000,
    security_stamp TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS ciphers (
    id TEXT PRIMARY KEY NOT NULL,
    user_id TEXT,
    organization_id TEXT,
    type INTEGER NOT NULL,
    data TEXT NOT NULL, -- JSON blob of all encrypted fields (name, notes, login, etc.)
    favorite BOOLEAN NOT NULL DEFAULT 0,
    folder_id TEXT,
    deleted_at TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (folder_id) REFERENCES folders(id) ON DELETE SET NULL
);
CREATE TABLE IF NOT EXISTS folders (
    id TEXT PRIMARY KEY NOT NULL,
    user_id TEXT NOT NULL,
    name TEXT NOT NULL, -- Encrypted folder name
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
第三步：配置环境变量与自定义域名
当 github 的 Action 执行完毕后，你就能在 CF 的 Workers 中找到刚才创建的 warden-worker 了，进入这个 Worker 的设置页面，在变量和机密栏添加以下三个密钥：

名称	说明
ALLOWED_EMAILS	允许注册的完整邮箱，用，分隔（例：your-name@example.com）
JWT_SECRET	随机长字符串
JWT_REFRESH_SECRET	随机长字符串
可选环境变量

IMPORT_BATCH_SIZE：用于控制导入密码库时，每次批操作的数据条数，默认为 30，设为 0 代表一次批操作导入所有数据（不推荐）；如果你需要导入的库特别大，可以适当调大这个值。
TRASH_AUTO_DELETE_DAYS: 配置回收站中的项目多少天后会被清理，默认为 30，设为 0 代表永不清理。
DISABLE_USER_REGISTRATION: 用于控制是否表明服务器不支持注册，默认为 true，设为 false 可以让客户端显示注册按钮；该选项不影响实际注册功能。
然后在域和路由中添加一个路由，区域选择你的域名，路由写你分给这个 Worker 的子域名 + /*（如 bitwarden.example.com/*）。

最后别忘记去给这个子域名添加一条 dns 记录。
如果你想优选 IP，那就不开小黄云，把它 CNAME 到一个优选过的域名。
要是不想优选，那就打开小黄云，目标随便填。

建议去域名配置页的 安全性 - 安全规则 里给 /identity/* 和 /api/accounts/* 添加速率限制规则。
第四步：创建账户并导入数据
这步其实没啥好说的了，如果已有 bitwarden 库，在电脑上把它导出为 JSON 即可。

由于没有前端，只能在手机 APP 上创建账户，然后在电脑上导入即可。 现在可以了

注意: 创建账户时用的密码必须记住，没有任何方式能找回。

以下开始为我自己加的可选步骤，建议整一下。

第五步：配置数据库同步 S3
我这里是整了一个自动导出 D1 并上传到 S3 的 Action，虽然也能用私有仓库备份，但有点怕判定为滥用，所以还是创一个私有的 S3 吧。

注意：

千万别像我一样傻 fufu 的用跟 Worker 同一个 Cloudflare 账户的 R2 来备份，不然你号一被封就全没了；
千万别用登录或者获取密钥需要依赖 Bitwarden 的账户，你得自己记着，不然就套娃了；它的恢复邮箱的密码你也得记着。
如果没什么特殊需求，可以试试 backblaze，虽然额度非常少但用来备份也够了，胜在完全免费不绑卡。
此处略过注册账户和创建桶的操作。

请确保你之前添加的 CLOUDFLARE_API_TOKEN 至少有 D1 的读取权限。
回到你 fork 的 github 仓库，继续添加以下 secrets:

Secret	Required	Description
S3_ACCESS_KEY_ID	yes	S3 access key ID
S3_SECRET_ACCESS_KEY	yes	S3 secret access key
S3_BUCKET	yes	桶名称
S3_REGION	yes	S3 区域，有就正常填，不知道就直接填 auto
S3_ENDPOINT	no	用 AWS 就不用填，其他填 https://your-s3-domain.com 的形式，带协议不带路径
BACKUP_ENCRYPTION_KEY	no	额外加密密钥，不填就不加密，填了就一定要记住
然后在 Action 页面中选到 Backup D1 Database to S3，手动触发一次，等待它运行完成，然后检查你的 S3 中是否有备份文件。成功后，每天都会自动备份一次，每个备份默认保存 30 天。

至于恢复操作还是请各位到时去看 readme 吧，希望这辈子不会用到。

额外说明
其实 Cloudflare D1 提供了回滚到 30 天内的任意时刻的功能：Cloudflare D1 Time Travel documentation

# 查看当前bookmark
# DATABASE_NAME换成你在Cloudflare控制台里设置的D1名称
wrangler d1 time-travel info DATABASE_NAME

# 回退到指定时间 (ISO 8601)
wrangler d1 time-travel restore DATABASE_NAME --timestamp=2024-01-15T12:00:00Z
    
# 回退到指定bookmark
wrangler d1 time-travel restore DATABASE_NAME --bookmark=<bookmark_id>
通过这个操作，就算库出了问题也能快速回滚，再配上 S3 备份，是不是感觉这一通操作不那么灵车了？

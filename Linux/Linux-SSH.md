# SSH 与 SCP 使用笔记

## 一、SSH 远程登录

### 1.1 基本用法

```bash
ssh user@hostname
```

- **user**：用户名
- **hostname**：IP 地址或域名

首次登录时会提示主机指纹验证：

```
The authenticity of host '123.57.47.211 (123.57.47.211)' can't be established.
ECDSA key fingerprint is SHA256:iy237yysfCe013/l+kpDGfEG9xxHxm0dnxnAbJTPpG8.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入 `yes` 并回车即可。该服务器的信息会被记录在 `~/.ssh/known_hosts` 文件中，之后再次连接时不会再提示。

随后输入密码即可登录到远程服务器。

### 1.2 指定端口

SSH 默认登录端口号为 22。如需登录特定端口：

```bash
ssh user@hostname -p 2222
```

### 1.3 直接执行命令

无需进入交互式 shell，可直接在远程服务器上执行命令：

```bash
# 基本格式
ssh user@hostname command

# 示例：查看远程目录
ssh user@hostname ls -a

# 示例：在远程执行循环（单引号中的 $i 会在远程求值）
ssh myserver 'for ((i = 0; i < 10; i++)); do echo $i; done'

# 注意：双引号中的 $i 会在本地先求值，结果为 0
ssh myserver "for ((i = 0; i < 10; i++)); do echo $i; done"
```

> **提示**：单引号 `'...'` 中的变量在远程服务器上求值；双引号 `"..."` 中的变量在本地先求值后再发送到远程，实际使用中需注意引号选择。


## 二、SSH 配置文件

### 2.1 基本配置

创建 `~/.ssh/config` 文件（若不存在）：

```bash
touch ~/.ssh/config
chmod 600 ~/.ssh/config
```

在文件中输入：

```ssh-config
Host myserver1
    HostName 192.168.1.100
    User root

Host myserver2
    HostName example.com
    User admin
    Port 2222
```

之后可以直接使用别名连接：

```bash
ssh myserver1
ssh myserver2
```

### 2.2 常用配置项

| 配置项                  | 说明                         |
| ----------------------- | ---------------------------- |
| `HostName`              | 实际的主机名或 IP 地址       |
| `User`                  | 登录用户名                   |
| `Port`                  | SSH 端口号                   |
| `IdentityFile`          | 私钥文件路径                 |
| `ProxyJump`             | 跳板机配置                   |
| `ServerAliveInterval`   | 发送保活探测的时间间隔（秒） |
| `ServerAliveCountMax`   | 保活探测失败的最大次数       |
| `ForwardAgent`          | 是否转发 SSH 代理            |
| `StrictHostKeyChecking` | 主机密钥检查策略             |

### 2.3 通配符与全局配置

`Host` 块支持通配符匹配，按文件中出现的顺序匹配，第一个匹配的块生效：

```ssh-config
# 全局配置（放在文件末尾，作为默认值）
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    TCPKeepAlive yes

# 匹配所有以 dev- 开头的主机
Host dev-*
    User developer
    IdentityFile ~/.ssh/dev_key

# 匹配特定域名
Host *.example.com
    User admin
    Port 2222
```

> **重要规则**：由于每个参数使用首次获得的值，具体的主机配置应放在文件靠前位置，通用默认配置放在最后。

### 2.4 保持长连接

在 `Host` 块中添加以下配置，防止 SSH 连接因空闲超时断开：

```ssh-config
Host myserver
    HostName 192.168.1.100
    User root
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

- **ServerAliveInterval 60**：每 60 秒向服务器发送一次保活消息
- **ServerAliveCountMax 3**：连续 3 次无响应后断开连接


## 三、密钥登录

### 3.1 生成密钥

```bash
# 推荐方式：使用 Ed25519（现代系统首选，安全性和性能最佳）
ssh-keygen -t ed25519

# 兼容旧系统：使用 RSA 4096 位
ssh-keygen -t rsa -b 4096
```

执行后一直回车即可。生成的密钥文件位于 `~/.ssh/` 目录下：

| 文件                            | 说明               | 权限 |
| ------------------------------- | ------------------ | ---- |
| `id_ed25519` / `id_rsa`         | 私钥（绝不能泄露） | 600  |
| `id_ed25519.pub` / `id_rsa.pub` | 公钥（可安全分发） | 644  |

> **密钥类型选择建议**：优先使用 Ed25519，它在安全性、性能和密钥长度方面都优于 RSA，仅当需要兼容旧系统时才使用 RSA 3072/4096 位。

### 3.2 上传公钥到服务器

**方式一：使用 ssh-copy-id（推荐）**

```bash
ssh-copy-id myserver
```

该命令会自动将本地公钥追加到服务器的 `~/.ssh/authorized_keys` 文件中，并自动设置正确的权限。

指定端口或密钥文件：

```bash
ssh-copy-id -p 2222 -i ~/.ssh/id_ed25519.pub user@hostname
```

**方式二：手动复制**

```bash
cat ~/.ssh/id_ed25519.pub | ssh user@hostname \
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

**方式三：直接粘贴**

将公钥文件内容复制，粘贴到服务器的 `~/.ssh/authorized_keys` 文件中。

### 3.3 ⚠️ 密钥登录权限要求

SSH 对文件权限有严格的安全要求，权限不正确会导致认证失败。

服务器端（authorized_keys 所在机器）:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R $(whoami):$(whoami) ~/.ssh
```

客户端（私钥所在机器）:

```bash
chmod 600 ~/.ssh/id_ed25519        # 私钥必须为 600
chmod 700 ~/.ssh                    # .ssh 目录必须为 700
```

> **关键点**：`~/.ssh` 目录权限为 700，`authorized_keys` 文件权限为 600，私钥文件权限为 600。如果 `~/.ssh` 或 `authorized_keys` 是组可写或其他人可写的，SSH 会拒绝使用密钥认证。


## 四、SCP 文件传输

### 4.1 基本用法

```bash
scp source destination
```

### 4.2 本地 → 远程

```bash
# 复制单个文件
scp ~/file.txt myserver:/home/user/

# 一次复制多个文件
scp file1.txt file2.txt myserver:/home/user/

# 复制文件夹（-r 递归）
scp -r ~/tmp myserver:/home/acs/

# 使用相对路径（相对于远程用户的家目录）
scp -r ~/tmp myserver:homework/
```

### 4.3 远程 → 本地

```bash
# 复制远程文件夹到本地当前路径
scp -r myserver:homework .

# 复制远程文件到本地指定目录
scp myserver:/var/log/app.log ~/logs/
```

### 4.4 远程 → 远程

```bash
# 两个远程主机之间的复制（默认通过本地中转）
scp user1@host1:/path/file user2@host2:/path/
```

### 4.5 指定端口号

```bash
scp -P 2222 source destination
```

> **注意**：`scp` 指定端口的参数是大写 `-P`，而 `ssh` 是小写 `-p`。

### 4.6 常用参数

| 参数               | 说明                                  |
| ------------------ | ------------------------------------- |
| `-r`               | 递归复制整个目录                      |
| `-P port`          | 指定远程端口号                        |
| `-p`               | 保留原文件的修改时间、访问时间和权限  |
| `-C`               | 启用压缩传输                          |
| `-l limit`         | 限制带宽，单位为 Kbit/s               |
| `-i identity_file` | 指定私钥文件                          |
| `-v`               | 详细模式，输出调试信息                |
| `-q`               | 静默模式，不显示进度条                |
| `-B`               | 批处理模式，避免重复询问密码          |
| `-o ssh_option`    | 传递 SSH 选项，格式同 `~/.ssh/config` |

参数可以组合使用，例如：

```bash
# 限速 10Mbit/s、启用压缩、保留文件属性、递归复制目录
scp -r -C -p -l 10000 -P 2222 ~/data myserver:/backup/
```

> **提示**：`scp` 的 `-r`、`-P` 等参数建议加在 source 和 destination 之前。

### 4.7 SCP 底层原理

scp 底层通过 SSH 协议传输数据，提供与 SSH 相同的安全保护。自 OpenSSH 9.0 起，scp 默认在内部使用 SFTP 协议进行传输，但命令行接口保持向后兼容。

与 SFTP 的主要区别：

| 特性     | SCP          | SFTP                               |
| -------- | ------------ | ---------------------------------- |
| 交互模式 | 无，仅命令行 | 有，类似 FTP                       |
| 目录列表 | 不支持       | 支持                               |
| 断点续传 | 不支持       | 支持                               |
| 文件操作 | 仅复制       | 上传、下载、重命名、删除等完整操作 |

SCP 适合快速一次性复制；需要复杂文件管理操作时建议使用 SFTP。


## 五、使用 SCP 同步配置文件

将本地的 vim 和 tmux 配置同步到服务器：

```bash
scp ~/.vimrc ~/.tmux.conf myserver:
```

`myserver:` 后不指定路径时，默认复制到远程用户的家目录（`~`）。


## 六、常见问题排查

### 6.1 Permission denied (publickey)

这是最常见的 SSH 密钥认证失败错误。排查顺序如下：

**① 检查公钥是否已添加到服务器**

```bash
# 在服务器上查看已授权的公钥列表
cat ~/.ssh/authorized_keys
```

**② 检查文件权限（最常见原因）**

```bash
# 服务器端
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# 客户端
chmod 600 ~/.ssh/id_ed25519
```

**③ 检查服务器 SSH 配置**

确认 `/etc/ssh/sshd_config` 中以下配置项已启用：

```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

修改后重启 SSH 服务：

```bash
sudo systemctl restart sshd
```

**④ 指定正确的私钥文件**

如果本地有多个密钥对，连接时需用 `-i` 指定：

```bash
ssh -i ~/.ssh/id_ed25519 user@hostname
```

**⑤ RSA 算法兼容性问题**

OpenSSH 8.8+ 默认禁用了 `ssh-rsa` 签名算法。如果服务器只支持 RSA，可在 `~/.ssh/config` 中添加：

```ssh-config
Host *
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
```

### 6.2 REMOTE HOST IDENTIFICATION HAS CHANGED!

当服务器重建、重装系统或更换了主机密钥后，本地 `known_hosts` 中记录的旧指纹与新指纹不匹配，会触发此警告。

解决方法：

```bash
# 删除该主机在 known_hosts 中的旧记录
ssh-keygen -R hostname
# 或指定 IP
ssh-keygen -R 123.57.47.211
```

删除后重新连接，会提示确认新的主机指纹，确认无误后输入 `yes` 即可。

> **安全提示**：删除前请先与服务器管理员确认新指纹的真实性，排除中间人攻击的可能性。


## 七、SSH 代理与跳板机

### 7.1 ProxyJump 跳板机配置

当目标服务器在内网中，需要通过跳板机（堡垒机）访问时，使用 `ProxyJump`：

```ssh-config
Host jumpbox
    HostName jumphost.example.com
    User jumpuser

Host internal
    HostName 192.168.1.100
    User admin
    ProxyJump jumpbox
```

配置后直接连接：

```bash
ssh internal
```

SSH 客户端会自动先连接到跳板机，再从跳板机转发连接到目标服务器。

### 7.2 命令行临时使用

```bash
# 使用 -J 参数
ssh -J jumphost.example.com internal.example.com

# 使用 -o 参数
ssh -o 'ProxyJump=jumphost.example.com' internal.example.com

# 多个跳板机链式指定
ssh -J user1@hop1,user2@hop2 user3@dest
```

`ProxyJump` 不需要服务器端额外配置，只需要能 SSH 访问跳板机即可。
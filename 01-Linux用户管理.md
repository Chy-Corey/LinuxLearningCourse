## 用户管理

> Create Date：2026-04-03
>
> Update Date: 2026-04-03

与 Windows 不同，登录 Windows 系统时通常只需输入密码即可——这是因为一台运行 Windows（尤其是家用版）的计算机通常默认只供单个用户使用，系统设计也围绕这一场景展开。

而 Linux 则是一个**多用户、多任务的分时操作系统**，其核心设计理念从一开始就面向**多用户共享同一台计算机**的场景（如早期的服务器、工作站等）。在 Linux 中，多个用户可以同时通过本地终端、SSH等方式登录系统，并各自独立地运行程序、访问文件，彼此之间互不干扰。

正是由于 Linux 的多用户特性，系统必须具备**严格、精细的用户权限管理机制**。因此，Linux 从内核层面就引入了完善的权限控制体系——包括**用户（User）、用户组（Group）和权限位（Read/Write/Execute）的三元模型**，以及基于文件所有者、所属组和其他用户的访问控制规则。

因此，理解用户管理是掌握 Linux 基础操作的关键。

---

### 一、Linux 用户的类型以及用户管理

用户类型包括：

1. **root 用户（超级管理员）**
   - UID 为 0，拥有对系统的**完全控制权**
   - 可以修改任何文件、安装软件、管理服务等
   - **不建议日常使用 root**，容易误操作导致系统损坏
2. **普通用户**
   - 由管理员创建，用于日常操作
   - 默认只能访问自己的home目录（如 `/home/yubao`）和部分公共目录
   - 可通过 `sudo` 临时获得 root 权限（需授权）

管理员可以创建、删除、修改普通用户，命令方式如下：

| 命令                    | 说明                                                 |
| ----------------------- | ---------------------------------------------------- |
| `useradd yubao`         | 创建用户 `yubao`（不创建家目录）                     |
| `adduser yubao`         | 交互式创建用户（自动创建家目录、设置密码等，更友好） |
| `passwd yubao`          | 为用户 `yubao` 设置或修改密码                        |
| `usermod -aG dev yubao` | 将 `yubao` 添加到 `dev` 组（`-aG` 表示追加）         |
| `userdel yubao`         | 删除用户（加 `-r` 同时删除家目录）                   |

以上及以下的命令不需要刻意去背，在开发过程中记忆即可，命令有非常多的选项符，可参考链接[Linux 用户和用户组管理 | 菜鸟教程](https://www.runoob.com/linux/linux-user-manage.html)。

### 二、用户与用户组的关系

- 每个用户必须属于**至少一个主组**（Primary Group），默认组名与用户名相同
- 用户可以同时属于**多个附加组**（Supplementary Groups）
- 权限可按**用户**或**组**分配，便于批量管理

> 🌰 举例：将开发人员 `yubao` 和 `bob` 加入 `dev` 组，然后赋予 `dev` 组对 `/project` 目录的读写权限，两人即可协作。

| 命令           | 说明                      |
| -------------- | ------------------------- |
| `groupadd dev` | 创建 `dev` 用户组         |
| `groupdel dev` | 删除 `dev` 用户组         |
| `groups yubao` | 查看 `yubao` 所属的所有组 |

### 三、核心配置文件

| 文件           | 作用                                                     |
| -------------- | -------------------------------------------------------- |
| `/etc/passwd`  | 存储用户基本信息（用户名、UID、GID、家目录、默认 Shell） |
| `/etc/shadow`  | 存储加密后的密码（仅 root 可读）                         |
| `/etc/group`   | 存储用户组信息（组名、GID、成员列表）                    |
| `/etc/sudoers` | 配置哪些用户或组可以使用 `sudo`（用 `visudo` 编辑）      |

### 四、文件/目录权限

每个文件/目录有三类权限对象：
- **所有者（User）**
- **所属组（Group）**
- **其他人（Others）**

每类有三种权限：
- **r（读）**：查看文件内容 / 列出目录
- **w（写）**：修改文件 / 在目录中增删文件
- **x（执行）**：运行程序 / 进入目录

可通过 `ls -l` 查看权限，例如：
```bash
-rw-r--r-- 1 yubao dev 1024 Apr 4 10:00 file.txt
```
表示：`yubao`（所有者）可读写，`dev` 组成员只读，其他人只读。

关于文件/目录权限的修改，在后续文件管理学习详细介绍。

### 六、举例

用户/用户组不仅可以配置对于文件/目录的权限，还可以配置执行命令的权限。

当然可以！下面是一个**完整、实用且安全的 `sudo` 权限分配示例**，适用于日常运维或开发环境。

#### 🎯 场景设定

你有一台 Linux 服务器，有以下用户：

- **`devuser`**：一名普通开发人员
- **需求**：允许他**重启 Nginx 服务**和**查看系统日志**，但**不能执行其他特权命令**（比如不能关机、不能改系统配置）。

#### ✅ 目标

通过 `visudo` 给 `devuser` 分配**最小必要权限**（遵循“最小权限原则”）。

#### 🔧 操作步骤

##### 第 1 步：用 `visudo` 安全编辑 sudoers 文件

```bash
sudo visudo
```

##### 第 2 步：在文件中添加以下规则

找到文件中类似 `# User privilege specification` 的部分，在下方添加：

```text
# 授权 devuser 只能执行特定命令
devuser ALL = /bin/systemctl restart nginx, /bin/systemctl status nginx, /usr/bin/journalctl -u nginx
```

规则解释：

| 部分           | 含义                               |
| -------------- | ---------------------------------- |
| `devuser`      | 用户名                             |
| `ALL`          | 适用于所有主机（在单机上就是本机） |
| `=` 后面的内容 | 允许执行的**具体命令路径**         |

> ⚠️ **必须使用命令的绝对路径！**
> 用 `which systemctl` 和 `which journalctl` 确认路径（通常是 `/bin/systemctl` 和 `/usr/bin/journalctl`）。

##### 第 3 步（可选）：让命令无需输入密码

如果希望 `devuser` 执行这些命令时**不用输自己的密码**（适合自动化脚本），改成：

```text
devuser ALL = NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl status nginx, /usr/bin/journalctl -u nginx
```

> 🔒 注意：`NOPASSWD` 会降低安全性，仅在可信环境中使用。

##### 第 4 步：保存并退出

- 如果用 `vi`：按 `Esc` → 输入 `:wq` 回车
- `visudo` 会自动检查语法，如果出错会提示，**不要强行覆盖**

#### ✅ 验证效果

切换到 `devuser` 用户：

```bash
su - devuser
```

尝试以下命令：

```bash
# 应该成功（可能需要输 devuser 自己的密码）
sudo systemctl restart nginx

# 应该成功
sudo journalctl -u nginx

# 应该被拒绝！
sudo reboot
sudo cat /etc/shadow
```

你会看到类似这样的拒绝信息：

```text
Sorry, user devuser is not allowed to execute '/sbin/reboot' as root...
```

✅ 权限控制生效！

#### 🛡 安全建议

1. **永远用绝对路径**：防止 PATH 劫持攻击

2. **避免 `ALL` 权限**：除非是真正的管理员

3. 优先用组授权

   （更易管理）：

   ```text
   %webadmins ALL = /bin/systemctl restart nginx
   ```

   然后把用户加到 

   ```
   webadmins
   ```

    组：

   ```
   sudo usermod -aG webadmins devuser
   ```

#### 📌 总结

通过 `visudo`，你可以精确控制：

> “**哪个用户，在哪台机器上，能以谁的身份，运行哪些命令**”

这个例子展示了如何**安全地赋予开发人员有限的运维能力**，既提升效率，又守住安全底线。

#### **蓝图 v2.0：AzerothCore 多大区自动化部署 (增强版)**

**最高目标 (Overall Goal)**:
> 实现对 AzerothCore 项目在远程服务器上的全自动化、可重复、一键式多大区部署，并确保部署过程严格遵循官方 `README.md` 的所有规范及历史经验。

**核心思想与设计原则 (Core Ideas & Design Principles)**:
> 1.  **配置即代码 (Configuration as Code)**: 所有部署参数都将通过唯一的 Ansible 变量文件进行管理。
> 2.  **幂等性 (Idempotency)**: Playbook 将被设计为完全幂等，反复执行不会产生副作用。
> 3.  **模块化与可扩展性 (Modularity & Scalability)**: 通过 Ansible Roles 对部署逻辑进行解耦，实现优雅地增删大区。
> 4.  **健壮性与自愈性 (Robustness & Self-Healing)**: 内置前置条件检查和错误规避机制，吸取过往部署经验。

---

#### **第一阶段：核心设计与规范 (Core Design & Specifications)**
> 1.  **命名规范 (Naming Conventions)**:
>     -   Ansible 角色: `setup`, `config`, `deploy`, `post-deploy`
>     -   Docker 服务/容器: `ac-database`, `ac-authserver`, `s<ID>-db-import`, `s<ID>-worldserver`
>     -   数据库: `acore_auth`, `s<ID>_world`, `s<ID>_characters`
>     -   配置文件: `s<ID>_worldserver.conf`
> 2.  **服务启动依赖 (Service Dependencies)**: (在 `docker-compose.yml` 中强制执行)
>     -   `ac-authserver` -> `s1-db-import` (成功完成)
>     -   `sX-worldserver` -> `sX-db-import` (成功完成)
>     -   所有 `db-import` -> `ac-database` (健康)
> 3.  **内联模板 (Inlined Templates)**: 为规避潜在的文件创建限制 (如 `.j2` 后缀)，所有模板内容将使用 `ansible.builtin.copy` 的 `content` 参数直接嵌入到 Ansible 任务中。

---

#### **第二阶段：项目结构规划 (Project Structure Planning)**
> `ansible` 目录将从零开始创建，用于存放所有自动化代码。
```
/data/azerothcore/      <-- 部署根目录 (由 Git Clone 创建)
├── ansible/
│   ├── deploy.yml      # 主 Playbook
│   ├── inventory.yml   # 主机清单
│   ├── group_vars/
│   │   └── all.yml     # 全局变量 (密码, 大区列表等)
│   └── roles/
│       ├── setup/
│       │   └── tasks/
│       │       └── main.yml  # Git 克隆, 环境准备
│       ├── config/
│       │   └── tasks/
│       │       └── main.yml  # 生成所有配置文件
│       ├── deploy/
│       │   └── tasks/
│       │       └── main.yml  # 部署并启动 Docker Compose 服务
│       └── post-deploy/
│           └── tasks/
│               └── main.yml  # 注册大区到数据库
├── ... (其他项目文件)
```
---

#### **第三阶段：核心配置与变量 (Core Configuration & Variables)**
> `ansible/group_vars/all.yml` 将作为单一数据源。
```yaml
# ansible/group_vars/all.yml

# --- General Settings ---
project_root: "/data/azerothcore"
project_repo: "git@github.com:duckdream/demo.git"
project_branch: "main"

# --- Database Settings ---
db_root_password: "a_very_secret_password"
db_external_port: 3306

# --- Auth Server Settings ---
auth_external_port: 3724

# --- Realms Definition ---
# 通过修改此列表来增删大区
realms:
  - id: 1
    name: "Ares"
    address: "10.10.0.146" # 需替换为您的服务器IP
    world_port: 8085
    soap_port: 7878
  - id: 2
    name: "Zeus"
    address: "10.10.0.146" # 需替换为您的服务器IP
    world_port: 8086
    soap_port: 7879

# --- Client Data Source ---
client_data_url: "http://10.10.0.109/downloads/data.zip"
client_data_version_string: "INSTALLED_VERSION=v16"
```
---

#### **第四阶段：核心任务实现规划 (Core Task Implementation Plan)**
>
> #### **角色 1: `setup` (环境初始化)**
> *   **任务 1.1**: 使用 `ansible.builtin.git` 克隆仓库。
> *   **任务 1.2**: 使用 `ansible.builtin.stat` 检查 `data-version` 文件，实现客户端数据准备的幂等性。
> *   **任务 1.3**: (条件任务) 若 `data-version` 不存在，则下载、解压、清理并创建版本文件。
>
> #### **角色 2: `config` (配置生成)**
> *   **任务 2.1**: 使用内联模板生成 `.env` 文件。
> *   **任务 2.2**: 循环遍历 `realms` 变量，使用内联模板为每个大区生成 `s<ID>_worldserver.conf`。模板将基于原始 `worldserver.conf`，仅修改 `RealmID` 和数据库连接信息，严格遵循 `README`。
>
> #### **角色 3: `deploy` (服务部署)**
> *   **任务 3.1**: 使用内联模板和 `realms` 变量生成 `docker-compose.yml` 文件，动态构建所有服务。
> *   **任务 3.2**: 使用 `community.docker.docker_compose_v2` 启动服务。
>
> #### **角色 4: `post-deploy` (部署后任务)**
> *   **任务 4.1**: **(修正案)** 在 `localhost` 上使用 `ansible.builtin.wait_for` 检查数据库端口，并明确设置 `become: false`，以避免本地 `sudo` 权限问题。
> *   **任务 4.2**: **(修正案)** 循环 `realms` 变量，使用 `community.mysql.mysql_query` 注册大区。
>     -   **(修正案)** 任务参数将使用正确的 `login_db` 代替 `database`。
>     -   **(修正案)** 任务将委派到 `localhost` 执行，并设置 `become: false`。
>     -   **前置条件**: Playbook 执行前，需确保 `localhost` 已安装 `PyMySQL` 库 (`pip install PyMySQL`)。此要求将被明确记录。

---

#### **第五阶段：执行与验证 (Execution & Verification)**
> **前置条件**:
> ```bash
> # 在您的本地机器（Ansible控制节点）上执行
> pip install PyMySQL ansible
> ```
>
> **执行命令**:
> ```bash
> # 进入ansible目录
> cd /path/to/project/ansible
> # 执行部署
> ansible-playbook -i inventory.yml deploy.yml
> ```
> **验证步骤**:
> 1.  检查 Docker 容器状态 (`docker ps`)。
> 2.  检查新大区的配置文件 (`cat /data/azerothcore/env/dist/etc/sX_worldserver.conf`)。
> 3.  连接数据库查询 `acore_auth.realmlist` 表，确认新大区已注册。
> 4.  启动游戏客户端，确认能看到所有大区。 
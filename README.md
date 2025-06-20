## AzerothCore项目 Docker 部署指南

### `!!!!部署第一步：获取代码!!!!`

> [!IMPORTANT]
> 本项目的所有自动化部署都基于一个完整的 Git 仓库结构。
>
> **禁止手动创建任何项目目录。**
>
> 您在目标服务器上需要执行的第一个，也是唯一一个初始化动作，就是完整克隆本仓库。

克隆项目仓库并进入目录：
**`!!!!建议使用SSH协议访问仓库!!!!`**
```bash
git clone -b main git@github.com:duckdream/demo.git /data/azerothcore
cd /data/azerothcore
```

---

### 运行时

- Docker Engine 20.10+
- Docker Compose 2.0+

### 目录结构与核心概念

`!!!!重要!!!!` **为确保所有操作的正确性，请务必理解以下核心概念：**

1.  **单一代码库 (Single Source of Truth)**
    本项目所有大区的配置与管理，均源自**同一个Git仓库副本**。您只需要克隆一次仓库，就可以部署和管理一个或多个大区。

2.  **项目根目录 (Project Root)**
    本文档中提到的所有文件路径（如 `env/dist/etc/`）和执行的命令，都**默认以您克隆下来的Git仓库根目录作为当前工作目录**。

3.  **部署目标目录**
    文档中提到的 `/data/azerothcore` 是部署在服务器上的**目标路径**，也就是您执行 `git clone` 命令时指定的目录。

下方展示的目录结构，正是您克隆仓库后在**项目根目录**下所看到的结构：

```
/data/azerothcore/
├── ac-client-data/     # 客户端数据文件
├── conf/
│   └── dist/
│       └── env.ac      # 环境配置文件
└── env/
    └── dist/
        ├── etc/        # 配置文件模版
        └── logs/       # 日志文件
```

### 准备环境与数据

`!!!!重要!!!!` **请在克隆完成后的项目根目录 (`/data/azerothcore`) 下执行以下所有命令。**

#### 1. 准备客户端数据
**`!!!!若 data-version 文件已存在，则跳过此步!!!!`**
```bash
wget http://10.10.0.109/downloads/data.zip
unzip -o -q data.zip -d ac-client-data
rm data.zip
echo 'INSTALLED_VERSION=v16' > data-version
```

#### 2. 配置环境变量 (`.env` 文件)
为了方便管理所有可配置的变量（如数据库密码、端口等），`docker-compose` 会自动加载项目根目录下的 `.env` 文件。

这是一个推荐的模板，您可以直接复制到项目根目录并按需修改：
```env
# 数据库Root密码
DOCKER_DB_ROOT_PASSWORD=password

# 数据库对外暴露的端口
DOCKER_DB_EXTERNAL_PORT=3306

# 认证服务器对外暴露的端口
DOCKER_AUTH_EXTERNAL_PORT=3724

# 环境变量定义文件路径 (通常无需修改)
DOCKER_AC_ENV_FILE=./conf/dist/env.ac

# 数据、配置、日志卷的路径 (通常无需修改)
DOCKER_VOL_ETC=./env/dist/etc
DOCKER_VOL_LOGS=./env/dist/logs
DOCKER_VOL_DATA=./ac-client-data
```

### 服务部署与配置

#### 启动顺序与依赖关系

服务启动顺序由依赖关系决定，具体如下：

1.  **ac-database**
    -   首先独立启动。

2.  **s1-db-import**
    -   大区1的数据库导入服务，是引导所有服务的基础。
    -   **关键职责**: 初始化共享的 `acore_auth` 库，以及大区1的 `s1_world` 和 `s1_characters` 库。
    -   依赖于 `ac-database` 服务的健康状态。

3.  **ac-authserver** 和 **s1-worldserver**
    -   `ac-authserver` (共享认证服) 依赖于 `s1-db-import` 服务的成功完成。
    -   `s1-worldserver` (大区1世界服) 依赖于 `s1-db-import` 服务的成功完成。

4.  **其他大区的服务 (sX-db-import, sX-worldserver for X > 1)**
    -   每个额外大区的 `sX-db-import` 服务依赖于 `ac-database` 的健康状态。
    -   每个 `sX-worldserver` 依赖于其对应的 `sX-db-import` 服务成功完成。

#### 核心配置文件

`etc/` 目录 (`/data/azerothcore/env/dist/etc`) 存放着所有服务的核心配置文件。部署的基本原则是：**`!!!!除了`worldserver.conf`需要为每个大区定制修改外，其余所有配置文件均保持默认，无需任何改动!!!!`**

-   **`sX_worldserver.conf`**: 每个大区专属的世界服务器配置文件。在多大区部署中，您需要为每个大区创建一个该文件的副本（例如 `s1_worldserver.conf`, `s2_worldserver.conf`），并修改其中的 `RealmID` 和数据库信息。

-   **`authserver.conf`**: 认证服务器的配置文件。此文件已包含合理的默认值，且所有大区共享同一个认证服务，因此**禁止修改**此文件。

-   **`dbimport.conf`**: 数据库导入工具的配置文件。此文件同样保持默认即可，其连接信息是通过环境变量在容器启动时动态传递的。

-   **`*.dist` 文件**: 这些是分发的示例配置文件，作为原始参考，不直接参与服务运行。

#### 服务详解

##### 数据库服务 (ac-database)

**镜像**: mysql:8.4

**端口映射**:
- `${DOCKER_DB_EXTERNAL_PORT:-3306}:3306` (默认: 3306)

**环境变量**:
- `MYSQL_ROOT_PASSWORD`: 数据库root密码 (默认: password)

**挂载卷**:
- `ac-database:/var/lib/mysql`: 数据库文件持久化存储

**健康检查**:
- 命令: `/usr/bin/mysql --user=root --password=$MYSQL_ROOT_PASSWORD --execute "SHOW DATABASES;"`
- 间隔: 5秒
- 超时: 10秒
- 重试次数: 40次

**重启策略**: unless-stopped (除非手动停止，否则总是重启)

##### 数据库导入服务 (sX-db-import)

**镜像**: registry.cn-shanghai.aliyuncs.com/demo-sh/ac-wotlk-db-import:master

**环境变量**:
- `AC_DATA_DIR`: 客户端数据目录 (值: "/azerothcore/env/dist/data")
- `AC_LOGS_DIR`: 日志目录 (值: "/azerothcore/env/dist/logs")
- `AC_LOGIN_DATABASE_INFO`: 认证数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_auth")
- `AC_WORLD_DATABASE_INFO`: **每个大区专属**的世界数据库连接信息 (例如: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};s1_world")
- `AC_CHARACTER_DATABASE_INFO`: **每个大区专属**的角色数据库连接信息 (例如: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};s1_characters")

**挂载目录**:
- `${DOCKER_VOL_ETC:-./env/dist/etc}:/azerothcore/env/dist/etc`: 配置文件目录
- `${DOCKER_VOL_LOGS:-./env/dist/logs}:/azerothcore/env/dist/logs:delegated`: 日志文件目录

**依赖条件**:
- 依赖于 ac-database 服务的健康状态 (condition: service_healthy)

##### 认证服务器 (ac-authserver)

**镜像**: registry.cn-shanghai.aliyuncs.com/demo-sh/ac-wotlk-authserver:master

**端口映射**:
- `${DOCKER_AUTH_EXTERNAL_PORT:-3724}:3724` (默认: 3724)

**环境变量文件**:
- `${DOCKER_AC_ENV_FILE:-./conf/dist/env.ac}` (默认: ./conf/dist/env.ac)

**环境变量**:
- `AC_LOGS_DIR`: 日志目录 (值: "/azerothcore/env/dist/logs")
- `AC_TEMP_DIR`: 临时文件目录 (值: "/azerothcore/env/dist/temp")
- `AC_LOGIN_DATABASE_INFO`: 认证数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_auth")

**挂载目录**:
- `${DOCKER_VOL_ETC:-./env/dist/etc}:/azerothcore/env/dist/etc`: 配置文件目录
- `${DOCKER_VOL_LOGS:-./env/dist/logs}:/azerothcore/env/dist/logs:delegated`: 日志文件目录

**依赖条件**:
- 依赖于 ac-database 服务的健康状态 (condition: service_healthy)
- 依赖于 **s1-db-import** 服务的成功完成 (condition: service_completed_successfully)

**重启策略**: unless-stopped (除非手动停止，否则总是重启)

##### 世界服务器 (sX-worldserver)

**镜像**: registry.cn-shanghai.aliyuncs.com/demo-sh/ac-wotlk-worldserver:master

**终端和标准输入**: 需要打开(tty和stdin_open设置为true)，**!!!!否则会启动异常!!!!**

**端口映射**:
- `!!!!必须为每个大区的世界服务分配唯一的外部端口!!!!`
- `${DOCKER_WORLD_EXTERNAL_PORT_X:-8085}:8085` (例如: 8085)
- `${DOCKER_SOAP_EXTERNAL_PORT_X:-7878}:7878` (例如: 7878)

**环境变量文件**:
- `${DOCKER_AC_ENV_FILE:-./conf/dist/env.ac}` (默认: ./conf/dist/env.ac)

**环境变量**:
- `AC_DATA_DIR`: 客户端数据目录 (值: "/azerothcore/env/dist/data")
- `AC_LOGS_DIR`: 日志目录 (值: "/azerothcore/env/dist/logs")

**挂载目录和文件**:
`!!!!注意!!!!` 每个 `sX-worldserver` 的所有数据库连接信息**必须通过**挂载下方为其专属创建的 `sX_worldserver.conf` 文件进行配置。
- `${DOCKER_VOL_ETC:-./env/dist/etc}:/azerothcore/env/dist/etc`: 配置文件目录
- `./env/dist/etc/sX_worldserver.conf:/azerothcore/env/dist/etc/worldserver.conf`: **每个大区专属**的世界服配置文件
- `${DOCKER_VOL_LOGS:-./env/dist/logs}:/azerothcore/env/dist/logs:delegated`: 日志文件目录
- `${DOCKER_VOL_DATA:-./ac-client-data}:/azerothcore/env/dist/data/:ro`: 客户端数据目录 (只读)

**依赖条件**:
- 依赖于 ac-database 服务的健康状态 (condition: service_healthy)
- 依赖于其对应的 **sX-db-import** 服务的成功完成 (condition: service_completed_successfully)

**重启策略**: unless-stopped (除非手动停止，否则总是重启)

### 部署与管理

#### 新增大区流程

`ac-authserver` 为所有大区共享。新增一个大区意味着需要为其创建一套专属的 `sX-db-import` 和 `sX-worldserver` 服务，并更新数据库。

以下流程以新增一个**ID为2**的大区为例。

##### 第一步：创建并配置 s2_worldserver.conf

`!!!!重要!!!!` **此步骤必须严格遵循，它是确保新大区被正确识别的关键。**

1.  **强制性动作：复制模板文件**
    `必须`将项目中的原始配置文件 `env/dist/etc/worldserver.conf` 复制一份。`禁止`从零创建。

2.  **重命名**
    根据命名规范 `s<大区ID>_worldserver.conf`，将副本重命名为 `s2_worldserver.conf`。

3.  **修改内容**
    打开 `s2_worldserver.conf` 文件，**仅修改**以下三行：
    -   `RealmID = 2` (修改为新大区的ID)
    -   `WorldDatabaseInfo = "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};s2_world"` (按 `s<大区ID>_world` 格式修改数据库名)
    -   `CharacterDatabaseInfo = "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};s2_characters"` (按 `s<大区ID>_characters` 格式修改数据库名)
    
    `!!!!注意：除以上三行外，其余所有参数均需保持默认值不变!!!!`

##### 第二步：定义新大区的服务

在您的 `docker-compose.yml` 或等效的部署工具中，为新大区添加服务定义。

1.  **数据库导入服务 (`s2-db-import`)**
    -   该服务依赖 `ac-database`，并应配置为使用新大区的数据库名 (`s2_world`, `s2_characters`)。

2.  **世界服务器 (`s2-worldserver`)**
    -   **端口映射**: `必须`为该服务分配**唯一**的外部端口，避免冲突。例如：`8086:8085`, `7879:7878`。
    -   **配置文件挂载**: 将上一步创建的 `s2_worldserver.conf` 文件挂载到容器内的 `/azerothcore/env/dist/etc/worldserver.conf`。
    -   该服务依赖 `ac-database` 和 `s2-db-import`。

##### 第三步：更新大区列表

连接到共享的认证数据库 (`acore_auth`)，执行以下SQL语句将新大区注册到列表中。

```sql
-- 示例：为ID为2的大区"Realms2"添加记录
-- IP地址说明：
--   - 如果您仅在本地或局域网内玩，请填写服务器的局域网IP。
--   - 如果您希望从公网访问，请填写服务器的公网IP地址或域名。
--   - 如果客户端和服务端在同一台机器上，可以直接使用 127.0.0.1。
-- 端口说明：
--   需将端口 `8086` 替换为您在第二步中为 `s2-worldserver` 分配的世界服务器外部端口。
INSERT INTO acore_auth.realmlist (id, name, address, port, flag, timezone) 
VALUES (2, 'Realms2', '服务器IP', 8086, 0, 1)
ON DUPLICATE KEY UPDATE name=VALUES(name), address=VALUES(address), port=VALUES(port), flag=VALUES(flag), timezone=VALUES(timezone);
```

#### 创建游戏账号
```bash
# 连接到任意一个大区的世界服务器控制台，例如大区1
docker attach s1-worldserver

# 创建账号
account create 用户名 密码

# 设置账号权限（可选，3为GM权限）
account set gmlevel 用户名 3 -1

# 退出控制台（按 Ctrl+P 然后 Ctrl+Q）
```

### 附录：多大区最终目录结构示例

当您按照以上流程创建了多个大区（例如，大区1和大区2）后，您的 `/data/azerothcore/env/dist/etc` 目录看起来应该像这样：

```
/data/azerothcore/env/dist/etc/
├── authserver.conf
├── authserver.conf.dist
├── dbimport.conf
├── dbimport.conf.dist
├── modules/
├── s1_worldserver.conf  <- 大区1的配置文件
├── s2_worldserver.conf  <- 大区2的配置文件
├── worldserver.conf     <- 原始模板文件
└── worldserver.conf.dist
```

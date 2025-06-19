## AzerothCore项目 Docker 部署指南

### 运行时

- Docker Engine 20.10+
- Docker Compose 2.0+

### 目录结构

基础目录为 `/data/azerothcore`，完整目录结构如下：

```
/data/azerothcore/
├── ac-client-data/     # 客户端数据文件
├── conf/
│   └── dist/
│       └── env.ac      # 环境配置文件
├── env/
│   └── dist/
│       ├── etc/        # 配置文件模版
│       └── logs/       # 日志文件
└── docker-compose.yml  # Docker Compose配置文件
```

1. 克隆项目仓库并进入目录：
**`!!!!使用SSH协议访问仓库!!!!`**
```bash
git clone -b main git@github.com:duckdream/demo.git /data/azerothcore
```

2. 准备客户端数据：
**`!!!!若data-version存在则不必下载!!!!`**
```bash
# 下载客户端数据文件
cd /data/azerothcore
wget http://10.10.0.109/downloads/data.zip
unzip -o -q data.zip -d ac-client-data
rm data.zip
echo 'INSTALLED_VERSION=v16' > data-version
```

### 服务

1. **数据库服务 (ac-database)**
   - 基于MySQL 8.4
   - 存储游戏所有数据

2. **数据库导入服务 (ac-db-import)**
   - 初始化数据库结构和数据
   - 一次性服务，完成后自动退出

3. **认证服务器 (ac-authserver)**
   - 处理玩家账号验证
   - 管理大区列表

4. **世界服务器 (ac-worldserver)**
   - 处理游戏世界逻辑
   - 管理玩家角色和游戏内容

### 启动顺序与依赖关系

服务启动顺序由依赖关系决定，具体如下：

1. **ac-database**
   - 首先启动数据库服务
   - 等待数据库健康检查通过

2. **ac-db-import**
   - 在数据库服务健康后启动
   - 导入所有必要的数据库结构和数据
   - 完成后自动退出

3. **ac-authserver** 和 **ac-worldserver**
   - 在数据库服务健康且数据库导入服务成功完成后启动
   - 两者可以并行启动，没有相互依赖

### 配置
项目仓库中除了`worldserver.conf`需要修改以外，**`!!!!其余所有配置文件保持默认无需修改!!!!`**。

#### 数据库服务 (ac-database)

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

#### 数据库导入服务 (ac-db-import)

**镜像**: registry.cn-shanghai.aliyuncs.com/demo-sh/ac-wotlk-db-import:master

**环境变量**:
- `AC_DATA_DIR`: 客户端数据目录 (值: "/azerothcore/env/dist/data")
- `AC_LOGS_DIR`: 日志目录 (值: "/azerothcore/env/dist/logs")
- `AC_LOGIN_DATABASE_INFO`: 认证数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_auth")
- `AC_WORLD_DATABASE_INFO`: 世界数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_world")
- `AC_CHARACTER_DATABASE_INFO`: 角色数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_characters")

**挂载目录**:
- `${DOCKER_VOL_ETC:-./env/dist/etc}:/azerothcore/env/dist/etc`: 配置文件目录
- `${DOCKER_VOL_LOGS:-./env/dist/logs}:/azerothcore/env/dist/logs:delegated`: 日志文件目录

**依赖条件**:
- 依赖于 ac-database 服务的健康状态 (condition: service_healthy)

### 认证服务器 (ac-authserver)

**镜像**: registry.cn-shanghai.aliyuncs.com/demo-sh/ac-wotlk-authserver:master

**端口映射**:
- `${DOCKER_AUTH_EXTERNAL_PORT:-3724}:3724` (默认: 3724)

**环境变量文件**:
- `${DOCKER_AC_ENV_FILE:-conf/dist/env.ac}` (默认: conf/dist/env.ac)

**环境变量**:
- `AC_LOGS_DIR`: 日志目录 (值: "/azerothcore/env/dist/logs")
- `AC_TEMP_DIR`: 临时文件目录 (值: "/azerothcore/env/dist/temp")
- `AC_LOGIN_DATABASE_INFO`: 认证数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_auth")

**挂载目录**:
- `${DOCKER_VOL_ETC:-./env/dist/etc}:/azerothcore/env/dist/etc`: 配置文件目录
- `${DOCKER_VOL_LOGS:-./env/dist/logs}:/azerothcore/env/dist/logs:delegated`: 日志文件目录

**依赖条件**:
- 依赖于 ac-database 服务的健康状态 (condition: service_healthy)
- 依赖于 ac-db-import 服务的成功完成 (condition: service_completed_successfully)

**重启策略**: unless-stopped (除非手动停止，否则总是重启)

### 世界服务器 (ac-worldserver)

**镜像**: registry.cn-shanghai.aliyuncs.com/demo-sh/ac-wotlk-worldserver:master

**终端和标准输入**: 需要打开(tty和stdin_open设置为true)，**!!!!否则会启动异常!!!!**

**端口映射**:
- `${DOCKER_WORLD_EXTERNAL_PORT:-8085}:8085` (默认: 8085): 世界服务器端口
- `${DOCKER_SOAP_EXTERNAL_PORT:-7878}:7878` (默认: 7878): SOAP接口端口

**环境变量文件**:
- `${DOCKER_AC_ENV_FILE:-./conf/dist/env.ac}` (默认: ./conf/dist/env.ac)

**环境变量**:
- `AC_DATA_DIR`: 客户端数据目录 (值: "/azerothcore/env/dist/data")
- `AC_LOGS_DIR`: 日志目录 (值: "/azerothcore/env/dist/logs")
- `AC_LOGIN_DATABASE_INFO`: 认证数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_auth")
- `AC_WORLD_DATABASE_INFO`: 世界数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_world")
- `AC_CHARACTER_DATABASE_INFO`: 角色数据库连接信息 (值: "ac-database;3306;root;${DOCKER_DB_ROOT_PASSWORD:-password};acore_characters")

**挂载目录和文件**:
- `${DOCKER_VOL_ETC:-./env/dist/etc}:/azerothcore/env/dist/etc`: 配置文件目录
- `./env/dist/etc/worldserver.conf:/azerothcore/env/dist/etc`: 世界服配置文件
- `${DOCKER_VOL_LOGS:-./env/dist/logs}:/azerothcore/env/dist/logs:delegated`: 日志文件目录
- `${DOCKER_VOL_DATA:-./ac-client-data}:/azerothcore/env/dist/data/:ro`: 客户端数据目录 (只读)

**依赖条件**:
- 依赖于 ac-database 服务的健康状态 (condition: service_healthy)
- 依赖于 ac-db-import 服务的成功完成 (condition: service_completed_successfully)

**重启策略**: unless-stopped (除非手动停止，否则总是重启)

### 新增大区步骤
1个区 = 公共服务ac-authserver + 1个ac-db-import + 1个ac-worldserver

**命名规范**：
- **大区ID**: 整数
- **新区名**: "Realms大区ID"
- **服务名**: "s大区ID-服务"
- **数据库**: "s大区ID_数据库"
- **配置文件**: "s大区ID_worldserver.conf"

**服务**:
- **ac-authserver**: 多区共用
- **ac-db-import**: 每区各1个，名称不可重复
- **ac-worldserver**: 每区各1个，名称不可重复，内部端口不变，外部端口不可重复

**配置**:
- **配置文件**: 使用模版文件!!!`/data/azerothcore/env/dist/etc/worldserver.conf`，每区个1个，名称不可重复，配置文件中的RealmID值与各区ID对应，挂载到容器内`/azerothcore/env/dist/etc/worldserver.conf`!!!
- **数据库**: acore_world和acore_characters，每区各1个，名称不可重复
- **更新大区列表**: 新区状态flag设置为0，timezone设置为1
```sql
INSERT INTO acore_auth.realmlist (id, name, address, port, flag, timezone) 
VALUES (大区ID, 'Realms大区ID', '服务器IP', 8085, 0, 1);
INSERT INTO acore_auth.realmlist (id, name, address, port, flag, timezone) 
VALUES (大区ID, 'Realms大区ID', '服务器IP', 8086, 0, 1);
```
**目录结构**:
!!!!多区共用项目仓库!!!!：`git clone -b main git@github.com:duckdream/demo.git /data/azerothcore`
```
/data/azerothcore
├── README.md
├── ac-client-data
│   └── data-version
├── conf
│   └── dist
│       ├── env.ac
│       └── env.docker
└── env
    ├── dist
    │   ├── etc
    │   │   ├── authserver.conf
    │   │   ├── authserver.conf.dist
    │   │   ├── dbimport.conf
    │   │   ├── dbimport.conf.dist
    │   │   ├── modules
    │   │   │   └── mod_eluna.conf.dist
    │   │   ├── s1_worldserver.conf  ## 大区1世界服务配置文件
    │   │   ├── s2_worldserver.conf  ## 大区2世界服务置文件
    │   │   ├── worldserver.conf
    │   │   └── worldserver.conf.dist
    │   └── logs
    └── user
```

### 创建游戏账号
```bash
# 连接到任意世界服务器控制台
docker attach ac-worldserver

# 创建账号
account create 用户名 密码

# 设置账号权限（可选，3为GM权限）
account set gmlevel 用户名 3 -1

# 退出控制台（按 Ctrl+P 然后 Ctrl+Q）
```

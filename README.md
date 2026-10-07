# Novel System · 小说推荐与评论系统

基于 **Spring Boot 3 + Vue 3** 的前后端分离平台，支持**读者阅读、作者创作、管理员运营**三种角色的完整业务闭环。

> 一句话说明：这是一个能跑起来的小说平台 —— 读者能看书、作者能发书、管理员能管书，并且带一套**混合推荐算法**帮读者发现想读的小说。

![Java](https://img.shields.io/badge/Java-17-blue)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.6-brightgreen)
![Vue](https://img.shields.io/badge/Vue-3.5-42b883)
![MySQL](https://img.shields.io/badge/MySQL-8-orange)
![Redis](https://img.shields.io/badge/Redis-cache-red)

---

## 功能概览

### 读者端
- 小说浏览、分类筛选、关键词搜索与高级搜索（按分类 / 作者 / 字数 / 状态多条件组合 + 排序）
- 在线阅读器、阅读进度自动记录与续读
- 书架收藏、点赞、评论与章节点评
- **个性化推荐**：基于用户行为的混合推荐算法（详见下文）
- 阅读社区：话题讨论与回复

### 作者端
- 作品管理、章节增删改查、草稿箱
- 作品数据统计（阅读量、收藏量趋势，ECharts 可视化）

### 管理员端
- 用户管理、作品审核、分类与标签管理
- 平台数据看板、评论与讨论治理
- 小说批量导入、封面爬取工具

---

## 技术栈

| 层次 | 技术选型 |
|---|---|
| 后端框架 | Spring Boot 3.2.6、Java 17 |
| 数据访问 | Spring Data JPA / Hibernate |
| 数据库 | MySQL 8 |
| 缓存 | Redis（Lettuce 连接池） |
| 安全认证 | Spring Security + JWT（jjwt 0.12.5） |
| 参数校验 | Jakarta Validation（`@Valid`） |
| 前端 | Vue 3.5、Vite 7、Vue Router、Pinia、Element Plus、ECharts、Axios |
| 代码规范 | ESLint + Prettier |
| 构建工具 | Maven（后端）、npm（前端） |

---

## 系统架构

```
+-----------------------------------------------------------+
|  前端（Vue 3 + Vite）                                       |
|  |- 读者端  :8081                                          |
|  |- 作者端  :8082                                          |
|  '- 管理端  :8083                                          |
+---------------------------+-------------------------------+
                            | HTTP / JSON（Axios，统一封装于 src/api）
+---------------------------v-------------------------------+
|  后端（Spring Boot，context-path = /api）                   |
|                                                            |
|  Controller 层（16 个，共 145 个接口）                       |
|    Auth / User / Novel / Reading / Comment / Bookmark /    |
|    BookList / Discussion / Category / Search /             |
|    Recommendation / Author / Admin / NovelImport /         |
|    NovelCoverCrawler / Proxy                               |
|                          |                                 |
|  Service 层（15 个）                                        |
|    NovelService  ReadingService  SearchService             |
|    RecommendationService  RedisCacheService                |
|    NovelImportService  UserBehaviorService  ...            |
|                          |                                 |
|  Repository 层（17 个 Spring Data JPA 接口）                |
|                          |                                 |
|  统一响应体 ApiResponse<T> + GlobalExceptionHandler         |
+-------------+-----------------------------+---------------+
              |                             |
      +-------v-------+             +-------v-------+
      |   MySQL 8     |             |    Redis      |
      |   18 张表      |             |  推荐/热点缓存 | 
      +---------------+             +---------------+
```

**接口规模**：16 个 Controller、**145 个 REST 接口**
**数据模型**：18 个 JPA 实体（User、Novel、Chapter、Comment、Bookmark、ReadingProgress、UserBehavior、Topic、Discussion、Review、NovelStats、Category、BookList 等）

---

## 核心亮点：混合推荐算法

`RecommendationService` 是本项目最核心的模块，实现 **协同过滤 + 基于内容** 的混合推荐，而非简单热度排序。

### 1. 用户行为权重模型

不同行为对兴趣画像的贡献不同：

| 行为 | 权重 | 设计理由 |
|---|---|---|
| 收藏 BOOKMARK | 5.0 | 最强的兴趣信号，用户主动留存 |
| 评论 COMMENT | 4.0 | 高投入行为，代表深度兴趣 |
| 点赞 LIKE | 3.0 | 轻度正向反馈 |
| 浏览 VIEW | 1.0 | 噪声最大，权重最低 |
| 阅读时长 READ_TIME | 0.1 / 10 秒 | 连续量，按时间累积 |

### 2. 推荐链路

```
用户行为日志 --> 兴趣画像构建
                    |
        +-----------+-----------+
        v                       v
   协同过滤                   基于内容
 （找收藏行为相似的用户，      （统计用户在「分类」和「标签」
   推荐他们收藏过、              上的偏好分布，匹配同偏好小说）
   当前用户未读的小说）              |
        +-----------+-----------+
                    v
      归一化热门度加权融合（阅读量 0.6 + 收藏量 0.4，映射到 0-1）
                    v
        Redis 缓存（缓存键包含结果数量参数）
                    v
                推荐结果
```

### 3. 冷启动处理

未登录用户或行为数据不足的新用户，自动降级为**全局热门推荐**，保证首屏永远有内容。

---

## 性能验证

对 10 个高频只读接口（首页推荐、分类列表、关键词搜索、搜索建议、阅读进度、热门推荐、评论分页等）
各压测 5 轮，**成功率 100%**：

| 指标 | 数值 |
|---|---|
| 平均响应时间 | **5.05 ms** |
| 最快 / 最慢 | 1.93 ms / 14.05 ms |
| 标准差 | 2.83 ms |

完整报告与原始截图见 [`docs/perf/api-perf-report.md`](docs/perf/api-perf-report.md)。

> 说明：本轮为**热点读接口的基线压测**；写接口与个性化推荐算法链路（依赖登录态与行为数据）暂未纳入。

---

## 快速开始

### 环境要求

- JDK 17+
- Maven 3.8+
- MySQL 8.0+
- Redis 6+
- Node.js 20.19+ 或 22.12+

### 1. 初始化数据库

```sql
CREATE DATABASE novel_db DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

表结构由 JPA 自动生成（`spring.jpa.hibernate.ddl-auto=update`）。

**示例数据**：数据库备份文件较大，通过百度网盘获取 —— [下载链接](https://pan.baidu.com/s/1SUVQqHmgeSi88I16S1OIvA?pwd=21k4)　提取码：`21k4`，下载后导入 MySQL 即可恢复数据。

### 2. 配置环境变量

不要把密钥写进配置文件提交到仓库，通过环境变量注入：

```bash
# Windows PowerShell
$env:DB_USERNAME="root"
$env:DB_PASSWORD="你的数据库密码"
$env:JWT_SECRET="用 openssl rand -hex 32 生成一个随机值"

# macOS / Linux
export DB_USERNAME=root
export DB_PASSWORD=你的数据库密码
export JWT_SECRET=$(openssl rand -hex 32)
```

### 3. 启动后端

```bash
cd novel-system
mvn spring-boot:run
```

后端地址：`http://localhost:8080/api`

### 4. 启动前端

```bash
cd vue-project
npm install
npm run dev          # 读者端 http://localhost:8081
npm run dev:author   # 作者端 http://localhost:8082
npm run dev:admin    # 管理端 http://localhost:8083
```

---

## 项目结构

```
thesis-project/
|- novel-system/                    # 后端
|  |- src/main/java/com/novel/
|  |  |- controller/                # 16 个 Controller（145 个接口）
|  |  |- service/                   # 15 个业务服务
|  |  |- repository/                # 17 个 JPA Repository
|  |  |- model/                     # 18 个实体
|  |  |- dto/                       # 请求 / 响应对象
|  |  |- config/                    # Security / Redis / Charset 配置
|  |  '- exception/                 # 全局异常处理
|  '- src/main/resources/
|     '- application.yml
|- vue-project/                     # 前端
|  '- src/
|     |- api/modules/               # 按模块拆分的接口封装
|     |- components/                # 通用组件
|     |- views/                     # 页面（读者 / 作者 / 管理）
|     |- router/  stores/  utils/
|     '- assets/
'- docs/perf/                       # 压测报告与原始数据
```

---

## 接口示例

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth/login` | 登录，返回 JWT |
| POST | `/api/auth/register` | 注册 |
| GET | `/api/novels` | 小说列表（分页） |
| GET | `/api/search?keyword=` | 关键词搜索 |
| GET | `/api/search/advanced` | 高级搜索（分类 / 作者 / 字数 / 状态 / 排序） |
| GET | `/api/recommendations/personalized` | 个性化推荐（未登录自动降级热门） |
| GET | `/api/recommendations/popular` | 热门推荐 |
| GET | `/api/reading/progress` | 获取阅读进度 |
| POST | `/api/comments` | 发表评论 |

完整接口文档见仓库内的 API 文档。

---

## 已知改进项

- [ ] 补齐核心 Service 的单元测试（推荐打分逻辑目前依赖手动验证）
- [ ] 对章节列表、评论分页等高基数字段补充索引，并用 EXPLAIN 记录优化前后对比
- [ ] 将个性化推荐算法链路纳入压测范围（需构造测试用户与行为数据）
- [ ] 将 `@CrossOrigin(origins = "*")` 收敛为白名单
- [ ] 推荐算法引入离线评估指标（准确率 / 召回率）

---

## 作者

谢宏枭 · 2026 届 · 安徽信息工程学院 计算机科学与技术

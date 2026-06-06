## Java 项目代码规范（通用）

0. 协作流程（最重要）

- **不要自己执行项目的打包、运行命令**：`java` / `gradle` / `mvn` 等一律不跑。
- 改完代码直接告诉我变更点，编译、类型检查、构建、启动由我执行。
- 如果确实需要启动项目来验证，先告诉我，由我决定是否启。
- 可以做的：读代码、改代码。

1. 分层 & 包结构

```
  domain/{aggregate}/         — Entity, Repository(接口), Service, VO, Enum
  application/
    events/                   — 应用事件对象，如 CatInteractedEvent
    service/{aggregate}/      — AppService
    coordinator/{aggregate}/  — Coordinator, AppService 与 domain / Domain Service 之间的应用层中间层
    shared/{aggregate}/       — 应用层共享语义，如跨 RO / Coordinator / AppService 使用的枚举、轻量值对象
    arv/assembler/            — Domain / 应用层对象与 VO 之间的装配转换
    arv/ro/{aggregate}/       — Request Object
    arv/vo/{aggregate}/       — View Object
    profile/                  — ApplicationProfile
  infrastructure/
    repositoryimpl/{aggregate}/  — PO, Mapper, RepositoryImpl
    properties/               — @ConfigurationProperties 类
    config/                   — @Configuration / Bean 装配
    {component}/              — OssClient, WorkWeiXin 等基础设施组件
  interfaces/controller/{aggregate}/
  interfaces/listeners/{aggregate}/
```

2. Domain Entity

- 尽量不暴露 @Getter；按需给必要字段 / 方法暴露访问能力 + @ToString + @EqualsAndHashCode(of = "id")
- 字段全部 @Nonnull / @Nullable 强标注
- 私有无参构造 + 私有 @JsonCreator 构造（给 Jackson 反序列化用）
- Jackson 序列化基于字段（field-based），不依赖 getter；分析/改动实体序列化行为直接看字段 + @JsonCreator，无需关注 getter 是否暴露
- 静态工厂方法 命名：通用用 new_()，业务化的用动词（Mood.checkin()）
- 单字段修改：resetXxx(value)
- 谓词：is/can 前缀（isOwner / canModify / canBeViewedBy），纯函数、返回 boolean、不抛异常、不依赖外部上下文
- ID：IdGenerator.snowflakeId()（er.rennala.kits）
- 校验类异常可以从 domain 抛 BizException（Album.sort() 是范例），但鉴权的抛异常留给 AppService，domain 只暴露谓词
- 聚合内的有序集合用 LinkedHashSet，对外用 getXxxAsCopy() 返不可变副本

3. Domain Service

- 位置：domain/{aggregate}/{Aggregate}Service.java，和 Entity 平级（不另起 services 包）
- 命名：{Aggregate}Service（AlbumService）
- 何时引入：业务逻辑操作多个同类聚合（如批量排序），无法落在单个 Entity 上时
- 单例模式（避免 Spring 依赖污染 domain）：
  - private static final {Service} INSTANCE = new {Service}();
  - public static {Service} getInstance() { return INSTANCE; }
  - 私有无参构造
- 方法接收 Entity 集合作参数，通过调 Entity 的 reset / 谓词方法做 mutate
- 不持有状态，不依赖 Repository / 外部组件，纯业务逻辑
- 可以 @Slf4j；可以抛 BizException（校验类，跟 Entity 同款）；鉴权异常留给 AppService
- AppService 用字段持有 `private final AlbumService albumService = AlbumService.getInstance();`，按普通方法调

4. Repository

- 接口在 domain/{aggregate}/{Aggregate}Repository
- 实现 @Repository class {Aggregate}RepositoryImpl
- 方法命名:save / delete / findById / findByXxx / lock（for update）
- 返回 Optional<T> 或 List<T>，永远不返回 null，空集合用 List.of()
- 查询用 LambdaQueryWrapper；空入参集合短路返回（避免 IN () 报错）
- Mapper 继承 DereferenceEnhancedMapper<PO>，@Mapper

5. PO

- @TableName(value = "{prefix}_{name}", autoResultMap = true)
- 表名前缀：m_ mood、u_ user、al_ album、ca_ ca、lan_ lan
- @TableId 主键；SQL 关键字字段加反引号 ``@TableField("`date`")``
- 嵌套对象 / 集合 → @TableField(typeHandler = JacksonTypeHandler.class) 存 JSON
- @Setter @Getter @ToString，没有手写构造
- toPO(domain) / toDomain(po) 静态方法，统一用 JacksonObjectMapper.convert
- 枚举不加特殊 handler，按 name 存 varchar

6. AppService

- @Service + @Slf4j
- 写操作 @Transactional(rollbackFor = Exception.class)
- 第一个参数永远是 ApplicationProfile p（鉴权上下文）
- 参数校验集中在方法开头，统一 throw new BizException(Codes.ArgsIllegal)
- 鉴权三步走：加载 → 调谓词 → 不通过抛异常
- 业务异常用 CodeAndMessage 项目枚举；通用异常用 er.rennala.response.Codes
- 编排薄、不写规则逻辑，规则在 domain 方法 / Domain Service 里
- AppService 与 domain / Domain Service 之间的中间编排可下沉到 application/coordinator/{aggregate}，避免 AppService 变厚
- RO、Coordinator、AppService 都需要使用的应用层语义放 application/shared/{aggregate}；Coordinator 不依赖 RO

7. Event / Publisher / Listener

- 事件对象放 application/events，命名 `{Biz}Event`，如 CatInteractedEvent
- 事件是应用层事实载体，不放业务逻辑；字段用 @Nonnull/@Nullable 标注
- 事件发布方通常是 AppService，使用 ApplicationEventPublisher，在业务写入成功后 publishEvent
- Listener 放 interfaces/listeners/{aggregate}，@Component + @Slf4j
- Listener 只做：监听事件 → 调 AppService；不直接调 Repository，不写业务规则
- 需要事务提交后处理副作用时，Listener 用 @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
- Listener catch 异常时 log.error(..., e)，避免事件副作用影响主流程
- 事件处理链路：publisher → interfaces/listeners → application/service → application/coordinator → domain / repository
- Coordinator 负责把事件 / 应用层参数编排成领域对象或调用领域服务，不持有 Listener 职责
- Event 可以作为 AppService / Coordinator 入参；如果事件开始携带投递元数据、跨进程消息字段，或一个事件被多个用例以不同语义消费，应在 Listener 中转换成 application/shared/{aggregate} 的 Command / 轻量值对象

8. Controller

- @RestController + @RequestMapping("/{aggregate}")（单数）
- 字段命名：AlbumAppService s + HttpServletRequest request（新代码用 s，老的 service 不强求统一）
- 路径 kebab-case，动词风格：/create /update /delete/{id} /find-recent /find-by-month /find-someones-albums
- 写操作 @PostMapping + @RequestBody {Entity}{Action}RO
- 读操作 @GetMapping，查询参数直接绑定（不必加 @RequestParam）；id 走 @PathVariable
- profile 从 ProfileReader.getProfileOrElseThrows(request) 拿
- 返回 R.ok(...) / R.ok()（R<Void>）
- 不放业务逻辑，只做：拿 profile → 调 AppService → 包 R

9. 错误处理

- 通用错误码：er.rennala.response.Codes（ArgsIllegal / RecordNotFound / ServerError / TokenInvalid）
- 业务错误码：domain/global/CodeAndMessage，按域分段编号（wx 70100 段、oss 70200 段、ca 10200 段...）
- 异常类：er.rennala.response.BizException
- 不向前端泄露原始异常 message；catch 里 log.error(..., e) 记日志，再抛预定义错误码
- 反探测：用户/相册不存在或无权访问，返回与"业务失败"相同的错误码

10. RO / VO / Assembler

- RO：{Entity}{Action}RO（AlbumCreateRO / AlbumUpdateRO / MoodCheckinRO）
- VO：{Entity}VO
- 字段 public，加 @Nonnull/@Nullable，类上加 @ToString
- 没有 setter（用 public 字段）
- VO 只承载接口输出数据，不依赖 Domain Entity，不定义 `of()` / `from()` 等转换方法
- Domain Entity / 应用层对象 → VO 的转换统一放 `application/arv/assembler`
- Assembler 命名 `{Entity}Assembler`，使用 `@Component`，由 AppService 注入调用；Controller 不直接调用 Assembler
- 单对象转换方法命名 `toVO(entity)`；集合转换方法命名 `toVOs(entities)`
- 集合遍历和逐项转换统一由 Assembler 完成；AppService 直接调用 `assembler.toVOs(entities)`，不自行 `stream().map(assembler::toVO)`
- 简单同名字段转换可使用 `JacksonObjectMapper.convert`；涉及字段重命名、组合、脱敏、外部查询时显式赋值
- Assembler 只负责数据转换和展示装配，不做鉴权、参数校验、业务规则，不直接写 Repository
- 需要 Repository / 外部数据补充 VO 时可注入依赖，但查询应服务于展示装配，复杂查询和业务编排仍留在 AppService / Coordinator

11. 数据库 / Liquibase

- 路径：src/main/resources/liquibase/v{ver}/{seq}_{tableName}.xml
- master.xml 集中 <include>
- changeSet id 格式：table-yyyyMMdd-NN、data-yyyyMMdd-NN
- 表与索引拆 createTable + createIndex
- 枚举存 varchar(16)；JSON 字段 type="json"；时间 timestamp；布尔 tinyint

12. 配置

- application.yml 主配置，profile 切换：active: ${CONFIGURATION_TAIL} 环境变量
- application-{profile}.yaml 放在项目根目录（不在 resources）
- Properties 类：@Component + @ConfigurationProperties(prefix = "kebab-case") + @Setter @Getter @ToString
- 字段都给默认值（空字符串 / 默认数字），避免 NPE

13. 工具 / 常用库

┌─────────────────┬──────────────────────────────────────────────────┐
│      用途       │                       选用                       │
├─────────────────┼──────────────────────────────────────────────────┤
│ 空集合判断      │ cn.hutool.core.collection.CollUtil.isEmpty       │
├─────────────────┼──────────────────────────────────────────────────┤
│ 空字符串        │ cn.hutool.core.util.StrUtil.isBlank / isNotBlank │
├─────────────────┼──────────────────────────────────────────────────┤
│ 对象比较 / 判空 │ java.util.Objects.equals / isNull / nonNull      │
├─────────────────┼──────────────────────────────────────────────────┤
│ 时间            │ Instant / LocalDate / YearMonth                  │
├─────────────────┼──────────────────────────────────────────────────┤
│ ID              │ er.rennala.kits.IdGenerator.snowflakeId()        │
├─────────────────┼──────────────────────────────────────────────────┤
│ Jackson         │ er.rennala.kits.JacksonObjectMapper.convert      │
├─────────────────┼──────────────────────────────────────────────────┤
│ 日志            │ @Slf4j                                           │
└─────────────────┴──────────────────────────────────────────────────┘

14. 代码细节

- 方法 ≤ 50 行
- 中文注释，// 后留空格，中文汉字 + 英文标点
- domain 类顶部加 /** <p> 一句话说明 */
- 新增 / 修改 public class、record、interface、enum 时，顶部加 Javadoc，说明一句核心职责
- public 方法必须加 Javadoc，说明方法意图、关键业务规则、@param、@return
- private 方法如果包含业务语义、转换规则、分支策略、非显而易见的计算，也要加 Javadoc
- 简单 getter / setter / 构造器 / 明显自解释的私有小工具方法，不强制加 Javadoc
- 方法注释写业务含义，不复述代码；复杂副作用要说明触发时机和失败影响
- @Nonnull/@Nullable 不要省
- 多个参数换行对齐，每行一个
- 不要返回 null 集合
- 字段名 s 表示 AppService、p 表示 Profile、ro 表示 Request Object

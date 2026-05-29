## Java 项目代码规范（通用）

0. 协作流程（最重要）

- **不要自己执行项目的打包、运行命令**：`java` / `gradle` / `mvn` 等一律不跑。
- 改完代码直接告诉我变更点，编译、类型检查、构建、启动由我执行。
- 如果确实需要启动项目来验证，先告诉我，由我决定是否启。
- 可以做的：读代码、改代码。

1. 分层 & 包结构

  domain/{aggregate}/         — Entity, Repository(接口), Service, VO, Enum
  application/
    service/{aggregate}/      — AppService
    arv/ro/{aggregate}/       — Request Object
    arv/vo/{aggregate}/       — View Object
    profile/                  — ApplicationProfile
  infrastructure/
    repositoryimpl/{aggregate}/  — PO, Mapper, RepositoryImpl
    properties/               — @ConfigurationProperties 类
    config/                   — @Configuration / Bean 装配
    {component}/              — OssClient, WorkWeiXin 等基础设施组件
  interfaces/controller/{aggregate}/

2. Domain Entity

- 尽量不暴露 @Getter；按需给必要字段 / 方法暴露访问能力 + @ToString + @EqualsAndHashCode(of = "id")
- 字段全部 @Nonnull / @Nullable 强标注
- 私有无参构造 + 私有 @JsonCreator 构造（给 Jackson 反序列化用）
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

7. Controller

- @RestController + @RequestMapping("/{aggregate}")（单数）
- 字段命名：AlbumAppService s + HttpServletRequest request（新代码用 s，老的 service 不强求统一）
- 路径 kebab-case，动词风格：/create /update /delete/{id} /find-recent /find-by-month /find-someones-albums
- 写操作 @PostMapping + @RequestBody {Entity}{Action}RO
- 读操作 @GetMapping，查询参数直接绑定（不必加 @RequestParam）；id 走 @PathVariable
- profile 从 ProfileReader.getProfileOrElseThrows(request) 拿
- 返回 R.ok(...) / R.ok()（R<Void>）
- 不放业务逻辑，只做：拿 profile → 调 AppService → 包 R

8. 错误处理

- 通用错误码：er.rennala.response.Codes（ArgsIllegal / RecordNotFound / ServerError / TokenInvalid）
- 业务错误码：domain/global/CodeAndMessage，按域分段编号（wx 70100 段、oss 70200 段、ca 10200 段...）
- 异常类：er.rennala.response.BizException
- 不向前端泄露原始异常 message；catch 里 log.error(..., e) 记日志，再抛预定义错误码
- 反探测：用户/相册不存在或无权访问，返回与"业务失败"相同的错误码

9. RO / VO

- RO：{Entity}{Action}RO（AlbumCreateRO / AlbumUpdateRO / MoodCheckinRO）
- VO：{Entity}VO
- 字段 public，加 @Nonnull/@Nullable，类上加 @ToString
- 没有 setter（用 public 字段）

10. 数据库 / Liquibase

- 路径：src/main/resources/liquibase/v{ver}/{seq}_{tableName}.xml
- master.xml 集中 <include>
- changeSet id 格式：table-yyyyMMdd-NN、data-yyyyMMdd-NN
- 表与索引拆 createTable + createIndex
- 枚举存 varchar(16)；JSON 字段 type="json"；时间 timestamp；布尔 tinyint

11. 配置

- application.yml 主配置，profile 切换：active: ${CONFIGURATION_TAIL} 环境变量
- application-{profile}.yaml 放在项目根目录（不在 resources）
- Properties 类：@Component + @ConfigurationProperties(prefix = "kebab-case") + @Setter @Getter @ToString
- 字段都给默认值（空字符串 / 默认数字），避免 NPE

12. 工具 / 常用库

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

13. 代码细节

- 方法 ≤ 50 行
- 中文注释，// 后留空格，中文汉字 + 英文标点
- domain 类顶部加 /** <p> 一句话说明 */
- @Nonnull/@Nullable 不要省
- 多个参数换行对齐，每行一个
- 不要返回 null 集合
- 字段名 s 表示 AppService、p 表示 Profile、ro 表示 Request Object

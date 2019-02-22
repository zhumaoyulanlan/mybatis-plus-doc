# Sql 注入器

::: tip 注入器配置
全局配置 `sqlInjector` 用于注入 `ISqlInjector` 接口的子类，实现自定义方法注入。

例如逻辑删除注入器 [LogicSqlInjector](https://gitee.com/baomidou/mybatis-plus/blob/3.0/mybatis-plus-extension/src/main/java/com/baomidou/mybatisplus/extension/injector/LogicSqlInjector.java)
:::

- SQL 自动注入器接口 `ISqlInjector`
```java
public interface ISqlInjector {

    /**
     * <p>
     * 检查SQL是否注入(已经注入过不再注入)
     * </p>
     *
     * @param builderAssistant mapper 信息
     * @param mapperClass      mapper 接口的 class 对象
     */
    void inspectInject(MapperBuilderAssistant builderAssistant, Class<?> mapperClass);

    /**
     * <p>
     * 注入 SqlRunner 相关
     * </p>
     *
     * @param configuration 全局配置
     * @see ISqlRunner
     */
    void injectSqlRunner(Configuration configuration);
}
```

- 实现接口抽象类 `AbstractSqlInjector`
```java
public abstract class AbstractSqlInjector implements ISqlInjector {

    @Override
    public void inspectInject(MapperBuilderAssistant builderAssistant, Class<?> mapperClass) {
        String className = mapperClass.toString();
        Set<String> mapperRegistryCache = GlobalConfigUtils.getMapperRegistryCache(builderAssistant.getConfiguration());
        if (!mapperRegistryCache.contains(className)) {
            List<AbstractMethod> methodList = this.getMethodList();
            Assert.notEmpty(methodList, "No effective injection method was found.");
            // 循环注入自定义方法
            methodList.forEach(m -> m.inject(builderAssistant, mapperClass));
            mapperRegistryCache.add(className);
            /**
             * 初始化 SQL 解析
             */
            if (GlobalConfigUtils.getGlobalConfig(builderAssistant.getConfiguration()).isSqlParserCache()) {
                SqlParserHelper.initSqlParserInfoCache(mapperClass);
            }
        }
    }

    @Override
    public void injectSqlRunner(Configuration configuration) {
        // to do nothing
    }

    /**
     * <p>
     * 获取 注入的方法
     * </p>
     *
     * @return 注入的方法集合
     */
    public abstract List<AbstractMethod> getMethodList();
}
```

自定义自己的通用方法可以实现接口 `ISqlInjector` 也可以继承抽象类  `AbstractSqlInjector` 注入通用方法 `SQL 语句` ，然后继承 `BaseMapper` 添加自定义方法，全局配置 `sqlInjector` 注入 MP 会自动将类所有方法注入到 `mybatis` 容器中。

- Exam

当数据库使用MicroSoft SQl Server 时，由于Jdbc3KeyGenerator和MS SQL驱动不支持， Service 提供的 `saveBatch` 方法会出错，可以通过实现`ISqlInjector`或继承 `AbstractSqlInjector` 然后注入到容器。然后继承`ServiceImpl`类，重写`saveBatch` 和 `saveOrUpdateBatch`。

参考`AbstractMethod`的子类`Insert` 写一个不自动回填自增长类型ID的Insert方法
```java
public class InsertNoKeyGenerator extends AbstractMethod {
    
    //-------add-------注入sql对应的Mapper方法名
    public final static String INSERT_NO_KEY_GENERATOR="insertNoKeyGenerator";
    
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
        KeyGenerator keyGenerator = new NoKeyGenerator();
        SqlMethod sqlMethod = SqlMethod.INSERT_ONE;
        String columnScript = SqlScriptUtils.convertTrim(tableInfo.getAllInsertSqlColumn(false),
                LEFT_BRACKET, RIGHT_BRACKET, null, COMMA);
        String valuesScript = SqlScriptUtils.convertTrim(tableInfo.getAllInsertSqlProperty(false, null),
                LEFT_BRACKET, RIGHT_BRACKET, null, COMMA);
        String keyProperty = null;
        String keyColumn = null;
        // 表包含主键处理逻辑,如果不包含主键当普通字段处理
        if (StringUtils.isNotEmpty(tableInfo.getKeyProperty())) {
            if (tableInfo.getIdType() == IdType.AUTO) {
                /** 自增主键 */
                //----delete-----删除配置 Jdbc3KeyGenerator 将不再获取自增主键
                //keyGenerator = new Jdbc3KeyGenerator();
                keyProperty = tableInfo.getKeyProperty();
                keyColumn = tableInfo.getKeyColumn();
            } else {
                if (null != tableInfo.getKeySequence()) {
                    //--------update--------- 原来的代码为sqlMethod.getMethod(),  修改为  INSERT_NO_KEY_GENERATOR
                    keyGenerator = TableInfoHelper.genKeyGenerator(tableInfo, builderAssistant, INSERT_NO_KEY_GENERATOR, languageDriver);
                    keyProperty = tableInfo.getKeyProperty();
                    keyColumn = tableInfo.getKeyColumn();
                }
            }
        }
        String sql = String.format(sqlMethod.getSql(), tableInfo.getTableName(), columnScript, valuesScript);
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sql, modelClass);
        //--------update--------- 原来的代码为sqlMethod.getMethod(),  修改为  INSERT_NO_KEY_GENERATOR
        return this.addInsertMappedStatement(mapperClass, modelClass, INSERT_NO_KEY_GENERATOR, sqlSource, keyGenerator, keyProperty, keyColumn);
    }
}
```
实现`ISqlInjector`或继承`AbstractSqlInjector`
注意：由于注入`ISqlInjector`的实现类，会使原本默认的sql注入器`DefaultSqlInjector`失效，所以本处直接继承`ISqlInjector` 的实现类`LogicSqlInjector`。

```java
public class MSSqlLogicSqlInjector extends LogicSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList() {
        List<AbstractMethod> list = super.getMethodList();
        //增加一个不返回主键的insert方法
        list.add(new InsertNoKeyGenerator());
        return list;
    }
}
```

向容器注入`ISqlInjector`的实现类
```java
@Configuration
public class MybatisPlusConfig {
   @Bean
    public ISqlInjector mSSqlLogicSqlInjector(){
        return  new MSSqlLogicSqlInjector();
    }
}
```


继承`ServiceImpl`，参考`ServiceImpl`的`saveBatch`方法，修改要调用的`Mapper`中的方法名。ps：由于我们不需要在`Mapper`使用`insertNoKeyGenerator`方法，我们不再继承`BaseMapper `添加方法。
```java

public class BaseServiceImpl<M extends BaseMapper<T>, T> extends ServiceImpl<M, T> implements IService<T> {

 @Override
    public boolean saveBatch(Collection<T> entityList, int batchSize) {
        int i = 0;
        //--------update------获取Mapper的insertNoKeyGenerator方法全名
        String sqlStatement = SqlHelper.table(super.currentModelClass()).getSqlStatement(InsertNoKeyGenerator.INSERT_NO_KEY_GENERATOR);
        try (SqlSession batchSqlSession = sqlSessionBatch()) {
            for (T anEntityList : entityList) {
                batchSqlSession.insert(sqlStatement, anEntityList);
                if (i >= 1 && i % batchSize == 0) {
                    batchSqlSession.flushStatements();
                }
                i++;
            }
            batchSqlSession.flushStatements();
        }
        return true;
    }
    
    
      @Override
    public boolean saveOrUpdateBatch(Collection<T> entityList, int batchSize) {
        if (CollectionUtils.isEmpty(entityList)) {
            throw new IllegalArgumentException("Error: entityList must not be empty");
        }
        Class<?> cls = currentModelClass();
        TableInfo tableInfo = TableInfoHelper.getTableInfo(cls);
        int i = 0;
        try (SqlSession batchSqlSession = sqlSessionBatch()) {
            for (T anEntityList : entityList) {
                if (null != tableInfo && StringUtils.isNotEmpty(tableInfo.getKeyProperty())) {
                    Object idVal = ReflectionKit.getMethodValue(cls, anEntityList, tableInfo.getKeyProperty());
                    if (StringUtils.checkValNull(idVal) || Objects.isNull(getById((Serializable) idVal))) {
                        //---------------update-------------获取Mapper的insertNoKeyGenerator方法全名
                        batchSqlSession.insert(SqlHelper.table(currentModelClass()).getSqlStatement(InsertNoKeyGenerator.INSERT_NO_KEY_GENERATOR), anEntityList);
                    } else {
                        MapperMethod.ParamMap<T> param = new MapperMethod.ParamMap<>();
                        param.put(Constants.ENTITY, anEntityList);
                        batchSqlSession.update(sqlStatement(SqlMethod.UPDATE_BY_ID), param);
                    }
                    if (i >= 1 && i % batchSize == 0) {
                        batchSqlSession.flushStatements();
                    }
                    i++;
                } else {
                    throw ExceptionUtils.mpe("Error:  Can not execute. Could not find @TableId.");
                }
                batchSqlSession.flushStatements();
            }
        }
        return true;
    }
}
```






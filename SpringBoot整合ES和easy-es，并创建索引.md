## SpringBoot整合ES和easy-es，并创建索引

#### 一、easy-es介绍

easy-es是针对elasticSerarch的一个简易的新型操作框架，等同于官方的RestHighLevelClient ，在一些简单的增删改查和一般的MySQL一样，简洁方便，但是一些复杂的聚合查询就比较麻烦且部分不支持，不过支持RestHighLevelClient 和easy-es进行混合查询。新手建议还是用RestHighLevelClient ，免得要学习两种语法，糅合起来特别困难，对熟悉es的可以使用，能节省很多的不必要的代码。[官网在这](https://www.easy-es.cn/pages/949ac4/#%E7%BC%96%E7%A0%81)。

#### 二、easy-es相关配置

pom依赖如下

```xml
<dependency>
            <groupId>cn.easy-es</groupId>
            <artifactId>easy-es-boot-starter</artifactId>
            <version>1.0.3</version>
        </dependency>
```

easy-es的依赖已经加载es的依赖了，所以不用再额外添加es的依赖了。

配置文件如下

```yaml
# ee配置
easy-es:
  enable: true # 是否开启EE自动配置
  address : ${es.address} # es连接地址+端口 格式必须为ip:port,如果是集群则可用逗号隔开
  schema: https # 默认为http
  banner: false #去掉打印logo
  username: ${es.username} #如果无账号密码则可不配置此行
  password: ${es.password} #如果无账号密码则可不配置此行
  keep-alive-millis: 30000 # 心跳策略时间 单位:ms
  connectTimeout: 30000 # 连接超时时间 单位:ms
  socketTimeout: 30000 # 通信超时时间 单位:ms
  #  requestTimeout: 5000 # 请求超时时间 单位:ms
  connectionRequestTimeout: 30000 # 连接请求超时时间 单位:ms
  maxConnTotal: 100 # 最大连接数 单位:个
  maxConnPerRoute: 100 # 最大连接路由数 单位:个
  global-config:
    process_index_mode: manual #索引处理模式,smoothly:平滑模式,默认开启此模式, not_smoothly:非平滑模式, manual:手动模式
    print-dsl: true # 开启控制台打印通过本框架生成的DSL语句,默认为开启,测试稳定后的生产环境建议关闭,以提升少量性能
    distributed: true # 当前项目是否分布式项目,默认为true,在非手动托管索引模式下,若为分布式项目则会获取分布式锁,非分布式项目只需synchronized锁.
    asyncProcessIndexBlocking: true # 异步处理索引是否阻塞主线程 默认阻塞 数据量过大时调整为非阻塞异步进行 项目启动更快
    activeReleaseIndexMaxRetry: 60 # 分布式环境下,平滑模式,当前客户端激活最新索引最大重试次数若数据量过大,重建索引数据迁移时间超过60*(180/60)=180分钟时,可调大此参数值,此参数值决定最大重试次数,超出此次数后仍未成功,则终止重试并记录异常日志
    activeReleaseIndexFixedDelay: 180 # 分布式环境下,平滑模式,当前客户端激活最新索引最大重试次数 若数据量过大,重建索引数据迁移时间超过60*(180/60)=180分钟时,可调大此参数值 此参数值决定多久重试一次 单位:秒
    db-config:
      map-underscore-to-camel-case: false # 是否开启下划线转驼峰 默认为false
      #      table-prefix: daily_ # 索引前缀,可用于区分环境  默认为空 用法和MP一样
      id-type: customize # id生成策略 customize为自定义,id值由用户生成,比如取MySQL中的数据id,如缺省此项配置,则id默认策略为es自动生成
      field-strategy: not_empty # 字段更新策略  默认为not_null:非Null判断,字段值为非Null时,才会被更新 ； not_empty: 非空判断,字段值为非空字符串时才会被更新；ignore: 忽略判断,无论字段值为什么,都会被更新
      enable-track-total-hits: true # 默认开启,查询若指定了size超过1w条时也会自动开启,开启后查询所有匹配数据,若不开启,会导致无法获取数据总条数,其它功能不受影响.
      refresh-policy: immediate # 数据刷新策略,默认为不刷新
      enable-must2-filter: false # 是否全局开启must查询类型转换为filter查询类型 默认为false不转换
```

es自动配置类

```java
@Configuration
@ConditionalOnClass(RestHighLevelClient.class)
@EnableConfigurationProperties(EasyEsConfigProperties.class)
@ConditionalOnProperty(prefix = "easy-es", name = {"enable"}, havingValue = "true", matchIfMissing = true)
public class EsAutoConfiguration {
    @Autowired
    private EasyEsConfigProperties easyEsConfigProperties;

    /**
     * 装配RestHighLevelClient
     *
     * @return RestHighLevelClient bean
     */
    @Bean
    @ConditionalOnMissingBean
    public RestHighLevelClient restHighLevelClient() {
        SSLFactory sslFactory = SSLFactory.builder()
                .withTrustingAllCertificatesWithoutValidation()
                .withHostnameVerifier((host, session) -> true)
                .build();
        // 处理地址
        String address = easyEsConfigProperties.getAddress();
        if (StringUtils.isEmpty(address)) {
            throw ExceptionUtils.eee("please config the es address");
        }
        if (!address.contains(COLON)) {
            throw ExceptionUtils.eee("the address must contains port and separate by ':'");
        }
        String schema = StringUtils.isEmpty(easyEsConfigProperties.getSchema())
                ? DEFAULT_SCHEMA : easyEsConfigProperties.getSchema();
        List<HttpHost> hostList = new ArrayList<>();
        Arrays.stream(easyEsConfigProperties.getAddress().split(","))
                .forEach(item -> hostList.add(new HttpHost(item.split(":")[0],
                        Integer.parseInt(item.split(":")[1]), schema)));

        // 转换成 HttpHost 数组
        HttpHost[] httpHost = hostList.toArray(new HttpHost[]{});
        // 构建连接对象
        RestClientBuilder builder = RestClient.builder(httpHost);

        // 设置账号密码最大连接数之类的
        String username = easyEsConfigProperties.getUsername();
        String password = easyEsConfigProperties.getPassword();
        Integer maxConnTotal = easyEsConfigProperties.getMaxConnTotal();
        Integer maxConnPerRoute = easyEsConfigProperties.getMaxConnPerRoute();
        Integer keepAliveMillis = easyEsConfigProperties.getKeepAliveMillis();
        boolean needSetHttpClient = (StringUtils.isNotEmpty(username) && StringUtils.isNotEmpty(password))
                || (Objects.nonNull(maxConnTotal) || Objects.nonNull(maxConnPerRoute)) || Objects.nonNull(keepAliveMillis);
        if (needSetHttpClient) {
            builder.setHttpClientConfigCallback(httpClientBuilder -> {
                // 设置心跳时间等
                Optional.ofNullable(keepAliveMillis).ifPresent(p -> httpClientBuilder.setKeepAliveStrategy((response, context) -> p));
                Optional.ofNullable(maxConnTotal).ifPresent(httpClientBuilder::setMaxConnTotal);
                Optional.ofNullable(maxConnPerRoute).ifPresent(httpClientBuilder::setMaxConnPerRoute);
                if (StringUtils.isNotEmpty(username) && StringUtils.isNotEmpty(password)) {
                    final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
                    // 设置账号密码
                    credentialsProvider.setCredentials(AuthScope.ANY,
                            new UsernamePasswordCredentials(easyEsConfigProperties.getUsername(), easyEsConfigProperties.getPassword()));
                    httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                }
                httpClientBuilder.setSSLContext(sslFactory.getSslContext()).setSSLHostnameVerifier(sslFactory.getHostnameVerifier());
                return httpClientBuilder;
            });
        }

        // 设置超时时间之类的
        Integer connectTimeOut = easyEsConfigProperties.getConnectTimeout();
        Integer socketTimeOut = easyEsConfigProperties.getSocketTimeout();
        Integer connectionRequestTimeOut = easyEsConfigProperties.getConnectionRequestTimeout();
        boolean needSetRequestConfig = Objects.nonNull(connectTimeOut) || Objects.nonNull(connectionRequestTimeOut);
        if (needSetRequestConfig) {
            builder.setRequestConfigCallback(requestConfigBuilder -> {
                Optional.ofNullable(connectTimeOut).ifPresent(requestConfigBuilder::setConnectTimeout);
                Optional.ofNullable(socketTimeOut).ifPresent(requestConfigBuilder::setSocketTimeout);
                Optional.ofNullable(connectionRequestTimeOut)
                        .ifPresent(requestConfigBuilder::setConnectionRequestTimeout);
                return requestConfigBuilder;
            });
        }

        return RestHighLevelClientBuilder.build(builder);
    }

}
```

并再springboot启动类上面加上@EsMapperScan注解

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.sf.retail.**"})
@EsMapperScan("com.sf.retail.order.elasticsearch.mapper")
public class NewRetailOrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(NewRetailOrderApplication.class, args);
    }

}
```

注意：es中的mapper和mybatis的mapper不要在同一个文件夹下面，各自分开，不然会报错，具体可以看官方文档的示例。

最后添加好实体类就可以，配置好esIP地址就可以进行简单的增删改查了。

#### 三、索引和索引别名的构建

es中可以建立索引别名，把多个索引联合搜索。在easy-es中可以在实体类上加上注解，然后在配置文件中开启自动配置。也可以采用自己配置，更加灵活，我这边使用的是自己配置。

先创建一张SQL表，表结构如下

```mysql
CREATE TABLE `es_index_create_config`  (
  `id` bigint(0) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `mapping_json` text CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT 'mapping映射',
  `index_name` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '表名',
  `status` tinyint(1) NULL DEFAULT 1 COMMENT '是否有效 0删除 1可用',
  `create_time` datetime(0) NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime(0) NULL DEFAULT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 9 CHARACTER SET = utf8 COLLATE = utf8_general_ci COMMENT = 'es索引创建表' ROW_FORMAT = Dynamic;
```

再添加一个自动更新索引的job，代码如下

```java
@RestController
@RequestMapping("/job")
@Slf4j
public class AutoCreateEsIndexJob {
    private static final String[] INDEX_SUFFIX = new String[]{"0103", "0406", "0709", "1012"};
    @Autowired
    private RestHighLevelClient restHighLevelClient;

    @Autowired
    private TEsIndexCreateConfigMapper tEsIndexCreateConfigMapper;


    @RequestMapping("/autoCreateEsIndex")
    public boolean autoCreateEsIndex() throws Exception {
        log.info("execute autoCreateEsIndexHandler");
        LambdaQueryWrapper<TEsIndexCreateConfig> wrapper = Wrappers.lambdaQuery(TEsIndexCreateConfig.class)
                .eq(TEsIndexCreateConfig::getStatus, true);
        List<TEsIndexCreateConfig> needCreateIndexList = tEsIndexCreateConfigMapper.selectList(wrapper);
        if (CollectionUtils.isEmpty(needCreateIndexList)) {
            return true;
        }
        LocalDate now = LocalDate.now();
        String nowYear = String.valueOf(now.getYear());
        needCreateIndexList.forEach(config -> {
            String mappingJson = config.getMappingJson();
            String indexName = config.getIndexName();
            for (String indexSuffix : INDEX_SUFFIX) {
                String index = String.format("%s_%s%s", indexName, nowYear, indexSuffix);
                log.info("正在处理 ====>> index:{}", index);
                try {
                    if (exists(indexName, nowYear, indexSuffix)) {
                        printLog(index);
                    } else {
                        create(mappingJson, index,indexName);
                    }
                } catch (Exception e) {
                    log.error(e.getMessage(), e);
                }
            }
        });
        return true;
    }

    private void printLog(String index) {
        log.info("ES索引名:{},已经被创建了！", index);
    }

    private boolean exists(String indexName, String year, String indexSuffix) throws IOException {
        String index = String.format("%s_%s%s", indexName, year, indexSuffix);
        GetIndexRequest getIndexRequest=new GetIndexRequest(index);
        boolean exists = restHighLevelClient.indices().exists(getIndexRequest, RequestOptions.DEFAULT);
        log.info("index exists{}", exists);
        return exists;
    }

    private void create(String mappingJson, String index,String indexName) throws IOException {
        CreateIndexRequest request = new CreateIndexRequest(index);
        request.settings(
                Settings.builder()
                        .put("index.max_result_window", 1000000)
                        .put("index.number_of_shards", 3)
                        .put("index.number_of_replicas", 2));
        request.mapping(mappingJson, XContentType.JSON);
        request.alias(new Alias(indexName));
        CreateIndexResponse createIndexResponse =
                restHighLevelClient.indices().create(request, RequestOptions.DEFAULT);
        String indexCreate = createIndexResponse.index();
        log.info("索引创建成功:{}，并添加别名:{}", indexCreate, indexName);
    }
}
```

索引名称可以按照自己业务自定义，我这边使用的按时间分季度去计算的，INDEX_SUFFIX里面对应的4个季度。

在修改了表结构后，先更新es_index_create_config里的mapping_json，这里面的是实体类在es中索引（表）结构，一定要先更改这个，不然直接更新数据不起作用，改完后再执行一下这个job，重新创建索引，最后再去更新数据！

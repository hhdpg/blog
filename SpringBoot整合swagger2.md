## SpringBoot整合swagger2

#### 一、swagger介绍

​	swagger是后端通过配置可以直接生成接口文档的一个框架，本文将介绍swagger的一些基础配置，全局token配置，以及实现接口测试。

#### 二、swagger基础配置	

1. maven依赖

```xml
<properties>
        <swagger2.version>2.9.2</swagger2.version>
</properties>
<!-- swagger -->
	   <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>${swagger2.version}</version>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>${swagger2.version}</version>
        </dependency>
```

2. 基础配置和全局token配置

   下文是swagger的一些基础配置以及全局的token配置，直接copy就可以适用了。

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket createRestApi(Environment environment) {
        //仅在开发环境，本地，测试环境可以使用swagger，生产环境不要配置
        Profiles profiles = Profiles.of("dev", "local", "uat");
        boolean flag = environment.acceptsProfiles(profiles);
        
        return new Docket(DocumentationType.SWAGGER_2)
                //文档title注释等
                .apiInfo(apiInfo())
                .enable(flag)
                .select()
                //swagger扫描路径
                .apis(RequestHandlerSelectors.basePackage("com.sf.retail.common.services.controller.sysMsg"))
                //过滤路径
                .paths(PathSelectors.any())
                .build()
                //进行全局token的配置
                .securitySchemes(securitySchemes())
                .securityContexts(securityContexts());
    }


    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("service-xx-vv-vvvv")
                .description("xx vv vvv")
                .version("0.0.1")
                .build();
    }

    //设置基本的介绍信息(无需更改)
    private List<ApiKey> securitySchemes() {
        List<ApiKey> apiKeyList = new ArrayList<>();
        apiKeyList.add(new ApiKey("Authorization", "Authorization", "header"));
        return apiKeyList;
    }
    //过滤不需要进行验证的页面
    private List<SecurityContext> securityContexts() {
        List<SecurityContext> securityContexts = new ArrayList<>();
        securityContexts.add(
                SecurityContext.builder()
                        .securityReferences(defaultAuth())
                        //带有auth的页面将不用提供token即可访问。
                        .forPaths(PathSelectors.regex("^(?!auth).*$"))
                        .build());
        return securityContexts;
    }
    //全局的token配置
    private List<SecurityReference> defaultAuth() {
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        List<SecurityReference> securityReferences = new ArrayList<>();
        securityReferences.add(new SecurityReference("Authorization", authorizationScopes));
        return securityReferences;
    }
}
```

#### 三、启动swagger界面

​	启动服务后，在浏览器中输入这个网址<http://localhost:服务的端口/swagger-ui.html> ，就可以看到如下场景说明启动成功了。

![](C:\Users\01415465\Desktop\markdown\photo\swagger启动成功.png)

点击controller右边的箭头就可以看到自动生成了每个接口的接口文档，models是接口中所包含的实体类

#### 四、在接口上丰富接口描述

把以下这些参数放在每个接口就可以，写好每一个接口文档是每个后端的职责

```java
@ApiOperation(value = "接口名称", notes = "接口注释")
@ApiImplicitParams({
            @ApiImplicitParam(name = "参数名", value = "参数描述", required = true, paramType = "query", dataType = "参数数据类型，eg:String， Integer, Boolean")
    })
```

#### 五、进行简单的接口测试

​	打开controller列表后，点击任意一个接口，可以在右上角看到一个<kbd>try it out</kbd>的按钮，点击一下这个按钮。如果配置了四中的详细的参数，直接填写参数内容即可。如果没有配置四中详细的参数，在request中填入json数据，和postman测试是一样的。然后点击<kbd>excute</kbd>按钮即可。

![](C:\Users\01415465\Desktop\markdown\photo\swagger接口测试.png)

如果要使用token，之前我们配置了全局的token，见上图swagger启动成功界面的右上角，有个<kbd>authorize</kbd>的绿色小图标，点击进去后填入token值即可。

# sentinel 结合 nacos 实现规则持久化
1、为什么要将 Sentienl 规则持久化？
-----------------------

一旦我们重启应用，sentinel 规则将消失，生产环境需要将配置规则进行持久化

2、持久化的思路
--------

我们现在将限流配置规则持久化进 Nacos 保存，只要刷新 8401 某个 rest 地址，sentinel 控制台的流控规则就能看到，只要 Nacos 里面的配置不删除，针对 8401 上 sentinel 上的流控规则持续有效。

3、操作步骤
------

### 3.1 修改 Nacos 依赖项
首先，你需要打开 sentinel-dashboard 项目的 pom.xml 文件，找到其中的依赖项 sentinel-datasource-nacos，它是连接 Nacos Config 所依赖的必要组件。

但这里有一个问题。在 Sentinel 的源码中，sentinel-datasource-nacos 的 scope 是 test，意思是依赖项只在项目编译期的 test 阶段才会生效。

所以接下来，你需要将这个依赖项的标签注释掉。
```xml
<!-- for Nacos rule publisher sample -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <!-- 将scope注释掉，改为编译期打包 -->
    <!--<scope>test</scope>-->
</dependency>
```
我们将test这一行代码注释掉以后，sentinel-datasource-nacos 就将作为编译期的依赖项，被打包到最终的 sentinel-dashboard.jar 执行文件中。

依赖项就这么轻松地修改完毕了，接下来我们就可以在后端程序中实现 Nacos Config 的对接了。
### 3.2 后端程序对接 Nacos
首先，你需要打开 sentinel-dashboard 项目下的 src/test/java 目录（注意是 test 目录而不是 main 目录），然后定位到 com.alibaba.csp.sentinel.dashboard.rule.nacos 包。在这个包下面，你会看到 4 个和 Nacos Config 有关的类，它们的功能描述如下。

* NacosConfig：初始化 Nacos Config 的连接；
* NacosConfigUtil：约定了 Nacos 配置文件所属的 Group 和文件命名后缀等常量字段；
* FlowRuleNacosProvider：从 Nacos Config 上获取限流规则；
* FlowRuleNacosPublisher：将限流规则发布到 Nacos Config。

为了让这些类在 Sentinel 运行期可以发挥作用，你需要在 src/main/java 下创建同样的包路径，然后将这四个文件从 test 路径拷贝到 main 路径下。
接下来，我们要做两件事，一是在 NacosConfig 类中配置 Nacos 连接信息，二是在 Controller 层接入 Nacos 做限流规则持久化。

我们先从 Nacos 连接信息改起。你需要打开 NacosConfig 类，找到其中的 nacosConfigService 方法。这个方法创建了一个 ConfigService 类，它是 Nacos Config 定义的通用接口，提供了 Nacos 配置项的读取和更新功能。

FlowRuleNacosProvider 和 FlowRuleNacosPublisher 这两个类都是基于这个 ConfigService 类实现 Nacos 数据同步的。我们来看一下改造后的代码。
```java
@Configuration
@ConfigurationProperties(prefix = "nacos")
@Data
public class NacosConfig {
    // nacos地址
    private String serverAddr;
    private String username;
    private String password;
    private String namespace;

    @Bean
    public ConfigService nacosConfigService() throws Exception {
        Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
        properties.put(PropertyKeyConst.USERNAME, username);
        properties.put(PropertyKeyConst.PASSWORD, password);
        properties.put(PropertyKeyConst.NAMESPACE, namespace);
        return ConfigFactory.createConfigService(properties);
    }
}
```
在上面的代码中，我通过自定义的 Properties 属性构造了一个 ConfigService 对象，将 ConfigService 背后的 Nacos 数据源地址指向了 localhost:8848，并指定了命名空间为 dev。这里我采用了硬编码的方式，你也可以对上面的实现过程做进一步改造，通过配置文件来注入 serverAddr 和 namespace 等属性。

```properties
nacos.addr=localhost:8848
```

这样，我们就完成了第一件事：在 NacosConfig 类中配置了 Nacos 连接信息。

第二件事，你需要在 Controller 层接入 Nacos 来实现限流规则持久化。

我们需要在 FlowControllerV2 中正式接入 Nacos 。FlowControllerV2 对外暴露了 REST API，用来创建和修改限流规则。在这个类的源代码中，你需要修改两个变量的 Qualifier 注解值。
```java
@Autowired
// 指向刚才我们从test包中迁移过来的FlowRuleNacosProvider类
// @Qualifier("flowRuleDefaultProvider")
@Qualifier("flowRuleNacosProvider")
private DynamicRuleProvider<List<FlowRuleEntity>> ruleProvider;
 
@Autowired
// 指向刚才我们从test包中迁移过来的FlowRuleNacosPublisher类
// @Qualifier("flowRuleDefaultPublisher")
@Qualifier("flowRuleNacosPublisher")
private DynamicRulePublisher<List<FlowRuleEntity>> rulePublisher;
```

在代码中，我通过 Qualifier 标签将 FlowRuleNacosProvider 注入到了 ruleProvier 变量中，又采用同样的方式将 FlowRuleNacosPublisher 注入到了 rulePublisher 变量中。FlowRuleNacosProvider 和 FlowRuleNacosPublisher 就是上一步我们刚从 test 目录 Copy 到 main 目录下的两个类。

修改完成之后，FlowControllerV2 底层的限流规则改动就会被同步到 Nacos 服务器了。这个同步工作是由 FlowRuleNacosPublisher 执行的，它会发送一个 POST 请求到 Nacos 服务器来修改配置项。我来带你看一下 FlowRuleNacosPublisher 类的源码。

```java
/*
 * Copyright 1999-2018 Alibaba Group Holding Ltd.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package com.alibaba.csp.sentinel.dashboard.rule.nacos.flow;

import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.FlowRuleEntity;
import com.alibaba.csp.sentinel.dashboard.rule.DynamicRulePublisher;
import com.alibaba.csp.sentinel.dashboard.rule.nacos.NacosConfigUtil;
import com.alibaba.csp.sentinel.datasource.Converter;
import com.alibaba.csp.sentinel.util.AssertUtil;
import com.alibaba.nacos.api.config.ConfigService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * @author Eric Zhao
 * @since 1.4.0
 */
@Component("flowRuleNacosPublisher")
public class FlowRuleNacosPublisher implements DynamicRulePublisher<List<FlowRuleEntity>> {

    @Autowired
    private ConfigService configService;
    @Autowired
    private Converter<List<FlowRuleEntity>, String> converter;

    @Override
    public void publish(String app, List<FlowRuleEntity> rules) throws Exception {
        AssertUtil.notEmpty(app, "app name cannot be empty");
        if (rules == null) {
            return;
        }
        configService.publishConfig(app + NacosConfigUtil.FLOW_DATA_ID_POSTFIX,
            NacosConfigUtil.GROUP_ID, converter.convert(rules));
    }
}

```

在上面的代码中，FlowRuleNacosPublisher 会在 Nacos Config 上创建一个用来保存限流规则的配置文件，这个配置文件以“application.name”开头，以“-flow-rules”结尾，而且它所属的 Group 为“SENTINEL_GROUP”。这里用到的文件命名规则和 Group 都是通过 NacosConfigUtil 类中的常量指定的。

```java
 
public final class NacosConfigUtil {

    // 这个是Sentinel注册的配置项所在的分组
    public static final String GROUP_ID = "SENTINEL_GROUP";

    // 流量整形规则的后缀
    public static final String FLOW_DATA_ID_POSTFIX = "-flow-rules";
}
```

对于DegradeController，我们可以参照FlowControllerV2的方法。

此处我们现根据flowRuleNacosProvider和flowRuleNacosPublisher，以相同的模式构建degradeRuleNacosProvider和degradeRuleNacosPublisher。这里不赘述。

```java
// 添加如下两行
@Autowired
@Qualifier("degradeRuleNacosProvider")
private DynamicRuleProvider<List<DegradeRuleEntity>> ruleProvider;
@Autowired
@Qualifier("degradeRuleNacosPublisher")
private DynamicRulePublisher<List<DegradeRuleEntity>> rulePublisher;

// 修改apiQueryMachineRules方法中的以下内容，改变rules的来源
// List<DegradeRuleEntity> rules = sentinelApiClient.fetchDegradeRuleOfMachine(app, ip, port);
List<DegradeRuleEntity> rules = ruleProvider.getRules(app);

// 修改publish方法的以下内容，改变rules的发布方式
// return sentinelApiClient.setDegradeRuleOfMachine(app, ip, port, rules);
try {
    rulePublisher.publish(app, rules);
    return true;
} catch(Exception ex) {
    logger.error("Publish degrade rules failed", ex);
    return false;
}
```
到这里，我们就完成了对后端程序的改造，将 Sentinel 限流规则和降级规则同步到了 Nacos。接下来我们需要对前端页面稍加修改，开放一个独立的页面，用来维护那些被同步到 Nacos 上的限流规则。

### 3.3 后端程序对接 Nacos
前端页面改造
首先，我们打开 sentinel-dashboard 模块下的 webapp 目录，该目录存放了 Sentinel 控制台的前端页面资源。我们需要改造的文件是 sidebar.html，这个 html 文件定义了控制台的左侧导航栏。

接下来，我们在导航列表中加入下面这段代码，增加一个导航选项，这个选项指向一个全新的限流页面。

sentinel-dashboard\src\main\webapp\resources\app\scripts\directives\sidebar\sidebar.html
```xml
<li ui-sref-active="active" ng-if="!entry.isGateway">
    <a ui-sref="dashboard.flow({app: entry.app})">
    <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则</a>
</li>
```
sentinel-dashboard\src\main\webapp\resources\app\scripts\controllers\identity.js

```js
app.controller('IdentityCtl', ['$scope', '$stateParams', 'IdentityService',
    // 此处 FlowServiceV1 需要替换为 FlowServiceV2
    //'ngDialog', 'FlowServiceV1', 'DegradeService', 'AuthorityRuleService', 'ParamFlowService', 'MachineService',
    'ngDialog', 'FlowServiceV2', 'DegradeService', 'AuthorityRuleService', 'ParamFlowService', 'MachineService',
  '$interval', '$location', '$timeout',
  function ($scope, $stateParams, IdentityService, ngDialog,
    FlowService, DegradeService, AuthorityRuleService, ParamFlowService, MachineService, $interval, $location, $timeout) {

    $scope.app = $stateParams.app;
```
---
title: "通过Maven检查三方库依赖的许可"
date: 2019-04-07T17:18:08+08:00
lastmod: 2019-04-07T17:18:08+08:00
draft: false
tags: ["license", "maven"]
categories: ["Java"]
description: ""
comment: true
toc: false
autoCollapseToc: false
contentCopyright: '<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a><a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>'
---

### 背景

最近Dubbo的依赖的某三方库发现使用了LGPL的依赖，由于Dubbo在Apache基金会里面进行孵化，需要确保所有的三方依赖都和Apache许可证进行兼容。也就是说任何LGPL，GPL许可的三方依赖理论上都不能被纳入Dubbo的依赖，无论是直接依赖还是传递依赖。

### 解决方案

为了解决这个问题，maven的license插件提供了许可证查询机制，可以采用了如下命令

```
mvn license:add-third-party -Dlicense.useMissingFile
```

上述命令会自动扫描所有模块当中的依赖，包括直接和传递依赖，输出到每个模块的`target/generated-sources/license/THIRD-PARTY.txt`文件下，内容如下

```
Lists of 114 third-party dependencies.
     (ASF 2.0) Code Generation Library (cglib:cglib-nodep:2.2 - http://cglib.sourceforge.net/)
     (Eclipse Public License - v 1.0) (GNU Lesser General Public License) Logback Classic Module (ch.qos.logback:logback-classic:1.2.2 - http://logback.qos.ch/logback-classic)
     (Eclipse Public License - v 1.0) (GNU Lesser General Public License) Logback Core Module (ch.qos.logback:logback-core:1.2.2 - http://logback.qos.ch/logback-core)
     (Apache 2) fastjson (com.alibaba:fastjson:1.2.46 - https://github.com/alibaba/fastjson)
     (Apache License, Version 2.0) Hessian Lite(Alibaba embed version) (com.alibaba:hessian-lite:3.2.5 - https://github.com/dubbo/hessian-lite)
     (Apache License, Version 2.0) metrics-common (com.alibaba.middleware:metrics-common:2.0.1 - https://github.com/dubbo/metrics/metrics-common)
     (Apache License, Version 2.0) metrics-core-api (com.alibaba.middleware:metrics-core-api:2.0.1 - https://github.com/dubbo/metrics/metrics-core-api)
     (Apache License, Version 2.0) metrics-core-impl (com.alibaba.middleware:metrics-core-impl:2.0.1 - https://github.com/dubbo/metrics/metrics-core-impl)
     (Apache License, Version 2.0) nacos-api 1.0.0-RC3 (com.alibaba.nacos:nacos-api:1.0.0-RC3 - http://maven.apache.org)
     (Apache License, Version 2.0) nacos-client 1.0.0-RC3 (com.alibaba.nacos:nacos-client:1.0.0-RC3 - https://github.com/alibaba/nacos)
     (Apache License, Version 2.0) nacos-common 1.0.0-RC3 (com.alibaba.nacos:nacos-common:1.0.0-RC3 - http://maven.apache.org)
     (The Apache Software License, Version 2.0) java-util (com.cedarsoftware:java-util:1.9.0 - https://github.com/jdereg/java-util)
     (The Apache Software License, Version 2.0) json-io (com.cedarsoftware:json-io:2.5.1 - https://github.com/jdereg/json-io)
     ...
```

可以看到有一些依赖是暴露了双协议或者多协议，这种情况下可以选择兼容度最高的一种协议。例如`logback`作为一个典型的例子，就是暴露的是EPL+LGPL双协议，而EPL属于在Apache基金金下的`Category B`，即允许二进制形式的依赖，所以`logback`是允许作为三方依赖出现的。

为了排除这些双协议造成的干扰，采用如下命令进行过滤后，只剩下2条记录：

```
find . -name THIRD-PARTY.txt | xargs grep GPL | grep -v Apache | grep
-v MIT | grep -v CDDL
./dubbo-registry/dubbo-registry-nacos/target/generated-sources/license/THIRD-PARTY.txt:
(GNU Lesser General Public License (LGPL), Version 2.1) Jackson
(org.codehaus.jackson:jackson-core-lgpl:1.9.6 -
http://jackson.codehaus.org)
./dubbo-registry/dubbo-registry-nacos/target/generated-sources/license/THIRD-PARTY.txt:
(GNU Lesser General Public License (LGPL), Version 2.1) Data
Mapper for Jackson (org.codehaus.jackson:jackson-mapper-lgpl:1.9.6 -
http://jackson.codehaus.org)
```

上述的依赖需要进行清理，nacos-client-1.0.0-RC3开始修复了这个问题。

但是，这里有一个坑，就是三方依赖的`optional`依赖没法通过上述命令检测出来，例如nacos-client的如下依赖也是LGPL，用mvn license插件无法检查出来。

```xml
<dependency>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-annotations</artifactId>
  <optional>true</optional>
</dependency>
```

事实上maven license插件提供了一个`<includeOptional>`[选项](https://www.mojohaus.org/license-maven-plugin/add-third-party-mojo.html)，但是实际上对于三方依赖的optional依赖却不生效，原因待确定。个人推测如果在nacos-client下运行上述命令，应该是可以检查出来的。

### 总结

后续Dubbo每新增一个三方依赖，都需要检查License的兼容情况，尤其是三方依赖的optional依赖需要仔细检查，目前没有特别好的办法，如果有想到好办法欢迎留言。
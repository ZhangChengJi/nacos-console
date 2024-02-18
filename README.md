# nacos 2.3.1-SNAPSHOT 源码springboot方式启动(详细)附改造工程地址

![image-20240218164739073](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218164739073.png)

文章时间是2024-2-18日，nacos默认`develop`分支，最新版是`2.3.1-SNAPSHOT`版本。

![image-20240218165040670](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218165040670.png)

我们这里就以nacos最新版进行改造成springboot启动方式。

## 1. Clone 代码

nacos github地址：https://github.com/alibaba/nacos.git

根据上面git地址把源码克隆到本地然后用idea 打开。

注意nacos java 使用版本是1.8需要在idea设置对应版本

![image-20240218165406417](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218165406417.png)

## 2. 全局编译打包

然后执行maven编译打包，跳过test检查

![image-20240218165910272](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218165910272.png)

全部打包完成

![image-20240218170143048](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218170143048.png)

## 3. 在console子项目添加jar

在console 子项目的src/main/resources/下面添加一个`lib`文件夹

![image-20240218170511461](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218170511461.png)

然后需要把每个子项目的里面的target文件下的  xxxx-2.3.1-SNAPSHOT.jar 复制到这个lib文件夹里面。如下

![image-20240218170859185](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218170859185.png)

## 4.修改application配置文件

1. 把` application.properties`改成` application.yml`，然后复制下面配置文件到yml
2. 在com.alibaba.nacos.console.config.ConsoleConfig.java文件里也需要改成@PropertySource("/application.yml")

> 注意数据库配置要改成自己的

```yml
server:
  port: 8848
  tomcat:
    basedir: logs
  servlet:
    context-path: /nacos

db:
  num: 1
  user: ${MYSQL_USER:root}
  password: ${MYSQL_PWD:password}
  url:
    0: jdbc:mysql://${MYSQL_HOST:127.0.0.1}:${MYSQL_PORT:3306}/${MYSQL_DB:nacos}?characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=false&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=GMT%2B8&nullCatalogMeansCurrent=true&allowPublicKeyRetrieval=true
  pool:
    config:
      connectionTimeout: 30000
      validationTimeout: 10000

nacos:
  core:
    auth:
      server:
        identity:
          key: serverIdentity
          value: security
      plugin.nacos.token.secret.key: SecretKey012345678901234567890123456789012345678901234567890123456789
      enabled: true
      system.type: nacos
  security:
    ignore:
      urls: /,/error,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/v1/auth/**,/v1/console/health/**,/actuator/**,/v1/console/server/**

spring:
  datasource:
    platform: mysql  #这个过期属性不能修改，nacos 代码对此有硬编码
  security:
    enabled: true
  boot:  # 接入 spring boot admin
    admin:
      client:
        url: http://locahost:5001
        username: xueyi
        password: xueyi
        instance:
          service-host-type: ip
  application:
    name: nacos
  main:
    allow-circular-references: true

useAddressServer: true

management:
  endpoints:
    web:
      exposure:
        include: '*'
  metrics:
    export:
      influx:
        enabled: false
      elastic:
        enabled: false
```

## 5. 初始化sql脚本

在config子项目的src/main/resources文件夹下找到 `mysql-schema.sql`文件。

![image-20240218171849018](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218171849018.png)

需要先创建数据库在执行sql脚本，这里已经都整理一个文件里面了可以直接复制使用。

```sql
DROP DATABASE IF EXISTS `nacos`;

CREATE DATABASE  `nacos` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

USE `nacos`;
/******************************************/
/*   表名称 = config_info                  */
/******************************************/
CREATE TABLE `config_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) DEFAULT NULL COMMENT 'group_id',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `c_desc` varchar(256) DEFAULT NULL COMMENT 'configuration description',
  `c_use` varchar(64) DEFAULT NULL COMMENT 'configuration usage',
  `effect` varchar(64) DEFAULT NULL COMMENT '配置生效的描述',
  `type` varchar(64) DEFAULT NULL COMMENT '配置的类型',
  `c_schema` text COMMENT '配置的模式',
  `encrypted_data_key` text NOT NULL COMMENT '密钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';

/******************************************/
/*   表名称 = config_info_aggr             */
/******************************************/
CREATE TABLE `config_info_aggr` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
  `content` longtext NOT NULL COMMENT '内容',
  `gmt_modified` datetime NOT NULL COMMENT '修改时间',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';


/******************************************/
/*   表名称 = config_info_beta             */
/******************************************/
CREATE TABLE `config_info_beta` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '密钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';

/******************************************/
/*   表名称 = config_info_tag              */
/******************************************/
CREATE TABLE `config_info_tag` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';

/******************************************/
/*   表名称 = config_tags_relation         */
/******************************************/
CREATE TABLE `config_tags_relation` (
  `id` bigint(20) NOT NULL COMMENT 'id',
  `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
  `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `nid` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'nid, 自增长标识',
  PRIMARY KEY (`nid`),
  UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';

/******************************************/
/*   表名称 = group_capacity               */
/******************************************/
CREATE TABLE `group_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';

/******************************************/
/*   表名称 = his_config_info              */
/******************************************/
CREATE TABLE `his_config_info` (
  `id` bigint(20) unsigned NOT NULL COMMENT 'id',
  `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT 'nid, 自增标识',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  `op_type` char(10) DEFAULT NULL COMMENT 'operation type',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '密钥',
  PRIMARY KEY (`nid`),
  KEY `idx_gmt_create` (`gmt_create`),
  KEY `idx_gmt_modified` (`gmt_modified`),
  KEY `idx_did` (`data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';


/******************************************/
/*   表名称 = tenant_capacity              */
/******************************************/
CREATE TABLE `tenant_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';


CREATE TABLE `tenant_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `kp` varchar(128) NOT NULL COMMENT 'kp',
  `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
  `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
  `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
  `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
  `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';

CREATE TABLE `users` (
	`username` varchar(50) NOT NULL PRIMARY KEY COMMENT 'username',
	`password` varchar(500) NOT NULL COMMENT 'password',
	`enabled` boolean NOT NULL COMMENT 'enabled'
);

CREATE TABLE `roles` (
	`username` varchar(50) NOT NULL COMMENT 'username',
	`role` varchar(50) NOT NULL COMMENT 'role',
	UNIQUE INDEX `idx_user_role` (`username` ASC, `role` ASC) USING BTREE
);

CREATE TABLE `permissions` (
    `role` varchar(50) NOT NULL COMMENT 'role',
    `resource` varchar(128) NOT NULL COMMENT 'resource',
    `action` varchar(8) NOT NULL COMMENT 'action',
    UNIQUE INDEX `uk_role_permission` (`role`,`resource`,`action`) USING BTREE
);

INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);

INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
SET FOREIGN_KEY_CHECKS = 1;
```

## 7. 复制 console变成一个独立项目，修改pom.xml

需要把console 子项目变成一个独立的工程。我们把这个console复制出来到你电脑的任意一个位置即可。

然后用idea 打开。

![image-20240218173835362](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218173835362.png)

下面进行修改console工程的pom.xml

1. 首先要修改的就是parent基础依赖，因为我这里没有依赖其他项目就直接写spring-boot的基础依赖了，如果有同学需要嵌入到自己主要项目中就需要写父工程的parent

```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
```

然后删除pom其他依赖，复制下面包依赖到pom.xml

```xml
<artifactId>nacos-console</artifactId>
    <packaging>jar</packaging>
    <name>nacos-console ${project.version}</name>
    <url>https://nacos.io</url>
    <properties>
<properties>
    <revision>2.3.1-SNAPSHOT</revision>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <!-- Compiler settings properties -->
    <java.version>1.8</java.version>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
    <!-- Maven properties -->
    <maven.test.skip>false</maven.test.skip>
    <maven.javadoc.skip>true</maven.javadoc.skip>
    <!-- Exclude all generated code -->
    <sonar.exclusions>file:**/generated-sources/**,**/test/**</sonar.exclusions>
    <!-- plugin version -->
    <versions-maven-plugin.version>2.2</versions-maven-plugin.version>
    <dependency-mediator-maven-plugin.version>1.0.2</dependency-mediator-maven-plugin.version>
    <clirr-maven-plugin.version>2.7</clirr-maven-plugin.version>
    <maven-enforcer-plugin.version>1.4.1</maven-enforcer-plugin.version>
    <maven-compiler-plugin.version>3.5.1</maven-compiler-plugin.version>
    <maven-javadoc-plugin.version>2.10.4</maven-javadoc-plugin.version>
    <maven-jar-plugin.version>3.2.2</maven-jar-plugin.version>
    <maven-source-plugin.version>3.0.1</maven-source-plugin.version>
    <maven-pmd-plugin.version>3.8</maven-pmd-plugin.version>
    <apache-rat-plugin.version>0.12</apache-rat-plugin.version>
    <maven-resources-plugin.version>3.0.2</maven-resources-plugin.version>
    <jacoco-maven-plugin.version>0.8.7</jacoco-maven-plugin.version>
    <maven-surefire-plugin.version>2.20</maven-surefire-plugin.version>
    <findbugs-maven-plugin.version>3.0.4</findbugs-maven-plugin.version>
    <sonar-maven-plugin.version>3.0.2</sonar-maven-plugin.version>
    <maven-gpg-plugin.version>1.6</maven-gpg-plugin.version>
    <maven-failsafe-plugin.version>2.19.1</maven-failsafe-plugin.version>
    <maven-assembly-plugin.version>3.0.0</maven-assembly-plugin.version>
    <maven-checkstyle-plugin.version>3.1.2</maven-checkstyle-plugin.version>
    <maven-easyj-version>1.1.5</maven-easyj-version>
    <!-- dependency version related to plugin -->
    <extra-enforcer-rules.version>1.0-beta-4</extra-enforcer-rules.version>
    <p3c-pmd.version>1.3.0</p3c-pmd.version>

    <!-- dependency version -->
    <spring-boot-dependencies.version>2.7.18</spring-boot-dependencies.version>
    <spring-boot-admin.version>2.7.11</spring-boot-admin.version>

    <servlet-api.version>3.0</servlet-api.version>
    <commons-io.version>2.7</commons-io.version>
    <commons-collections.version>3.2.2</commons-collections.version>
    <slf4j-api.version>1.7.26</slf4j-api.version>
    <logback.version>1.2.13</logback.version>
    <log4j.version>2.17.1</log4j.version>

    <mysql-connector-java.version>8.0.28</mysql-connector-java.version>
    <derby.version>10.14.2.0</derby.version>
    <jjwt.version>0.11.2</jjwt.version>
    <javatuples.version>1.2</javatuples.version>
    <grpc-java.version>1.57.2</grpc-java.version>
    <proto-google-common-protos.version>2.17.0</proto-google-common-protos.version>
    <protobuf-java.version>3.22.3</protobuf-java.version>
    <protoc-gen-grpc-java.version>${grpc-java.version}</protoc-gen-grpc-java.version>
    <hessian.version>4.0.63</hessian.version>
    <mockito-all.version>1.10.19</mockito-all.version>
    <mockito-core.version>3.8.0</mockito-core.version>
    <HikariCP.version>3.4.2</HikariCP.version>
    <jraft-core.version>1.3.12</jraft-core.version>
    <rpc-grpc-impl.version>${jraft-core.version}</rpc-grpc-impl.version>
    <SnakeYaml.version>2.0</SnakeYaml.version>
    <nacos.lib.path>${project.basedir}/src/main/resources/lib</nacos.lib.path>
    </properties>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <!-- Import dependency management from Spring Boot -->
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot-dependencies.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <distributionManagement>
        <snapshotRepository>
            <!-- The ID here must be exactly the same as the value
             of the server element id in the settings.xml file of MAVEN -->
            <id>sona</id>
            <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
        </snapshotRepository>
        <repository>
            <id>sona</id>
            <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
    </distributionManagement>
    <dependencies>
      <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot-dependencies.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-api</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-api-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-auth</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-auth-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-auth-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-auth-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-cmdb</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-cmdb-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-config</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-config-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-persistence</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-persistence-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-consistency</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-consistency-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-control-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-control-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-config-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-config-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-core</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-core-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-istio</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-istio-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-naming</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-naming-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>default-auth-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/default-auth-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>default-control-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/default-control-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-trace-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-trace-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-prometheus</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-prometheus-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-sys</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-sys-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-datasource-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-datasource-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-encryption-plugin</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-encryption-plugin-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-common</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-common-${revision}.jar</systemPath>
        </dependency>
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>${revision}</version>
            <scope>system</scope>
            <systemPath>${nacos.lib.path}/nacos-client-${revision}.jar</systemPath>
        </dependency>
        <!-- SpringBoot Web容器 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                    <groupId>org.springframework.boot</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>log4j-to-slf4j</artifactId>
                    <groupId>org.apache.logging.log4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- web 容器使用 undertow 性能更强 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ldap</groupId>
            <artifactId>spring-ldap-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpcore</artifactId>
            <version>4.4.16</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.14</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpcore-nio</artifactId>
            <version>4.4.16</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpasyncclient</artifactId>
            <version>4.1.5</version>
        </dependency>
        <dependency>
            <groupId>com.caucho</groupId>
            <artifactId>hessian</artifactId>
        </dependency>
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-core</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.derby</groupId>
            <artifactId>derby</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alipay.sofa</groupId>
            <artifactId>jraft-core</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alipay.sofa</groupId>
            <artifactId>rpc-grpc-impl</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>31.1-jre</version>
        </dependency>
        <dependency>
            <groupId>org.javatuples</groupId>
            <artifactId>javatuples</artifactId>
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-influx</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-elastic</artifactId>
        </dependency>


        <dependency>
            <groupId>io.envoyproxy.controlplane</groupId>
            <artifactId>api</artifactId>
            <version>0.1.27</version>
        </dependency>

        <!-- log -->
        <!-- apache commons logging通过slf4j来代理 -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
        </dependency>
        <!-- java.util.logging 通过slf4j来代理 -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jul-to-slf4j</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-client</artifactId>
            <version>${spring-boot-admin.version}</version>
        </dependency>


        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>servlet-api</artifactId>
            <version>3.0-alpha-1</version>
            <scope>provided</scope>
        </dependency>
        <!--        <dependency>-->
        <!--            <groupId>com.zaxxer</groupId>-->
        <!--            <artifactId>HikariCP</artifactId>-->
        <!--            <version>3.4.2</version>-->
        <!--        </dependency>-->
        <dependency>
            <groupId>com.caucho</groupId>
            <artifactId>hessian</artifactId>
            <version>4.0.63</version>
        </dependency>
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.7</version>
        </dependency>
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.2</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.28</version>
        </dependency>
        <!--        <dependency>-->
        <!--            <groupId>org.apache.derby</groupId>-->
        <!--            <artifactId>derby</artifactId>-->
        <!--            <version>10.14.2.0</version>-->
        <!--        </dependency>-->
        <dependency>
            <groupId>com.alipay.sofa</groupId>
            <artifactId>jraft-core</artifactId>
            <version>1.3.12</version>
            <exclusions>
                <exclusion>
                    <groupId>com.alipay.sofa</groupId>
                    <artifactId>bolt</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-core</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-slf4j-impl</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-jcl</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.alipay.sofa</groupId>
            <artifactId>rpc-grpc-impl</artifactId>
            <version>1.3.12</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.11.2</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.11.2</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.11.2</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>javax.annotation</groupId>
            <artifactId>javax.annotation-api</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.javatuples</groupId>
            <artifactId>javatuples</artifactId>
            <version>1.2</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>1.57.2</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>1.57.2</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>1.57.2</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-testing</artifactId>
            <version>1.57.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.google.api.grpc</groupId>
            <artifactId>proto-google-common-protos</artifactId>
            <version>2.17.0</version>
        </dependency>
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.22.3</version>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-all</artifactId>
            <version>1.10.19</version>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>3.8.0</version>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-inline</artifactId>
            <version>3.8.0</version>
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.9.0</version>
        </dependency>
        <dependency>
            <groupId>org.yaml</groupId>
            <artifactId>snakeyaml</artifactId>
            <version>2.0</version>
        </dependency>
    </dependencies>
    <build>
        <finalName>${project.artifactId}</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
<!--                <version>${spring-boot.version}</version>-->
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <!-- 作用:项目打成jar的同时将本地jar包也引入进去 -->
                    <includeSystemScope>true</includeSystemScope>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

## 8. 设置单机模式运行，添加 banner.txt

在Nacos.java 启动类型添加单机模式启动，关闭tomcat启动使用undertow

```java
@SpringBootApplication
@ComponentScan(basePackages = "com.alibaba.nacos", excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = {NacosTypeExcludeFilter.class}),
        @Filter(type = FilterType.CUSTOM, classes = {TypeExcludeFilter.class}),
        @Filter(type = FilterType.CUSTOM, classes = {AutoConfigurationExcludeFilter.class})})
@ServletComponentScan
@EnableScheduling
public class Nacos {
   
    public static void main(String[] args) {
        // true 单机模式 false 为集群模式 集群模式需搭配 cluster.conf 使用 使用方法请查看文档
        System.setProperty("nacos.standalone", "true");
        System.setProperty("server.tomcat.accesslog.enabled", "false");
        SpringApplication.run(Nacos.class, args);
    }
}
```

删除test测试目录src/test。

在src/main/resources文件夹下添加banner.txt。

```
         ,--.
       ,--.'|
   ,--,:  : |                                           Nacos ${application.version}
,`--.'`|  ' :                       ,---.               Running in ${nacos.mode} mode, ${nacos.function.mode} function modules
|   :  :  | |                      '   ,'\   .--.--.    Port: ${server.port}
:   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: ${pid}
|   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://${nacos.local.ip}:${server.port}${server.servlet.context-path}/index.html
'   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_
|   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io
'   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \
|   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /
'   : |     ;  :   .'   \   :    : `----'  '--'.     /
;   |.'     |  ,     .-./\   \  /            `--'---'
'---'        `--`---'     `----'

```



刷新maven 启动

![image-20240218181134210](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218181134210.png)

界面：

访问地址：[http://localhost:8848/nacos/index.html#/login](http://localhost:8848/nacos/index.html#/login)

账号/密码：nacos/nacos

![image-20240218183804209](https://gitee.com/zhangchengji/pic/raw/master/2024/image-20240218183804209.png)

附修改后工程地址：https://github.com/ZhangChengJi/nacos-console.git
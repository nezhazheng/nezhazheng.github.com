---
layout: post
title:  "Java世界最棒的DB Migration Tool"
---

[liquibase](http://www.liquibase.org/)
---------------------

### Why Database Migration Tool？

我觉得在项目中使用数据库迁移工具的主要原因有二：

1. 项目部署了多套环境（Dev,Production,Test...），那么一个好用的数据库迁移工具会直接提升项目团队的生产效率。并且保证多套环境数据库的一致。

2. 当选择相应的软件版本时，能快速的生成此软件版本对应的数据库。

### Why liquibase?

事实上还有一个不错的数据库迁移工具[Flyway](http://flywaydb.org/)，但是不知道为什么flyway并没有提供配置文件式的变更记录方式，而是只提供了Java API与SQL文件的方式作为数据库变更的记录方式，个人认为Java不同于Ruby，编译型语言采用这种原生代码的形式来做配置其实是比较痛苦的。所以相比较下，liquibase提供的多种配置文件式(json,yaml,xml..)的对非DBA开发人员来说显得更友好一些。

还有一个原因Flyway的官网UI虽然做的很棒，但是实质性的参考文档确是偷工减料了许多，liquibase在文档这一块显得要更用心一些。

### How

* Maven pom 中的配置，configuration中得可用配置项可以直接参考[官网文档](http://www.liquibase.org/documentation/maven/)中得解释。

{% highlight xml %}
<build>
	<plugins>
		<plugin>
			<groupId>org.liquibase</groupId>
			<artifactId>liquibase-maven-plugin</artifactId>
			<version>3.0.5</version>
			<configuration>
				<changeLogFile>src/main/resources/db.xml</changeLogFile>
				<driver>${database.driverClassName}</driver>
				<url>${database.url}</url>
				<username>${database.username}</username>
				<password>${database.password}</password>
				<promptOnNonLocalDatabase>false</promptOnNonLocalDatabase>
		        <outputChangeLogFile>db.xml</outputChangeLogFile>
			</configuration>
			<dependencies>
				<dependency>
					<groupId>mysql</groupId>
					<artifactId>mysql-connector-java</artifactId>
					<version>5.1.18</version>
				</dependency>
			</dependencies>
			<executions>
				<execution>
					<phase>process-resources</phase>
					<goals>
						<goal>update</goal>
					</goals>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
{% endhighlight %}


* ChangeLogFile ChangeSet中得可选配置可以参考，原生sql什么的都是支持的[官网文档链接](http://www.liquibase.org/documentation/changes/index.html)

{% highlight xml %}
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.0.xsd">
    <changeSet author="nezhazheng (generated)" id="1388030352607-1" runOnChange="true">
        <createTable catalogName="test_community_helper" tableName="community">
            <column autoIncrement="true" name="id" type="INT(10)">
                <constraints primaryKey="true"/>
            </column>
            <column name="name" type="VARCHAR(255)"/>
        </createTable>
    </changeSet>
    <changeSet author="nezhazheng (generated)" id="1388030352607-2" runOnChange="true">
        <createTable catalogName="test_community_helper" tableName="image">
            <column autoIncrement="true" name="id" type="INT(10)">
                <constraints primaryKey="true"/>
            </column>
            <column name="platform" type="VARCHAR(255)"/>
            <column name="type" type="VARCHAR(255)"/>
            <column name="url" type="VARCHAR(255)"/>
        </createTable>
    </changeSet>
</databaseChangeLog>
{% endhighlight %}

* 常用maven命令

对已有数据库生成对应的ChangeLogFile

`mvn liquibase:generateChangeLog -Dliquibase.outputChangeLogFile=db.xml`

把ChangeLog中得记录同步到数据库，在完成上面一条生成ChangeLogFile的命令后，需要紧接着运行下面这条命令，在数据库同步ChangeLogFile

`mvn liquibase:changelogSync`

类似于Source Control中的Tag，用于方便的切换版本。

`mvn liquibase:tag -Dliquibase.tag=0.1`

回滚数据库，可选参数:

rollbackCount: 回滚changeset的数量
rollbackTag: 回滚到相应的Tag位置

`mvn -Dliquibase.rollbackCount=2 liquibase:rollback`

更新数据库

`mvn liquibase:update`

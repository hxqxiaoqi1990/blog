---
title: solr安装
tags: [solr]
categories: 中间件
date: 2019-08-11 18:00:00
---


# 介绍
Solr是一个独立的企业级搜索应用服务器，它对外提供类似于Web-service的API接口。用户可以通过http请求，向搜索引擎服务器提交一定格式的XML文件，生成索引；也可以通过Http Get操作提出查找请求，并得到XML格式的返回结果。

Solr是一个高性能，采用Java开发，基于Lucene的全文搜索服务器。同时对其进行了扩展，提供了比Lucene更为丰富的查询语言，同时实现了可配置、可扩展并对查询性能进行了优化，并且提供了一个完善的功能管理界面，是一款非常优秀的全文搜索引擎。

# 安装步骤
Solr的安装必须先 [安装JDK](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85-linux/)，这个的安装这里不再赘述，下面将要介绍的是solr的安装流程。

 1. 下载tomcat和solr，并解压
 2. 复制solr文件到tomcat
 3. 启动并访问solr
 4. 创建核心
 5. 下载并配置中文分词

## 下载tomcat和solr，并解压
``` bash
# 安装目录
cd /data

# 下载
wget http://mirror.bit.edu.cn/apache/tomcat/tomcat-8/v8.5.45/bin/apache-tomcat-8.5.45.tar.gz
wget http://archive.apache.org/dist/lucene/solr/7.1.0/solr-7.1.0.tgz

tar xf apache-tomcat-8.5.45.tar.gz
mv  apache-tomcat-8.5.45 tomcat-solr

tar xf solr-7.1.0.tgz
```

## 复制solr文件到tomcat

``` bash
cd /data
cp -a solr-7.1.0/server/solr-webapp/webapp tomcat-solr/webapps/solr
cp -a solr-7.1.0/server/lib/ext/* tomcat-solr/webapps/solr/WEB_INF/lib
cp -a solr-7.1.0/server/lib/metrics* tomcat-solr/webapps/solr/WEB_INF/lib
cp -a solr-7.1.0/dist/solr-dataimporthandler* tomcat-solr/webapps/solr/WEB_INF/lib

cd tomcat-solr/webapps/solr/WEB-INF/
mkdir classes
cd classes
cp -a solr-7.1.0/server/resources/log4j.properties .

cd /data
mkdir solrhome
cp -a solr-7.1.0/server/solr/* solrhome
cp -a solr-7.1.0/dist/ solrhome/
cp -a solr-7.1.0/contrib/ solrhome/

vim tomcat-solr/webapps/solr/WEB_INF/web.xml
	# 在web-app节点下添加，然后把security-constraint相关的都注释掉
    <env-entry>
       <env-entry-name>solr/home</env-entry-name>
       <env-entry-value>/data/solrhome</env-entry-value>
       <env-entry-type>java.lang.String</env-entry-type>
    </env-entry>
```

## 启动并访问solr

``` bash
# 启动
/data/tomcat-solr/bin/startup.sh

# 访问
Ip:8080/solr/index.html
```
## 创建核心

``` bash
cd /data/solrhome
mkdir -p testCore/conf

cd testCore/conf
cp -a /data/solr-7.1.0/server/solr/configsets/_default/conf/* .

vim solrconfig.xml
# 添加以下配置，并把原来的<lib dir= 也改为当前路径
<lib dir="/data/solrhome/contrib/extraction/lib" regex=".*\.jar" />
    <lib dir="/data/solrhome/dist/" regex="solr-cell-\d.*\.jar" />
    <lib dir="/data/solrhome/contrib/clustering/lib/" regex=".*\.jar" />
    <lib dir="/data/solrhome/dist/" regex="solr-clustering-\d.*\.jar" />
    <lib dir="/data/solrhome/contrib/langid/lib/" regex=".*\.jar" />
    <lib dir="/data/solrhome/dist/" regex="solr-langid-\d.*\.jar" />
    <lib dir="/data/solrhome/contrib/velocity/lib" regex=".*\.jar" />
<lib dir="/data/solrhome/dist/" regex="solr-velocity-\d.*\.jar" />

cd /data/solrhome/testCore
mkdir data

vim core.properties
# 写入以下内容
name=testCore
config=conf/solrconfig.xml
schema=conf/managed-schema
dataDir=data
```

完成以上步骤重启tomcat就可以了

## 分词器安装

``` bash
# github地址
https://github.com/magese/ik-analyzer-solr/tree/v7.5.0

# 下载
wget https://search.maven.org/remotecontent?filepath=com/github/magese/ik-analyzer/7.5.0/ik-analyzer-7.5.0.jar

cp -a ik-analyzer-7.5.0.jar tomcat-solr/webapps/solr/WEB-INF/lib

vim solrhome/testCore/conf/managed-schema
# 添加以下配置
<fieldType name="text_ik" class="solr.TextField" positionIncrementGap="100">  
         	<analyzer type="index">    
            		<tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory" useSmart="true" />   
         	</analyzer>    
         	<analyzer type="query">    
             		<tokenizer class="org.wltea.analyzer.lucene.IKTokenizerFactory" useSmart="true" />   
         	</analyzer>    
</fieldType>  

<field name="text_ik" type="text_ik" indexed="true" stored="true" multiValued="false" />
```
配置完成后，重启tomcat就可以使用分词功能了

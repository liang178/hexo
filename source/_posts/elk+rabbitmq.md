---
title: "ELK+RabbitMQ架构处理nginx及tomcat日志"
date: 2016-08-11
categories: ELK
tags: [ elk,logstash,elasticsearch,kibana,rabbitmq ]
---

## 前言
查看日志的传统方法是：登录操作系统，使用命令工具如cat、tail、sed、awk、grep等等进行过滤输出后分析，处理少量日志还好，日志量大处理效率就没那么高了。而且很多情况下开发人员需要查看并分析日志进行排错，但他们对Linux命令又不是太熟悉，而且有时候又不能赋予他们服务器权限，更多时候是运维把日志文件导出来发给开发人员，这无疑会给我们增加工作量。ELK（Elasticsearch+Logstash+Kibana）架构就是专门为采集、分析、存储日志所设计的：

Elasticsearch：基于Lucenne的搜索服务器，提供一个分布式多用户的全文搜索引擎，能过做到实时搜索。

Logstash：可以对日志进行采集、过滤、输出。

Kibana：可以汇总、分析、搜索日志数据并提供友好的web界面。

工作流程：logstash agent监控并过滤日志，为了保证日志的完整性先将日志内容输出到RabbitMQ进行存储；logstash indexer再把RabbitMQ上的日志队列收集后发送给全文搜索服务器Elasticsearch，然后可以用Elasticsearch进行自定义搜索，再通过Kibana来结合自定义搜索进行页面展示。
## ELK架构图
![](http://obbrvy8yl.bkt.clouddn.com/20160803layout.png)

从官网下载软件logstash、elasticsearch、kibana以及JRE，分别在相应主机上安装，logstash和Elasticsearch依赖于JRE

    wget https://download.elastic.co/kibana/kibana/kibana-4.5.3-1.x86_64.rpm
    wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.3.4/elasticsearch-2.3.4.rpm
    wget https://download.elastic.co/logstash/logstash/packages/centos/logstash-2.3.4-1.noarch.rpm
    wget http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jre-8u101-linux-x64.rpm
## RabbitMQ配置

安装RabbitMQ

    yum install rabbitmq-server

启用RabbitMQ的web管理功能

    /usr/lib/rabbitmq/bin/rabbitmq-plugins enable rabbitmq_management
    /usr/lib/rabbitmq/bin/rabbitmq-plugins list

下载并安装命令管理工具rabbitmqadmin

    wget http://rabbitmq-server:15672/cli/rabbitmqadmin
    mv rabbitmqadmin /usr/local/bin
    chmod +x /usr/local/bin/rabbitmqadmin

给rabbitmqadmin工具准备配置文件

    # vim /etc/mqadmin.conf
    [default]
    hostname = localhost
    port = 55672
    username = liang
    password = liang123

创建一个vhost和user并赋权

    rabbitmqctl add_user liang liang123
    rabbitmqctl add_vhost elk
    rabbitmqctl set_permissions -p elk liang ".*" ".*" ".*"
    rabbitmqctl set_user_tags liang administrator
    rabbitmqctl list_permissions -p elk

创建一个exchange
   
    rabbitmqadmin -c /etc/mqadmin.conf declare exchange --vhost=elk name=elk_exchange type=direct

创建一个queue

    rabbitmqadmin -c /etc/mqadmin.conf declare queue --vhost=elk name=elk_queue durable=true

创建一个binding，绑定之前创建的exchange和queue并设置一个routing_key


    rabbitmqadmin -c /etc/mqadmin.conf --vhost=elk declare binding source="elk_exchange" destination="elk_queue" routing_key="elk_key"

以上关于RabbitMQ的配置均可以通过登录web控制台进行操作，更简单方便，这里就不再演示了。
## Elasticsearch配置

给elasticsearch安装shield插件，用于权限控制，此插件是收费产品，可免费使用30天，到期后可降级使用，只是关于集群的一些功能将不可用。

    cd /usr/share/elasticsearch
    bin/plugin install license
    bin/plugin install shield

如有需要修改elasticsearch.yml，配置服务监听地址，默认监听在127.0.0.1上，端口是9200和9300；配置日志数据的存储路径，默认保存在/var/lib/elasticsearch下

    path.data: /data/elastic_data
    network.host: 192.168.X.X

配置shield，修改shield的权限控制文件roles.yml，修改默认角色logstash的权限，指定可创建的索引文件；并添加一个readonly的角色dashboard，用于控制用户在kibana上的权限。


    logstash:
      cluster:
    	- manage_index_templates
      indices:
    	- names: [ 'logstash-*','nginx-*','tomcat-*' ]
      	  privileges:
    		- write
    		- delete
    		- create_index
    
    dashboard:
      indices:
    'tomcat-*':
      - indices:admin/mappings/fields/get
      - indices:admin/validate/query
      - indices:data/read/search
      - indices:data/read/msearch
      - indices:data/read/field_stats
      - indices:admin/get
    '.kibana':
      - indices:admin/exists
      - indices:admin/mappings/fields/get
      - indices:admin/refresh
      - indices:admin/validate/query
      - indices:data/read/get
      - indices:data/read/mget
      - indices:data/read/search

配置shield，创建用户logstash、kibana、readuser、admin并指定相应角色

    cd /usr/share/elasticsearch/bin/shield
    ./esusers useradd logstash -p 123456 -r logstash
    ./esusers useradd kibana -p 123456 -r kibana4_server  
    ./esusers useradd readuser -p 123456 -r dashboard
    ./esusers useradd admin -p 123456 -r admin 

启动Elasticsearch


    service elasticsearch start


## Logstash配置

配置nginx服务器输出json格式日志


    log_format json '{"@timestamp":"$time_iso8601",'
       '"host":"$server_addr",'
       '"clientip":"$remote_addr",' '"size":$body_bytes_sent,'
       '"responsetime":$request_time,'
       '"upstreamtime":"$upstream_response_time",'
       '"upstreamhost":"$upstream_addr",'
       '"http_host":"$host",'
       '"url":"$uri",'
       '"xff":"$http_x_forwarded_for",'
       '"referer":"$http_referer",'
       '"agent":"$http_user_agent",'
       '"status":"$status"}';
    access_log  /usr/local/nginx/logs/api_json.log  json;


配置logstash agent采集nginx日志并输出到RabbitMQ；为了排错，同时输出一份日志到本地。



    # vim /etc/logstash/conf.d/ngx_log.conf
    input {
    file {
    	path => "/usr/local/nginx/logs/api_json.log"
    	codec => "json"
    	type => "nginx"
    }
    }
    
    output {
    rabbitmq { 
    	host => "RabbitMQ_server"
    	port => "5672"
    	vhost => "elk"
    	exchange => "elk_exchange"
    	exchange_type => "direct"
    	key => "elk_key"
    	user => "liang"
    	password => "liang123"
    	}
    stdout { codec => rubydebug }
    }

配置tomcat服务器输出json格式日志，修改工程的logback.xml配置文件，添加如下配置


	<appender name="LOGSTASH" class="ch.qos.logback.core.rolling.RollingFileAppender"> 
        <file>${catalina.base}/logs/tomcat_json.log</file>
	    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
	        <charset>utf8</charset>
	    </encoder> 
	    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
	        <fileNamePattern>${catalina.base}/logs/tomcat_json-%d{yyyy-MM-dd}.log</fileNamePattern>  
	    </rollingPolicy>  
	</appender>
    <root level="info">
        <appender-ref ref="LOGSTASH" />
    </root>

下载依赖的jar包logstash-logback-encoder到{CATALINA_BASE}/lib



    wget http://central.maven.org/maven2/net/logstash/logback/logstash-logback-encoder/4.4/logstash-logback-encoder-4.4.jar

配置logstash agent采集tomcat日志并输出到RabbitMQ



    # vim /etc/logstash/conf.d/tomcat_log.conf
    input {
    file {
    	path => "/usr/local/tomcat/logs/tomcat_json.log"
    	codec => "json"
    	type => "tomcat"
    }
    }
    
    output {
    rabbitmq { 
    	host => "RabbitMQ_server"
    	port => "5672"
    	vhost => "elk"
    	exchange => "elk_exchange"
    	exchange_type => "direct"
    	key => "elk_key"
    	user => "liang"
    	password => "liang123"
    	}
    stdout { codec => rubydebug }
    }

配置logstash indexer把日志从RabbitMQ输出到Elasticsearch


    # vim /etc/logstash/conf.d/rabbitmq.conf
    input {
    rabbitmq {
    	host => "127.0.0.1"
    	subscription_retry_interval_seconds => "5"
    	vhost => "elk"
    	exchange => "elk_exchange"
    	queue => "elk_queue"
    	durable => "true"
    	key => "elk_key"
    	user => "liang"
    	password => "liang123"
    	}
    }
    output {
    if [type] == "nginx" {
    	elasticsearch {
    		hosts => "Elasticsearch_server:9200"
    		user => "logstash"
    		password => "123456"
    		index => "nginx-%{+YYYY.MM.dd}"
    	}
    }
    else if [type] == "tomcat" {
    	elasticsearch {
    		hosts => "Elasticsearch_server:9200"
    		user => "logstash"
    		password => "123456"
    		index => "tomcat-%{+YYYY.MM.dd}"
    	}
    }
    else {
    	file {
    		path => "/var/log/logstash/unknown_messages.log"
    		} 
    	}
    stdout { codec => rubydebug }
    }


启动logstash服务

    service logstash start


在RabbitMQ服务器上查看是否接收到日志消息，登录RabbitMQ的web控制台查看详细信息。

    rabbitmqctl list_queues -p elk

## kibana配置

给kibana安装shield插件，用于权限控制

    cd /opt/kibana/bin
    ./kibana plugin --install kibana/shield/2.3.4

修改kibana的主配置文件kibana.yml，似乎启用权限控制后强制使用https，shield的加密key可以随便指定，会话超时时间默认是30分钟，超时时间的单位为毫秒


    elasticsearch.username: "kibana"
    elasticsearch.password: "123456"
    elasticsearch.url: "http://localhost:9200"
    server.ssl.cert: /opt/kibana/ssl/kibana.crt
    server.ssl.key: /opt/kibana/ssl/kibana.key
    shield.encryptionKey: "abc123"
    shield.sessionTimeout: 600000

启动kibana

    service kibana start

kibana启用后，就可以通过https://server:5601 进行访问了。如有需要配置Apache做个反向代理

    NameVirtualHost *:443
    <VirtualHost *:443>
    ProxyRequests on
    SSLEngine on
    SSLProxyEngine on
    SSLCertificateFile /opt/kibana/ssl/kibana.crt
    SSLCertificateKeyFile /opt/kibana/ssl/kibana.key
    ProxyPass / https://127.0.0.1:5601/
    ProxyPassReverse / https://127.0.0.1:5601/
    </VirtualHost>

登录界面。

![](http://obbrvy8yl.bkt.clouddn.com/20160803login.png)

填入之前定义的索引文件，就可以处理日志了。
![](http://obbrvy8yl.bkt.clouddn.com/20160803define.png)

看到kibana有收到日志就算成功了。
![](http://obbrvy8yl.bkt.clouddn.com/20160803status.png)

至此，整个架构部署完毕，如有错误或不足之处，欢迎指正。







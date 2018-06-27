# elk安装部署 #

elasticsearch 官网：<http://www.elastic.co>

1、安装jdk
   
    [root@elk ~]# yum install java -y
    [root@elk ~]# java -version
    openjdk version "1.8.0_171"
    OpenJDK Runtime Environment (build 1.8.0_171-b10)
    OpenJDK 64-Bit Server VM (build 25.171-b10, mixed mode)
2、安装elasticsearch

    [root@elk ~]# cd /usr/local/src
    [root@elk src]# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.4.tar.gz
    [root@elk src]# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.4.tar.gz.sha512
    [root@elk src]# yum install perl-Digest-SHA -y
    [root@elk src]#  shasum -a 512 -c elasticsearch-6.2.4.tar.gz.sha512
    [root@elk src]# tar zxf elasticsearch-6.2.4.tar.gz -C /opt/
    [root@elk src]# mv /opt/elasticsearch-6.2.4 /opt/elasticsearch
    [root@elk src]# useradd elasticsearch
修改elasticsearch配置

    [root@elk src]# cd /opt/elasticsearch/config/
    [root@elk config]# grep "^[a-z]" elasticsearch.yml 
    cluster.name: elk
    node.name: node-1
    path.data: /data/es-data/
    path.logs: /var/log/elasticsearch
    network.host: 192.168.6.8
    http.port: 9200
    [root@elk config]# mkdir /data/es-data/
    [root@elk config]# mkdir /var/log/elasticsearch
    [root@elk config]# chown -R elasticsearch:elasticsearch /opt/elasticsearch/
    [root@elk config]# chown -R elasticsearch:elasticsearch /data/es-data
    [root@elk config]# chown -R elasticsearch:elasticsearch /var/log/elasticsearch
修改内核参数

    [root@elk config]# vi /etc/sysctl.conf
    vm.max_map_count=655360
    [root@elk config]# sysctl -p
启动elasticsearch
 
    [root@elk config]# su - elasticsearch -c "/opt/elasticsearch/bin/elasticsearch -d -p pid"
    [root@elk elasticsearch]# ss -lntup|grep 9200
    tcp    LISTEN     0      128      ::ffff:192.168.6.8:9200                 :::*                   users:(("java",pid=11415,fd=171))
安装head插件

elasticsearch改版后，head安装方式有了很大的变化，具体如下：

5.0版本以前：

    /opt/elasticsearch/bin/plugin install mobz/elasticsearch-head
5.0版本以后：

安装node

由于head插件本质上还是一个nodejs的工程，因此需要安装node，使用npm来安装依赖的包，（npm可以理解为maven）

    [root@elk src]# wget https://nodejs.org/dist/latest/node-v10.3.0-linux-x64.tar.gz
    [root@elk src]# tar xf node-v10.3.0-linux-x64.tar.gz -C /opt/
    [root@elk src]# cd /opt
    [root@elk opt]# mv node-v10.3.0-linux-x64 node
    [root@elk opt]# echo "PATH=$PATH:/opt/node/bin:/opt/elasticsearch/bin" >>/etc/profile
    [root@elk opt]# source /etc/profile
    [root@elk opt]# node -v
    v10.3.0
    [root@elk opt]# npm -v
    6.1.0
或者在<https://nodejs.org/dist/>找到其它版本下载

安装grunt

grunt是一个很方便的构建工具，可以进行打包压缩、测试、执行等工作，5.0里的head插件就是通过grunt启动的。

    [root@elk opt]# npm install -g grunt-cli 
    [root@elk opt]# grunt --version
    grunt-cli v1.2.0
下载head源码

下载head包
在https://github.com/mobz/elasticsearch-head下下载head包
或者使用git下载 git clone git://github.com/mobz/elasticsearch-head.git

    [root@elk src]# tar zxf elasticsearch-head-5.0.0.tar.gz -C /opt/elasticsearch/  #注意，不要放在/opt/elasticsearch/plugins/下    
    [root@elk src]# cd /opt/elasticsearch/
    [root@elk elasticsearch]# mv elasticsearch-head-5.0.0 elasticsearch-head
    [root@elk elasticsearch]# cd elasticsearch-head/
修改源码，由于head的代码还是2.6版本的，直接执行有很多限制，比如无法跨机器访问
   
    [root@elk elasticsearch-head]# vi Gruntfile.js
    connect: {
                     server: {
                             options: {
                                     port: 9100,
                                     hostname: '0.0.0.0',
                                     base: '.',
                                     keepalive: true
                             }
                     }
             }
修改_site/app.js,
 
    [root@elk elasticsearch-head]# vi _site/app.js
    http://192.168.6.8:9200  #修改成本机IP，否则head无法连接es

修改elsearch的elasticsearch.yml，添加如下配置

    # 增加新的参数，这样head插件可以访问es
    http.cors.enabled: true
    http.cors.allow-origin: "*"

重启elasticsearch，然后在head目录，执行npm install，因为node默认使用的国外镜像，在未代理的情况下会比较慢，所以推荐重定向镜像，npm install -g cnpm --registry=https://registry.npm.taobao.org
 
    [root@elk elasticsearch-head]# npm install -g cnpm --registry=https://registry.npm.taobao.org
启动nodejs
 
    [root@elk elasticsearch-head]# nohup grunt server &exit
如果启动有问题，执行以下命令：
    
    [root@es-2 elasticsearch-head]# npm install grunt --save-dev
    [root@es-2 elasticsearch-head]# npm install grunt-contrib-clean@1.0.0 --save-dev
    [root@es-2 elasticsearch-head]# npm install grunt-contrib-concat@1.0.1 --save-dev
    [root@es-2 elasticsearch-head]# npm install grunt-contrib-connect@1.0.2 --save-dev
    [root@es-2 elasticsearch-head]# npm install grunt-contrib-copy@1.0.0 --save-dev
    [root@es-2 elasticsearch-head]# npm install grunt-contrib-watch@1.0.0 --save-dev
    
 
访问elasticsearch head：http://xxxx:9100

官档：<https://github.com/mobz/elasticsearch-head>

参考：

<https://blog.csdn.net/mergerly/article/details/53412417>

<https://blog.csdn.net/sulei12341/article/details/52935271>

<https://gruntjs.com/getting-started>

安装kibana

    [root@elk src]# wget https://artifacts.elastic.co/downloads/kibana/kibana-6.2.4-linux-x86_64.tar.gz
    [root@elk src]# tar xf kibana-6.2.4-linux-x86_64.tar.gz -C /opt/
    [root@elk src]# cd /opt/
    [root@elk opt]# ls
    elasticsearch  kibana-6.2.4-linux-x86_64  node
    [root@elk opt]# mv kibana-6.2.4-linux-x86_64 kibana
修改配置
    
    [root@elk opt]# cd kibana/config/
    [root@elk config]#  egrep -v "^#|^$" /opt/kibana/config/kibana.yml 
    server.port: 5601
    server.host: "192.168.6.8"
    elasticsearch.url: "http://192.168.6.8:9200"
    kibana.index: ".kibana"
    logging.dest: /var/log/kibana.log
    tilemap.url: 'http://webrd02.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}'
修改环境变量
    
    [root@elk config]# vi /etc/profile
    PATH=$PATH:/opt/node/bin:/opt/elasticsearch/bin:/opt/kibana/bin
    [root@elk config]# source /etc/profile 
启动kibana

    [root@elk elasticsearch-head]# /opt/kibana/bin/kibana &
    [root@elk elasticsearch-head]# ss -lntup|grep 5601
    tcp    LISTEN     0      128    192.168.6.8:5601                  *:*                   users:(("node",pid=15126,fd=13))
kibana访问： http://xxxx：5601

安装logstash

     [root@elk src]# wget https://artifacts.elastic.co/downloads/logstash/logstash-6.2.4.tar.gz
     [root@elk src]# tar xf logstash-6.2.4.tar.gz -C /opt/
     [root@elk src]# cd /opt/
     [root@elk opt]# mv logstash-6.2.4 logstash
     # logstash 启动文件
     [root@elk opt]# cat /opt/logstash/config/filebeat.yml 
      #input {
      #  beats {
      #    port => 5044
      #    type => "logs"
      #  }
      # }
      input {
          redis {
             data_type => "list"
             host => "192.168.6.9"
             db => "2"
             port => "6379"
             key => "service.log"
         }
     }
     filter {
         grok {
             match => ["message", "%{TIMESTAMP_ISO8601:logdate}"]
         }
         date {
             match => ["logdate", "yyyy-MM-dd HH:mm:ss.SSS"]
             target => "@timestamp"
        }
  
     }


     output {
           if [tags] == "redis" {             
           elasticsearch {
             hosts => ["192.168.6.8:9200"]
             index => "redis-%{+YYYY.MM.dd}"
           }
         }
           elasticsearch {
               hosts => ["192.168.6.8:9200"]
               index => "service-%{+YYYY.MM.dd}"
         }
     }
     启动logstash
     [root@elk opt]# /opt/logstash/bin/logstash -f /opt/logstash/config/filebeat.yml &
因为logstash占用资源太大，而且依赖Java环境，单单启用一个logstash就要占用500M内存，会造成很大浪费，我们采用filebeat收集的方式，然后经由logstash处理，filebeat不依赖任何环境，而且只占用10M内存左右，下面为一个应用模块的yaml示例:

[filebeat-configmap.yaml](https://github.com/Hanzhiwei210521/loading/blob/master/image/filebeat-configmap.yaml)

[activity.yaml](https://github.com/Hanzhiwei210521/loading/blob/master/image/activity.yaml)
   
## logstash 收集nginx日志 ##

elk收集分析nginx日志，需要使用grok插件将nginx日志结构化，然后根据结构化后的filed进行分析，通常我们使用到了高德地图这个功能，根据client_ip 进行定位显示，此功能需要用到IP地址库，logstash 6版本对应的地址库为GeoLite2。
 
下载geoip地址库
 
    wget http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.tar.gz
    tar xf GeoLite2-City.tar.gz -C /opt/logstash/
nginx access log_format

     log_format	 main 	'$remote_addr - $remote_user [$time_local] "$request" '
						'$status $body_bytes_sent referrer: "$http_referer" '
                        '"$http_user_agent" "$http_x_forward_for" request_time $request_time upstream_response_time $upstream_response_time "$upstream_addr"';    
nginx access log 例子：
    
    117.50.29.251 - - [27/Jun/2018:14:42:44 +0800] "GET / HTTP/1.1" 301 178 referrer: "http://www.jinxiudadi.com" "Mozilla/4.0 (compatible; MSIE 9.0; Windows NT 6.1)" "-" request_time 0.000 upstream_response_time - "-"
nginx error log 例子：

    2018/06/27 11:29:57 [error] 28408#0: *461147 open() "/opt/huoyuan/site/favicon.ico" failed (2: No such file or directory), client: 119.136.154.96, server: www.jinxiudadi.com, request: "GET /favicon.ico HTTP/1.1", host: "www.jinxiudadi.com"

nginx access pattern

    [root@es-1 config]# cat /opt/logstash/vendor/bundle/jruby/2.3.0/gems/logstash-patterns-core-4.1.2/patterns/nginx 
    NGUSERNAME [a-zA-Z\.\@\-\+_%]+
    NGUSER %{NGUSERNAME}
    NGINX_ACCESS_LOGS %{IPORHOST:client_ip} - - \[%{HTTPDATE:time_local}\] \"%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:httpversion}\" %{INT:status} %{INT:body_bytes_sent} referrer: "%{DATA:refer_url|-}" "%{DATA:ua}" \"(?:%{NUMBER:http_x_forward_for}|-)\" request_time %{NUMBER:request_time} upstream_response\_time (?:%{NUMBER:upstream_response_time}|-) "(?:%{URIHOST:upstream_addr}|-)"

logstash.conf

    [root@es-1 config]# cat logstash.conf 
    input {
        beats {
          port => 5044
          type => "logs"
        }
    }
    filter {
        if "nginx-access" in [tags] {
          grok {
            match => {
              "message" => "%{NGINX_ACCESS_LOGS}"
            }
          }
          date {
            match => ["time_local", "dd/MMM/yyyy:HH:mm:ss +0800"]
            target => "@timestamp"
          }
          geoip {
            source => "client_ip"
            target => "geoip"
            database => "/opt/logstash/GeoLite2-City_20180605/GeoLite2-City.mmdb"
            add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
            add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
          }
        }
        if "nginx-error" in [tags] {
          grok {
            match => [
              "message", "(?<time>\d{4}/\d{2}/\d{2}\s{1,}\d{2}:\d{2}:\d{2})\s{1,}\[%{DATA:err_severity}\]\s{1,}(%{NUMBER:pid:int}#%{NUMBER}:\s{1,}\*%{NUMBER}|\*%{NUMBER}) %{DATA:err_message}(?:,\s{1,}client:\s{1,}(?<client_ip>%{IP}|%{HOSTNAME}))(?:,\s{1,}server:\s{1,}%{IPORHOST:server})(?:, request: %{QS:request})?(?:, host: %{QS:client_ip})"]
          }
          date{
            match => ["time", "yyyy/MM/dd HH:mm:ss"]
            target => "@timestamp"
          }  
        }
    }
   
    output {
        if "nginx-access" in [tags] {
            elasticsearch {
               hosts => ["172.18.10.230:9200"]
               manage_template => true                                    
               index => "logstash-nginx-access-%{+YYYY-MM}"                        
            }
        }  
        if "nginx-error" in [tags] {
            elasticsearch {
               hosts => ["172.18.10.230:9200"]
               manage_template => true
               index => "logstash-nginx-error-%{+YYYY-MM}"       
            }
        }                                                          
    }


![](https://github.com/Hanzhiwei210521/loading/blob/master/image/QQ%E5%9B%BE%E7%89%8720180627155110.png)

在线gork正则的地址: <https://grokdebug.herokuapp.com/>

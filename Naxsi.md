# 什么是Naxsi？ #
Naxsi是指Nginx Anti XSS&SQL Injection，是一个开放源代码、高效、低维护规则的Nginx Web应用防火墙模块，Naxsi的主要目标是帮助人们加固Web应用程序，以抵御SQL注入、跨站脚本、跨域伪造请求、本地和远程文件等包含的漏洞

 从技术上讲，Naxsi是第三方的nginx模块，可作为许多类UNIX平台的软件包使用。该模块可以识别包含涉及网站漏洞的99%的已知模式的简单规则，主要部分是一个叫做naxsi_core.rules的核心文件，该文件并不是一个精准的防攻击策略规则，而是一个包含了几乎所有可以使用到的匹配方式的文件，Naxsi根据这个文件规则匹配合法查询。Naxsi管理员的工作为将一些触犯了naxsi _core.rules规则但又是合法行为的请求添加白名单，管理员可以分析nginx错误日志手动添加白名单，或者通过Naxsi的LearningMode学习模式添加白名单（推荐使用）。

与大多数Web应用程序防火墙相反，Naxis不像防病毒那样依赖签名库，因此无法通过未知攻击模式绕开，Naxsi和其他WAF的另一个主要区别是，Naxsi只过滤GET和POST请求。
# Naxsi模块的安装 #
    wget http://nginx.org/download/nginx-x.x.xx.tar.gz
    wget https://github.com/nbs-system/naxsi/archive/x.xx.x.tar.gz
    tar xvzf nginx-x.x.xx.tar.gz 
    tar xvzf naxsi-x.xx.tar.gz
    cd nginx-x.x.xx/
    /configure --add-module=../naxsi-x.xx/naxsi_src/ [add/remove your favorite/usual options]
    make
    make install
需要注意的是，Nginx将根据./configure中模块指令的顺序决定模块使用的具体顺序，所以，无论如何（除非你真的知道你在做什么），首先将naxsi放入./configure，否则，你可能会遇到各种问题。
# 基本配置 #
    http {} level: include naxsi_core.rules
    server {} level: Dynamic modifiers
    location {} level: 
      Enabled/Disabled directives
      LearningMode-related directives
      Whitelists
      CheckRules
      RequestDenied

Example configuration

    #Only for nginx's version with modular support
    load_module /../modules/ngx_http_naxsi_module.so;
    events {
      ...
    }
    http {
     include /tmp/naxsi_ut/naxsi_core.rules;
     ...
     server {
      listen ...;
      server_name  ...;
      location / {
     #Enable naxsi
      SecRulesEnabled;
     #Enable learning mode
      LearningMode;
      DeniedUrl "/50x.html"; 
      CheckRule "$SQL >= 8" BLOCK;
      CheckRule "$RFI >= 8" BLOCK;
      CheckRule "$TRAVERSAL >= 4" BLOCK;
      CheckRule "$EVADE >= 4" BLOCK;
      CheckRule "$XSS >= 8" BLOCK;
  
      error_log /.../foo.log;
      ...
      }
      error_page   500 502 503 504  /50x.html;
  
     location = /50x.html {
     return 418; #I'm a teapot \o/
       }
      }
    }
需要注意的是LearningMode;为开启学习模式，开启此模式，Naxsi会将触犯规则的请求记录log，并不会真正去阻止违法请求，该模式是为了让管理员根据日志进行分析，然后添加白名单，实际应用中，应关闭此模式。
## 
# naxsi_core.rules 核心规则详解(http{}level)： #
   
    ##################################
    ## INTERNAL RULES IDS:1-999     ##
    ##################################
    #@MainRule "msg:weird request, unable to parse" id:1;
    #@MainRule "msg:request too big, stored on disk and not parsed" id:2;
    #@MainRule "msg:invalid hex encoding, null bytes" id:10;
    #@MainRule "msg:unknown content-type" id:11;
    #@MainRule "msg:invalid formatted url" id:12;
    #@MainRule "msg:invalid POST format" id:13;
    #@MainRule "msg:invalid POST boundary" id:14;
    #@MainRule "msg:invalid JSON" id:15;
    #@MainRule "msg:empty POST" id:16;
    #@MainRule "msg:libinjection_sql" id:17;
    #@MainRule "msg:libinjection_xss" id:18;

    ##################################
    ## SQL Injections IDs:1000-1099 ##
    ##################################
    MainRule "rx:select|union|update|delete|insert|table|from|ascii|hex|unhex|drop" "msg:sql keywords" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1000;
    MainRule "str:\"" "msg:double quote" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:8,$XSS:8" id:1001;
    MainRule "str:0x" "msg:0x, possible hex encoding" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:2" id:1002;
    ## Hardcore rules
    MainRule "str:/*" "msg:mysql comment (/*)" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:8" id:1003;
    MainRule "str:*/" "msg:mysql comment (*/)" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:8" id:1004;
    MainRule "str:|" "msg:mysql keyword (|)"  "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:8" id:1005;
    MainRule "str:&&" "msg:mysql keyword (&&)" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:8" id:1006;
    ## end of hardcore rules
    MainRule "str:--" "msg:mysql comment (--)" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1007;
    MainRule "str:;" "msg:semicolon" "mz:BODY|URL|ARGS" "s:$SQL:4,$XSS:8" id:1008;
    MainRule "str:=" "msg:equal sign in var, probable sql/xss" "mz:ARGS|BODY" "s:$SQL:2" id:1009;
    MainRule "str:(" "msg:open parenthesis, probable sql/xss" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$SQL:4,$XSS:8" id:1010;
    MainRule "str:)" "msg:close parenthesis, probable sql/xss" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$SQL:4,$XSS:8" id:1011;
    MainRule "str:'" "msg:simple quote" "mz:ARGS|BODY|URL|$HEADERS_VAR:Cookie" "s:$SQL:4,$XSS:8" id:1013;
    MainRule "str:," "msg:comma" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1015;
    MainRule "str:#" "msg:mysql comment (#)" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1016;
    MainRule "str:@@" "msg:double arobase (@@)" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1017;

    ###############################
    ## OBVIOUS RFI IDs:1100-1199 ##
    ###############################
    MainRule "str:http://" "msg:http:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1100;
    MainRule "str:https://" "msg:https:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1101;
    MainRule "str:ftp://" "msg:ftp:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1102;
    MainRule "str:php://" "msg:php:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1103;
    MainRule "str:sftp://" "msg:sftp:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1104;
    MainRule "str:zlib://" "msg:zlib:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1105;
    MainRule "str:data://" "msg:data:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1106;
    MainRule "str:glob://" "msg:glob:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1107;
    MainRule "str:phar://" "msg:phar:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1108;
    MainRule "str:file://" "msg:file:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1109;
    MainRule "str:gopher://" "msg:gopher:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1110;

    #######################################
    ## Directory traversal IDs:1200-1299 ##
    #######################################                                          
    MainRule "str:.." "msg:double dot" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$TRAVERSAL:4" id:1200;
    MainRule "str:/etc/passwd" "msg:obvious probe" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$TRAVERSAL:4" id:1202;
    MainRule "str:c:\\" "msg:obvious windows path" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$TRAVERSAL:4" id:1203;
    MainRule "str:cmd.exe" "msg:obvious probe" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$TRAVERSAL:4" id:1204;
    MainRule "str:\\" "msg:backslash" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$TRAVERSAL:4" id:1205;
    #MainRule "str:/" "msg:slash in args" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$TRAVERSAL:2" id:1206;

    ########################################
    ## Cross Site Scripting IDs:1300-1399 ##
    ########################################
    MainRule "str:<" "msg:html open tag" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$XSS:8" id:1302;
    MainRule "str:>" "msg:html close tag" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$XSS:8" id:1303;
    MainRule "str:[" "msg:open square backet ([), possible js" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$XSS:4" id:1310;
    MainRule "str:]" "msg:close square bracket (]), possible js" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$XSS:4" id:1311;
    MainRule "str:~" "msg:tilde (~) character" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$XSS:4" id:1312;
    MainRule "str:`"  "msg:grave accent (`)" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$XSS:8" id:1314;
    MainRule "rx:%[23]."  "msg:double encoding" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$XSS:8" id:1315;

    ####################################
    ## Evading tricks IDs: 1400-1500 ##
    ####################################
    MainRule "str:&#" "msg:utf7/8 encoding" "mz:ARGS|BODY|URL|$HEADERS_VAR:Cookie" "s:$EVADE:4" id:1400;
    MainRule "str:%U" "msg:M$ encoding" "mz:ARGS|BODY|URL|$HEADERS_VAR:Cookie" "s:$EVADE:4" id:1401;

    #############################
    ## File uploads: 1500-1600 ##
    #############################
    MainRule "rx:\.ph|\.asp|\.ht" "msg:asp/php file upload" "mz:FILE_EXT" "s:$UPLOAD:8" id:1500;
### MainRule:定义检测规则和分数 ##
##### 1.MainRule后紧跟着的是匹配模式 #####
    
    "str:\"
* str:foo|bar: 字符串（类型），其后紧跟的为要具体匹配的字符 foo|bar
* rx:foo|bar: 正则匹配 匹配 foo or bar
* d:libinj_xss : libinjection检测为XSS (>= 0.55rc2)
* d:libinj_sql : libinjection检测为SQLi (>= 0.55rc2)

建议尽可能使用纯字符串匹配，因为速度更快。 所有字符串必须小写，因为naxsi的匹配不区分大小写

##### 2.然后是描述，仅用于描述规则，不会有其他动作 #####

    “msg:weird request, unable to parse” 
 
##### 3.mz是匹配区域，用于定义请求的哪部分将由规则检查。 #####
 
*  ARGS: GET的整个参数，如: foo=bar&in=%20
*  $ARGS_VAR: GET参数的参数名, 如：foo=bar&in=%20中的foo和in
*  $ARGS_VAR_X: 正则匹配的GET参数的参数名
*  HEADERS: 整个HTTP协议头
*  $HEADERS_VAR: HTTP协议头的名字
*  $HEADERS_VAR_X: 正则匹配的HTTP协议头的名字
*  BODY: POST的body内的参数
*  $BODY_VAR: POST参数的参数名
*  $BODY_VAR_X: 正则匹配的POST参数的参数名
*  URL: URL(?前的)
*  $URL_X: 正则匹配的URL(?前的)
*  FILE_EXT: 文件名 (POST上传文件时上传的文件名)

##### 4.紧跟着是评分Score #####

    "s:$SQL:2"
如果某个请求符合这条规则，那么名为$SQL的这个计数器就会加2；如果这个请求中有多个参数符合这条规则，那么计数器也会相应地增加。当$SQL这个计数器符合在location中CheckRule对于$SQL计数器的定义时，将采取对应的动作，如

    CheckRule "$SQL >= 8" BLOCK;
当计数器$SQL分数大于等于8，nginx将禁止访问

##### 5.最后是规则id，这id在编写白名单的时候会用到 #####

    id:1002
##
# naxsi动态配置(server{}level) #
从0.49版本开始，naxsi支持一组可以覆盖或修改其行为的有限变量，如下：

* naxsi_flag_learning ：如果存在，该变量将覆盖naxsi学习标志（“0”禁用学习，“1”启用它）。
* naxsi_flag_post_action ：如果存在并设置为“0”，则可以使用此变量在学习模式下禁用post_action。
* naxsi_flag_enable ：如果存在，该变量将覆盖naxsi的“SecRulesEnabled”（“0”以禁用naxsi，“1”以启用）。
* naxsi_extensive_log ：如果存在（并设置为“1”），则此变量将强制naxsi记录变量匹配规则的内容（请参见底部的注释）。

从0.54开始naxsi在运行时也支持libinjection的启用/禁用标志

* naxsi_flag_libinjection_sql
* naxsi_flag_libinjection_xss

配置方式：

     set $naxsi_flag_enable 0;
     location / {
     ...
     }
可以使用nginx，lua等的变量来改变naxsi的行为。

    if ($remote_addr = "1.2.3.4") {
    set $naxsi_flag_learning 1;
    }
    location / {
    ...
    } 

naxsi_extensivez _log

如果存在（并设置为“1”），则此变量将强制naxsi记录变量匹配规则的CONTENT，由于可能会对性能造成影响，请谨慎使用。Naxsi会将这些记录到nginx的error_log，即：

    NAXSI_EXLOG: ip=%V&server=%V&uri=%V&id=%d&zone=%s&var_name=%V&content=%V
Libinjection

libinjection是一个第三方库（由clinet9开发），旨在通过标记化来检测SQL注入（sqli）和跨站点脚本（xss）。
具体使用方法请参考 
![](https://github.com/nbs-system/naxsi/wiki/libinjection-integration)

# location{}level 配置： #
    location  / {
           include /usr/local/nginx/conf/naxsi.rules;
	       root /opt/huoyuan/site;
        }
    location /RequestDenied {
           return 403;
        }

    [root@SE1064-V0177 conf]# cat naxsi.rules 
    # 启用Naxsi模块
    SecRulesEnabled;
    #开启学习模式（实际应用应关闭）
    LearningMode;
    #拒绝访问时展示的页面
    DeniedUrl "/RequestDenied";
    #检查规则
    CheckRule "$SQL >= 8" BLOCK;
    CheckRule "$RFI >= 8" BLOCK;
    CheckRule "$TRAVERSAL >= 4" BLOCK;
    CheckRule "$EVADE >= 4" BLOCK;
    CheckRule "$XSS >= 8" BLOCK;
    #白名单
    BasicRule  wl:1015,1315 "mz:$HEADERS_VAR:cookie";

 naxsi.rules是一个通用规则，可以被include各个location，白名单的规则各个location可能有所不同，可以单独加在location位置。  

### BasicRule 定义MainRule的白名单 ###
白名单的添加最好能以日志为基础进行操作，如果你不想在配置Naxsi时候导致服务无法访问，那么可以打开学习模式。

首先看看日志都记录了什么:

    2018/04/03 14:50:21 [error] 15681#0: *460483 NAXSI_FMT: ip=106.37.192.82 &server=a168.jinxiudadi.com &uri=/static/img/header/jxdd.ico&learning=1&vers=0.55.3&total_processed=138&total_blocked=2&block=1&cscore0=$SQL&score0=16&cscore1=$XSS&score1=16&zone0=HEADERS&id0=1001&var_name0=cookie, client: 106.37.192.82 , server: a168.jinxiudadi.com , request: "GET /static/img/header/jxdd.ico HTTP/1.1", host: "a168.jinxiudadi.com ", referrer: "https://a168.jinxiudadi.com/ " 
* ip: client ip
* server: 请求的域名
* uri: 请求的uri(不带参数，？之前)
* learning: 是否处于学习模式
* vers: naxsi版本
* total_processed: nginx处理的请求总数
* total_blocked： naxsi阻止的请求总数
* block: 本次请求阻止个数
* cscoreN：触犯规则的name
* scoreN: 触犯规则的得分
* zoneN: 触犯规则的区域
* idN： 触犯规则的id
* var_nameN： 触犯规则的变量名称

从这个日志可以得出信息，naxsi开启了学习模式，版本号为0.55，本次请求客户端IP为106.37.192.82，请求的域名为a168.jinxiudadi.com，请求的uri为/static/img/header/jxdd.ico，nginx处理的总请求数138，naxsi阻止这类请求总数为2，本次阻止请求数1，触犯规则的域为HEADERS,规则id为1001，变量名为cookie,触犯规则name为$SQL，得分为16，$XSS的得分为16

那么白名单可以写成以下格式：
    BasicRule wl:1001 "mz:$HEADERS_VAR:Cookie";

### CheckRules 定义当分数达到阈值时采取的动作 ###

    CheckRule "$SQL >= 8" BLOCK;
    CheckRule "$RFI >= 8" BLOCK;
    CheckRule "$TRAVERSAL >= 4" BLOCK;
    CheckRule "$EVADE >= 4" BLOCK;
    CheckRule "$XSS >= 8" BLOCK;

以上时加载在location中的CheckRule.他们的主要作用时读取计数器中的数值，与CheckRule中所设定的数值进行对比，一旦符合设定，那么将进行相应的动作。

以上的CheckRule意味着当名为$SQL、$RFI、$TRAVERSAL、$EVADE和$XSS这些计数器的值达到设定值后，将禁止访问。

动作不仅仅有BLOCK，还有以下这些：
* DROP： 抛弃请求，不做任何响应
* BLOCK：根据DeniedUrl的设定进行跳转
* ALLOW：允许通过
* LOG：仅记录在日志中，不做任何动作。

更多使用方法请参考![](https://github.com/nbs-system/naxsi/wiki)
































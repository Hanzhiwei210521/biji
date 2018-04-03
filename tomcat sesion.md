# Tomcat 粘性链接 #

### 场景一： ###
使用单台tomcat作为web服务器，我们都知道session存在于服务器端，整个session都会由这台服务器来管理，根本不用鸟什么粘性问题。
### 场景二： ###
当我们业务量逐渐变大，单台服务器负载过高，我们就要加机器用于分布式部署，但又限于money问题，我们只能添加一台，那么我们现在就有两台web服务器，tomcatA和tomcatB，那么问题来了，用户访问系统时，通过负载均衡，此用户访问了位于tomcatA上的应用，系统给此用户分配了一个session来标识用户身份，并返回sessionID给浏览器。这时user又向服务器发送请求，假设这时候负载均衡到了tomcatB，但是此时tomcatB上并没有存有user相应的session信息，根据一般应用的设置，会提示user重新登陆，试想一下user就这样一直登陆，一直提示未登录，那将是多么糟糕的事，这就是session保持的问题。
# session保持的几种方法 #
### 1.ip_hash:###
该方法确保来自同一客户端的请求将始终传递到同一服务器，除非该服务器不可用。在后一种情况下，客户端请求将传递给另一台服务器，但是这种方法我们并不推荐使用原因如下:

1) nginx不是最前端的服务器：
ip_hash要求nginx一定是最前端的服务器，否则nginx得不到正确ip,就不能根据ip做hash。譬如使用的是squid为最前端，那么nginx取ip时只能得到squid的服务器ip地址，用这个地址来作分流时肯定错乱的。

2）nginx的后端还有其他方式的负载均衡

3）多个外网出口，很多公司上网有多个出口，多个ip地址，用户访问互联网时自动切换ip。
### 2.sticky cookie ###
    upstream backend {
         server backend1.example.com;
         server backend2.example.com;

         sticky cookie srv_id expires=1h domain=.example.com path=/;
    }
客户端第一次访问没有绑定任何server,会根据轮询算法将请求抛给后端server；当client再次访问时会带着cookie信息被抛给指定的server；如果指定的服务器无法处理请求，则选择新的服务器，就好像客户尚未绑定一样。
 
sticky cookie srv_id expires=1h domain=.example.com path=/详解：

第一个参数设置要设置或检查的cookie的名称。 Cookie值是IP地址和端口的MD5散列值或UNIX域套接字路径的十六进制表示形式。 但是，如果指定了服务器伪指令的“route”参数，则cookie值将是“route”参数的值：

    upstream backend {
        server backend1.example.com route=a;
        server backend2.example.com route=b;

        sticky cookie srv_id expires=1h domain=.example.com path=/;
    }
这种情况，“srv_id” cookie的值将是a或b。

expires=time 设置浏览器保存cookie的时长，如果值为max,那么cookie将会在“ 31 Dec 2037 23:55:55 GMT” 时过期，如果未指定参数，则会导致cookie在浏览器会话结束时过期。

domain=domain 定义设置cookie的域名

path=path 定义设置cookie的路径
### 场景三：###
随着访问量的不断变大，两台web服务器已经不能够正常的提供访问，需要再部署多台web，可以使用redis这种缓存数据库来实现session的统一管理。
![](https://github.com/Hanzhiwei210521/loading/blob/master/image/image004.png)

# Nginx-lua
第一步：创建目录
/usr/servers，以后我们把所有软件安装在此目录
mkdir -p /usr/servers  
cd /usr/servers/ 
第二步：安装依赖
首先检查一下自己的版本信息，方便安装对应的依赖
首先检查一下是否按章了gcc环境，如果出现下图即可

没有出现请使用下述指令安装gcc依赖
yum install gcc-c++
出现之后，输入 
gcc -v
打印gcc版本信息

发现是小红帽之后，去官网：http://openresty.org/cn/installation.html
发现我们需要安装的依赖在这：

因此输入：
yum install pcre-devel openssl-devel gcc curl
第三步：下载ngx_openresty-1.7.7.2.tar.gz并解压
wget http://openresty.org/download/ngx_openresty-1.7.7.2.tar.gz
 
tar -xzvf ngx_openresty-1.7.7.2.tar.gz
第四步：安装LuaJIT
cd ngx_openresty-1.7.7.2
cd bundle//LuaJIT-2.1-20150120/
make clean && make && make install

ln -sf luajit-2.1.0-alpha /usr/local/bin/luajit

第五步：下载ngx_cache_purge模块，该模块用于清理nginx缓存
wget https://github.com/FRiCKLE/ngx_cache_purge/archive/2.3.tar.gz
 
tar -xvf 2.3.tar.gz 

第六步：下载nginx_upstream_check_module模块，该模块用于ustream健康检查

第七步：安装ngx_openresty
cd /usr/servers/ngx_openresty-1.7.7.2/bundle  
 
wget https://github.com/yaoweibin/nginx_upstream_check_module/archive/v0.3.0.tar.gz  
 
tar -xvf v0.3.0.tar.gz
【说明】
--with***                             安装一些内置/集成的模块
--with-http_realip_module  取用户真实ip模块
-with-pcre                           Perl兼容的达式模块
--with-luajit                         集成luajit模块
--add-module                     添加自定义的第三方模块，如此次的ngx_che_purge

make && make install 
（此处我粘贴提示指令未发现，可能是输入法造成不识别）安装时间比较久，大概两分钟

第八步：到/usr/servers目录下
cd /usr/servers/  
 
ll 
会发现多出来了如下目录，说明安装成功

目录 解释
/usr/servers/luajit luajit环境，luajit类似于java的jit，即即时编译，lua是一种解释语言，通过luajit可以即时编译lua代码到机器代码，得到很好的性能
/usr/servers/lualib 要使用的lua库，里边提供了一些默认的lua库，如redis，json库等，也可以把一些自己开发的或第三方的放在这
/usr/servers/nginx 安装的nginx
通过/usr/servers/nginx/sbin/nginx  -V 查看nginx版本和安装的模块
第九步：启动nginx
/usr/servers/nginx/sbin/nginx
接下来该配置nginx+lua开发环境了
1、编辑nginx.conf配置文件 
vim /usr/servers/nginx/conf/nginx.conf
2、在http部分添加如下配置
#lua模块路径，多个之间”;”分隔，其中”;;”表示默认搜索路径，默认到/usr/servers/nginx下找  
 
lua_package_path "/usr/servers/lualib/?.lua;;";  #lua 模块  
 
lua_package_cpath "/usr/servers/lualib/?.so;;";  #c模块

3、为了方便开发我们在/usr/servers/nginx/conf目录下创建一个lua.conf
vi lua.conf
 
#lua.conf  
server {  
    listen       80;  
    server_name  _;  
} 
4、在nginx.conf中的http部分添加include lua.conf包含此文件片段
include lua.conf;


5、测试是否正常 
/usr/servers/nginx/sbin/nginx  -t
 如果显示如下内容说明配置成功
nginx: the configuration file /usr/servers/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/servers/nginx/conf/nginx.conf test is successful

HelloWorld
1、在lua.conf中server部分添加如下配置 
location /lua {  
 
    default_type 'text/html';  
 
        content_by_lua 'ngx.say("hello world")';  
 
}  

测试配置是否正确    
/usr/servers/nginx/sbin/nginx  -t  
(这里我因为 } 的中英文格式修改了一会，注意按报错信息修改即可，我的错误信息如下)
错误信息：

 正确如下

3、重启nginx 
/usr/servers/nginx/sbin/nginx -s reload 

4、访问如http://192.168.1.6/lua（自己的机器根据实际情况换ip），可以看到如下内容 

5、lua代码文件
我们把lua代码放在nginx配置中会随着lua的代码的增加导致配置文件太长不好维护，因此我们应该把lua代码移到外部文件中存储。 
vim /usr/servers/nginx/conf/lua/test.lua  
 
#添加如下内容  
 
ngx.say("hello world");
然后lua.conf修改为   
location /lua {  
    default_type 'text/html';  
    content_by_lua_file conf/lua/test.lua; #相对于nginx安装目录  
}   
此处conf/lua/test.lua也可以使用绝对路径/usr/servers/nginx/conf/lua/test.lua。
6、lua_code_cache 
默认情况下lua_code_cache  是开启的，即缓存lua代码，即每次lua代码变更必须reload nginx才生效，如果在开发阶段可以通过lua_code_cache  off;关闭缓存，这样调试时每次修改lua代码不需要reload nginx；但是正式环境一定记得开启缓存。
location /lua {  
        default_type 'text/html';  
        lua_code_cache off;  
        content_by_lua_file conf/lua/test.lua;  
}  
开启后reload nginx

会看到如下报警
nginx: [alert] lua_code_cache is off; this will hurt performance in /usr/servers/nginx/conf/lua.conf:8

7、错误日志
如果运行过程中出现错误，请不要忘记查看错误日志。 
tail -f /usr/servers/nginx/logs/error.log  
到此我们的基本环境搭建完毕。
nginx+lua项目构建
以后我们的nginx lua开发文件会越来越多，我们应该把其项目化，已方便开发。项目目录结构如下所示：
example
    example.conf     ---该项目的nginx 配置文件
    lua              ---我们自己的lua代码
      test.lua
    lualib            ---lua依赖库/第三方依赖
      *.lua
      *.so
 
其中我们把lualib也放到项目中的好处就是以后部署的时候可以一起部署，防止有的服务器忘记复制依赖而造成缺少依赖的情况。
 
我们将项目放到到/usr/example目录下。
/usr/servers/nginx/conf/nginx.conf配置文件如下(此处我们最小化了配置文件)通过绝对路径包含我们的lua依赖库和nginx项目配置文件。
#user  nobody;  
 
worker_processes  2;  
 
error_log  logs/error.log;  
 
events {  
 
    worker_connections  1024;  
 
}  
 
http {  
 
    include       mime.types;  
 
    default_type  text/html;  
 
  
 
    #lua模块路径，其中”;;”表示默认搜索路径，默认到/usr/servers/nginx下找  
 
    lua_package_path "/usr/example/lualib/?.lua;;";  #lua 模块  
 
    lua_package_cpath "/usr/example/lualib/?.so;;";  #c模块  
 
    include /usr/example/example.conf;  
 
}  

/usr/example/example.conf配置文件如下  lua文件我们使用绝对路径/usr/example/lua/test.lua。 
server {  
 
    listen       80;  
 
    server_name  _;  
 
  
 
    location /lua {  
 
        default_type 'text/html';  
 
        lua_code_cache off;  
 
        content_by_lua_file /usr/example/lua/test.lua;  
 
    }  
 
} 
 
到此我们就可以把example扔svn上了
现有目录结构如下：

注意这个时候我们配置的工程就还是没有更换的
看下图
Nginx.conf文件

观察一下他的配置

所以我们将其注释掉

重启Nginx服务？？？
不用，别别忘记了我们配置的缓存关闭，不需要每次都重启，我在Example的test.lua更改了一下打印信息
路径：

内容：

刷新页面，发现

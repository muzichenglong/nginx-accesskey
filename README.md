# nginx-accesskey
Mirror of accesskey module for nginx written by Mykola Grechukh
安装模块nginx-accesskey

模块地址：https://github.com/olcms2016/nginx-accesskey
步骤：
1：首先下载模块
[root@bogon bak]# git clone https://github.com/olcms2016/nginx-accesskey

2：编译模块
ps：为了保留上一个自带的模块，编译的时候也需要继续加上上一次已经安装的模块
--add-module=/data/bak/nginx-accesskey --with-http_secure_link_module

[root@bogon bak]# cd nginx-1.10.3
[root@bogon nginx-1.10.3]# ./configure --prefix=/usr/local/nginx  --add-module=/data/bak/nginx-accesskey --with-http_secure_link_module

3：make 生成二进制文件
4：查看已经安装模块信息：
[root@bogon nginx-1.10.3]# nginx -V
nginx version: nginx/1.10.3
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-16) (GCC) 
configure arguments: --prefix=/usr/local/nginx --add-module=/data/bak/nginx-accesskey --with-http_secure_link_module
[root@bogon nginx-1.10.3]# 

结合python web 实践：
1：结合昨天的示例，新增一个路径请求，并反向代理到新的python web服务去
相关的配置：

1:静态页面的web配置web.conf

server {
    listen 80;
    server_name 192.168.74.128;
    root /data/app/html/;
        
    location / {
        index  Index.html index.html;
        #proxy_pass http://192.168.182.155:8089;
    }
    
    location ~* ^/(upload)/{
    
        proxy_redirect off;
        proxy_set_header Host  $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://192.168.74.128:8089;
        
        #alias /data/app/html2/static/$1;    //文件可以放到别的目录
        #error_page 404 =200 @backend;                                         // 如果访问出现404转发到后台服务器
        
    }
}


1:静态页面的index.html

<html>
<head>
<title>protect</title>

<head>
</body>

<h1>protect</h1>
<div style="width:500px; height: 500px; border: 1px solid #ccc;">
    <img style="width:500px; height: 500px; border: 1px solid #ccc;" src='../upload/0.jpeg'>
       
</div>


<img src='/static/001.txt'>
<img src='/upload/1.mp4'>

<img src='/static/s.png'>

</body>
</html>


2:python web 代理配置文件web_app.com（因为python web使用到了相关uwsgi启动相关服务，所以这些需要配置一下使用nginx进行访问uwsgi启动的python web）

server {
    listen 8089;
    server_name 192.168.74.128;
    root /data/app/xianzhi/;
        
    location / {

       #新增模块的防盗链的位置
         accesskey on; #模块开关
         accesskey_hashmethod md5; #加密方式MD5或者SHA-1
         accesskey_arg "key"; #url中的关键字参数
         #accesskey_signature "mypass$remote_addr"; #加密值，此处为mypass和访问IP构成的字符串
         accesskey_signature "mypass$uri"; #加密值，此处为mypass和访问路径构成的字符串
       
        
        include uwsgi_params;
        uwsgi_param UWSGI_PYHOME /data/app/xianzhi;
        uwsgi_param UWSGI_CHDIR /data/app/xianzhi;
        uwsgi_param UWSGI_SCRIPT app; # 对应main.py
        uwsgi_pass  127.0.0.1:8086;
        client_max_body_size       50m;
        proxy_connect_timeout      1; #nginx跟后端服务器连接超时时间(代理连接超时)
        proxy_send_timeout         120; #后端服务器数据回传时间(代理发送超时)
        proxy_read_timeout         120; #连接成功后，后端服务器响应时间(代理接收超时)
    }
}


3:然后使用python代码生成对应的 secure_link_md5信息，用于在nginx进行相关鉴权处理

生成相关URL地址示例：
生成代相关认证签名的地址： http://192.168.74.128/upload/1.mp4?key=d289481ba0a53fa5d0cc8d42d7c1d68d
测试1：
访问： http://192.168.74.128/upload/1.mp4?key=d289481ba0a53fa5d0cc8d42d7c1d68d
可以正确下载文件
测试2：http://192.168.74.128/upload/1.mp4 返回403
测试3：修改key参数也返回了 403
注意事项PS：
部分的请求，如果直接不经过nginx访问的时候，可以直接进行处理，一个解决的方案就是设置防火墙，在linux中防火墙限制其端口，只能使用nginx进行访问，不能直接访问tomcat
加在自启动中
vim /etc/init.d/rc.local
设置如下
iptables -A INPUT -s 127.0.0.1 -p tcp -m tcp --dport 8080 -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 8080 -j DROP
–A 参数就看成是添加一条规则
–p 指定是什么协议，我们常用的tcp 协议，当然也有udp，例如53端口的DNS
–dport 就是目标端口，当数据从外部进入服务器为目标端口
–sport 数据从服务器出去，则为数据源端口使用
–j 就是指定是 ACCEPT -接收 或者 DROP 不接收
相关参考：
方法三、使用第三方模块ngx_http_accesskey_module做图片防盗链。
实现方法：
1，下载nginxhttpaccesskeymodule模块文件：nginx-accesskey-2.0.3.tar.gz；
2，解压此文件后，找到nginx-accesskey-2.0.3下的config文件。
编辑文件：
替换其中 的"$http_accesskey_module"为"ngx_http_accesskey_module"；
3，重新编译nginx：
复制代码 代码示例:
./configure --add-module=path/to/nginx-accesskey


修改nginx的conf文件，添加几行：

复制代码 代码示例:
location /download {
  accesskey             on;
  accesskey_hashmethod  md5;
  accesskey_arg         "key";
  accesskey_signature   "mypass$remote_addr";
  }

其中：
accesskey为模块开关；
accesskey_hashmethod为加密方式md5或者sha-1；
accesskey_arg为url中的关键字参数；
accesskey_signature为加密值，此处为mypass和访问ip构成的字符串。
访问测试脚本download.php：
复制代码 代码示例:
<?
  $ipkey= md5("mypass".$_server['remote_addr']);
  $output_add_key="<a href=http://www.xfcodes.com/download/g3200507120520lm.rar?key=".$ipkey.">download_add_key</a><br />";
  $output_org_url="<a href=http://www.xfcodes.com/download/g3200507120520lm.rar>download_org_path</a><br />";
  echo $output_add_key;
  echo $output_org_url;
?>

访问第一个download_add_key链接可以正常下载，第二个链接download_org_path会返回403 forbidden错误

作者：小钟钟同学
链接：https://www.jianshu.com/p/7e4bf2256968
来源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。


========================================================
cd /usr/sbin/

cp nginx nginx.bak

cd ~

/usr/sbin/nginx -V #确认你的nginx版本是1.14.1再进行下一步

wget http://nginx.org/download/nginx-1.14.1.tar.gz

tar zxvf nginx-1.14.1.tar.gz

cd /root

wget https://codeload.github.com/yunsuo-open/nginx-plugin/zip/master -O nginx-plugin-master.zip

unzip nginx-plugin-master.zip

cd nginx-1.14.1

yum install gcc #可先尝试编译，精简版系统需要安装依赖包

yum -y install pcre-devel openssl openssl-devel #可先尝试编译，精简版系统需要安装依赖包

./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --add-module=/root/nginx-plugin-master

make

rm -rf /usr/sbin/nginx

cp objs/nginx /usr/sbin/

service nginx restart

---
title: nginx配置大全
tags: [nginx]
categories: linux基础
date: 2019-09-18 19:00:00
---
# 全局配置
``` bash
# nginx启动的进程数
worker_processes 2; 
	
# 让不同的进程使用不同的cpu
worker_cpu_affinity 01 10;  

# 一个进程打开的最多文件描述符，不大于ulimit的限制
worker_rlimit_nofile 65535;

events {
	#epoll事件模型,处理效率高
	use epoll;       
	
	#nginx单个进程可连接的线程数
	worker_connections  1024;   
	
	#on:worker以串行方式处理连接,off:以并行方式,吞吐量大时建议关闭
	multi_accept on;                
}
```

# http区域设置

``` bash
http {
	# 调用mime.types，定义资源的媒体类型,#文件扩展名与类型映射表
	include  mime.types;           
	
	# 定义默认资源的媒体类型
	default_type  application/octet-stream; 
	
	# 立即将数据从磁盘读到缓存,注意：如果图片显示不正常把这个改成off.所以当 Nginx 是一个静态文件服务器的时候，开启SENDFILE 配置项能大大提高 Nginx 的性能,但是当 Nginx 是作为一个反向代理来使用的时候，SENDFILE 则没什么用了，因为 Nginx 是反向代理的时候。 in_fd 就不是文件句柄而是 socket，此时就不符合 sendfile 函数的参数要求了。
	sendfile  on;  
	
	# 防止网络阻塞，必须在sendfile开启模式才有效，与tcp_nodelay on;相反，使用数据量大的时候使用
	tcp_nopush on;                  
	
	# 可以指定一条tcp连接上最多能发送的请求数量，超过keepalive_requests数量时server端会关闭tcp连接，默认是100
	keepalive_requests 1000;	
	
	# 给客户端分配keep-alive链接超时时间
	keepalive_timeout  30;   
	
	# 要包涵在keepalived参数才有效,禁用了Nagle 算法，适合数据少的时候使用
	tcp_nodelay on;                 
	
	# nginx是支持读取非nginx标准的用户自定义header的，但是需要在http或者server下开启header的下划线支持:
	underscores_in_headers on;		
	
	# 这个将为打开文件指定缓存，默认是没有启用的，max指定缓存数量，建议和打开文件inactive 是指经过多长时间文件没被请求后删除缓存。
	open_file_cache max=102400 inactive=20s;
	
	 # 这个是指多长时间检查一次缓存的有效信息。
	open_file_cache_valid 30s;
	
	# open_file_cache指令中的inactive 参数时间内文件的最少使用次数，如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive 时间内一次没被使用，它将被移除。
	# max=64 表示设置缓存文件的最大数目为 64, 超过此数字后 Nginx 将按照 LRU 原则丢弃冷数据。
	# inactive=30d 与 open_file_cache_min_uses 8 表示如果在 30 天内某文件被访问的次数低于 8 次，那就将它从缓存中删除。
	# open_file_cache_valid 3m 表示每 3 分钟检查一次缓存中的文件元信息是否是最新的，如果不是则更新之。
	open_file_cache_min_uses 1;   
	

	# 客户端请求头部的缓冲区大小，这个可以根据你的系统分页大小来设置，一般一个请求头的大小不会超过 1k，不过由于一>般系统分页都要大于1k，所以这里设置为分页大小。分页大小可以用命令getconf PAGESIZE取得。
	client_header_buffer_size 4k; 
	
	 # 设置读取客户端请求头超时时间，默认为60s，如果在此超时时间内客户端没有发送完请求头，则响应408（RequestTime-out）状态码给客户端
	client_header_timeout 15;   
	
	# 设置读取客户端内容体超时时间，默认为60s，此超时时间指的是两次成功读操作间隔时间，而不是发送整个请求体的超时时间，如果在此超时时间内客户端没有发送任何请求体，则响应408（RequestTime-out）状态码给客户端
	client_body_timeout 15;   
	
	# 上传文件大小限制，默认1M
	client_max_body_size 10m;       

	# 告诉nginx关闭不响应的客户端连接。这将会释放那个客户端所占有的内存空间。
	reset_timedout_connection on; 
	
	# 设置发送响应到客户端的超时时间，默认为60s，此超时时间指的也是两次成功写操作间隔时间，而不是发送整个响应的超时时间。如果在此超时时间内客户端没有接收任何响应，则Nginx关闭此连接。
	send_timeout 15;     
	
	# 并不会让nginx执行的速度更快，但它可以关闭在错误页面中的nginx版本数字，这样对于安全性是有好处的。
	server_tokens off;              

	# limit模块，可防范一定量的DDOS攻击
	# 用来存储session会话的状态，如下是为session分配一个名为one的10M的内存存储区，限制了每秒只接受一个ip的一次请求 1r/s
	limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
	limit_conn_zone $binary_remote_addr zone=addr:10m;

	# 如果nginx是代理，因设置在代理上
	gzip on;	
	
	# 默认值: 0 ，不管页面多大都压缩# 设置允许压缩的页面最小字节数，页面字节数从header头中的Content-Length中进行获取。# 建议设置成大于1k的字节数，小于1k可能会越压越大。 即: gzip_min_length 1024
	gzip_min_length  10k;	
	
	# 设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流。 例如 4 4k 代表以4k为单位，按照原始数据大小以4k为单位的4倍申请内存。 4 8k 代表以8k为单位，按照原始数据大小以8k为单位的4倍申请内存。# 如果没有设置，默认值是申请跟原始数据相同大小的内存空间去存储gzip压缩结果。
	gzip_buffers     4 16k;	
	gzip_http_version 1.1;
	
	# 默认值：1(建议选择为4)# gzip压缩比/压缩级别，压缩级别 1-9，级别越高压缩率越大，当然压缩时间也就越长（传输快但比较消耗cpu）。
	gzip_comp_level 6;	
	
	# 默认值: gzip_types text/html (默认不对js/css文件进行压缩)# 压缩类型，匹配MIME类型进行压缩# 不能用通配符 text/*# (无论是否指定)text/html默认已经压缩 # 设置哪压缩种文本文件可参考 conf/mime.types
	gzip_types  text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif  application/javascript application/json;
	
	# 选择支持vary header；改选项可以让前端的缓存服务器缓存经过gzip压缩的页面; 这个可以不写，表示在传送数据时，给客户端说明我使用了gzip压缩。
	gzip_vary on;	
	gzip_proxied [off|expired|no-cache|no-store|private|no_last_modified|no_etag|auth|any] ...
	# 默认值：off
	# Nginx作为反向代理的时候启用，开启或者关闭后端服务器返回的结果，匹配的前提是后端服务器必须要返回包含"Via"的 header头。
	#off - 关闭所有的代理结果数据的压缩
	#expired - 启用压缩，如果header头中包含 "Expires" 头信息
	#no-cache - 启用压缩，如果header头中包含 "Cache-Control:no-cache" 头信息
	#no-store - 启用压缩，如果header头中包含 "Cache-Control:no-store" 头信息
	#private - 启用压缩，如果header头中包含 "Cache-Control:private" 头信息
	#no_last_modified - 启用压缩,如果header头中不包含 "Last-Modified" 头信息
	#no_etag - 启用压缩 ,如果header头中不包含 "ETag" 头信息
	#auth - 启用压缩 , 如果header头中包含 "Authorization" 头信息
	#any - 无条件启用压缩

	gzip_disable "MSIE [1-5]\.";
	# 禁用IE6的gzip压缩，又是因为杯具的IE6。当然，IE6目前依然广泛的存在，所以这里你也可以设置为“MSIE [1-5].”
	# IE6的某些版本对gzip的压缩支持很不好，会造成页面的假死，今天产品的同学就测试出了这个问题
	#后来调试后，发现是对img进行gzip后造成IE6的假死，把对img的gzip压缩去掉后就正常了
	#为了确保其它的IE6版本不出问题，所以建议加上gzip_disable的设置

	#定义日子格式
	log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
```

# proxy_pass反向代理配置详解
``` bash
	location / {
		proxy_pass  http://apachephp;
		proxy_redirect     default;		#重写后端返回的域名格式
		proxy_set_header   Host             $host;	#$host变量赋予Hsot，$host：客户端请求主机地址，$proxy_host：代理服务器请求的主机地址
		proxy_set_header   X-Real-IP        $remote_addr;	#$remote_addr：为客户端IP，把真实客户端IP写入到请求头X-Real-IP
		proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;	
		#把请求头中的X-Forwarded-For与$remote_addr用逗号合起来，如果请求头中没有X-Forwarded-For则$proxy_add_x_forwarded_for为$remote_addr
		#X-Forwarded-For代表了客户端IP，反向代理如Nginx通过$proxy_add_x_forwarded_for添加此项，X-Forwarded-For的格式为X-Forwarded-For:real client ip, proxy ip 1, proxy ip N，每经过一个反向代理就在请求头X-Forwarded-For后追加反向代理IP
		
		proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;	#注：支付功能服务最好关闭
		#error 和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现错误 
		#timeout 和后端服务器建立连接时，或者向后端服务器发送请求时，或者从后端服务器接收响应头时，出现超时 
		#invalid_header  后端服务器返回空响应或者非法响应头 
		#http_500  后端服务器返回的响应状态码为500 
		#http_502 # 后端服务器返回的响应状态码为502 
		#http_503 # 后端服务器返回的响应状态码为503 
		#http_504 # 后端服务器返回的响应状态码为504 
		#http_404 # 后端服务器返回的响应状态码为404 
		#off 	  # 停止将请求发送给下一台后端服务器

#proxy_buffers代理缓冲配置	
		proxy_temp_file_write_size 64k;
		proxy_max_temp_file_size 	0;		#临时文件由proxy_max_temp_file_size和proxy_temp_file_write_size这两个指令决定。 proxy_temp_file_write_size是一次访问能写入的临时文件的大小，默认是proxy_buffer_size和proxy_buffers中设置的缓冲区大小的2倍，Linux下一般是8k。proxy_max_temp_file_size指定当响应内容大于proxy_buffers指定的缓冲区时, 写入硬盘的临时文件的大小. 如果超过了这个值, Nginx将与Proxy服务器同步的传递内容, 而不再缓冲到硬盘. 设置为0时, 则直接关闭硬盘缓冲.
		proxy_connect_timeout      60;		#与后端/上游服务器建立连接的超时时间，默认为60s，此时间不超过75s。
		proxy_send_timeout         60;		#设置往后端/上游服务器发送请求的超时时间，默认为60s，此超时时间指的是两次成功写操作间隔时间，而不是发送整个请求的超时时间，如果在此超时时间内上游服务器没有接收任何响应，则Nginx关闭此连接。
		proxy_read_timeout         60;		#设置从后端/上游服务器读取响应的超时时间，默认为60s，此超时时间指的是两次成功读操作间隔时间，而不是读取整个响应体的超时时间，如果在此超时时间内上游服务器没有发送任何响应，则Nginx关闭此连接
		proxy_buffer_size          4k;		#后端服务器的相应头会放到proxy_buffer_size当中，这个大小默认等于proxy_buffers当中的设置单个缓冲区的大小。 proxy_buffer_size只是响应头的缓冲区，没有必要也跟着设置太大。 proxy_buffer_size最好单独设置，一般设置个4k就够了
		proxy_buffers              4 32k;	#proxy_buffers的缓冲区大小一般会设置的比较大，以应付大网页。 proxy_buffers当中单个缓冲区的大小是由系统的内存页面大小决定的，Linux系统中一般为4k。 proxy_buffers由缓冲区数量和缓冲区大小组成的。总的大小为number*size。若某些请求的响应过大,则超过_buffers的部分将被缓冲到硬盘(缓冲目录由_temp_path指令指定), 当然这将会使读取响应的速度减慢, 影响用户体验. 可以使用proxy_max_temp_file_size指令关闭磁盘缓冲.
		proxy_busy_buffers_size    64k;		#proxy_busy_buffers_size不是独立的空间，他是proxy_buffers和proxy_buffer_size的一部分。nginx会在没有完全读完后端响应的时候就开始向客户端传送数据，所以它会划出一部分缓冲区来专门向客户端传送数据(这部分的大小是由proxy_busy_buffers_size来控制的，建议为proxy_buffers中单个缓冲区大小的2倍)，然后它继续从后端取数据，缓冲区满了之后就写到磁盘的临时文件中。
		proxy_buffering						#这个参数用来控制是否打开后端响应内容的缓冲区，如果这个设置为off，那么proxy_buffers和proxy_busy_buffers_size这两个指令将会失效。 但是无论proxy_buffering是否开启，对proxy_buffer_size都是生效的

#proxy_cache代理缓存设置
	#nginx的web缓存功能的主要是由proxy_cache、fastcgi_cache指令集和相关指令集完成，proxy_cache指令负责反向代理缓存后端服务器的静态内容，fastcgi_cache主要用来处理FastCGI动态进程缓存
	http块： 
		proxy_cache_path /tmp/cache levels=1:2 keys_zone=nuget-cache:20m max_size=50g inactive=168h;
		#levels=1:2					#nginx会在上述配置的缓存文件路径下再创建两级目录，第一级目录命名为一个字符，第二级目录命名为2个字符
		#keys_zone=nuget-cache:20m	#定义缓存名称和共享内存大小，并mkdir创建nuget-cache缓存目录
		#max_size=50g				#最大缓存空间，如果缓存空间满，默认覆盖掉缓存时间最长的资源。
		#inactive=168h				#定义缓存时间
	server/location块： 
		proxy_cache nuget-cache;	#开启nuget-cache缓存
		proxy_cache_valid 200 304 1h;		#状态码200|304的过期为12h
		proxy_cache_valid 404 1m;			#状态码404的过期为1分钟
		proxy_cache_valid any 1d;			#其它的状态码过期为1天	
		proxy_ignore_headers X-Accel-Expires Expires Cache-Control Set-Cookie; #忽略后端头部缓存规则，这句代码很关键，尤其要忽略set-cookie
		proxy_hide_header Cache-Control;	#隐藏响应头部信息
		add_header Cache-Control no-store；	#设置响应头部请求
		proxy_cache_key $host$uri$is_args$args;#设置nginx服务器在内存中为缓存数据建立索引时使用的关键字

#upstream负载均衡配置
	Nginx的upstream支持5种分配方式，下面将会详细介绍，其中，前三种为Nginx原生支持的分配方式，后两种为第三方支持的分配方式：
	1、轮询         
			轮询是upstream的默认分配方式，即每个请求按照时间顺序轮流分配到不同的后端服务器，如果某个后端服务器down掉后，能自动剔除。
			upstream backend {
				server 192.168.1.101:8888;
				server 192.168.1.102:8888;
				server 192.168.1.103:8888;
			}
	2、weight        
			轮询的加强版，即可以指定轮询比率，weight和访问几率成正比，主要应用于后端服务器异质的场景下。
			upstream backend {
				server 192.168.1.101 weight=1;
				server 192.168.1.102 weight=2;
				server 192.168.1.103 weight=3;
			}
	3、ip_hash        
			每个请求按照访问ip（即Nginx的前置服务器或者客户端IP）的hash结果分配，这样每个访客会固定访问一个后端服务器，可以解决session一致问题。
			upstream backend {
				ip_hash;
				server 192.168.1.101:7777;
				server 192.168.1.102:8888;
				server 192.168.1.103:9999;
			}
	4、fair        
			fair顾名思义，公平地按照后端服务器的响应时间（rt）来分配请求，响应时间短即rt小的后端服务器优先分配请求。
			upstream backend {
				server 192.168.1.101;
				server 192.168.1.102;
				server 192.168.1.103;
				fair;
			}
	5、url_hash
			与ip_hash类似，但是按照访问url的hash结果来分配请求，使得每个url定向到同一个后端服务器，主要应用于后端服务器为缓存时的场景下。
			upstream backend {
				server 192.168.1.101;
				server 192.168.1.102;
				server 192.168.1.103;
				hash $request_uri;
				hash_method crc32;
			}
			其中，hash_method为使用的hash算法，需要注意的是：此时，server语句中不能加weight等参数。
	二、设备状态
			down：表示当前server已停用
			backup：表示当前server是备用服务器，只有其它非backup后端服务器都挂掉了或者很忙才会分配到请求。
			weight：表示当前server负载权重，权重越大被请求几率越大。默认是1.
			max_fails和fail_timeout一般会关联使用，如果某台server在fail_timeout时间内出现了max_fails次连接失败，那么Nginx会认为其已经挂掉了，从而在fail_timeout时间内不再去请求它，fail_timeout默认是10s，max_fails默认是1，即默认情况是只要发生错误就认为服务器挂掉了，如果将max_fails设置为0，则表示取消这项检查。
			
			举例说明如下：
			upstream backend {
				server    backend1.example.com    weight=5;
				server    127.0.0.1:8080          max_fails=3 fail_timeout=30s;
				server    unix:/tmp/backend3      down;     
			}

#location配置
	已=开头表示精确匹配
	如 A 中只匹配根目录结尾的请求，后面不能带任何字符串。
	^~ 开头表示uri以某个常规字符串开头，不是正则匹配
	~ 开头表示区分大小写的正则匹配;
	~* 开头表示不区分大小写的正则匹配
	/ 通用匹配, 如果没有其它匹配,任何请求都会匹配到
	顺序 no优先级：(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)

	#将符合js,css文件的等设定expries缓存参数，要求浏览器缓存。
	location ~* \.(jpg|jpeg|gif|bmp|png){
		deny all;	#location 标签，根目录下的.svn目录禁止访问
		allow   219.237.222.30;	##允许访问的ip
		limit_rate_after 100k;	#文件的大小限制
		limit_rate 100k;	#超过文件大小限制，限制速度为100k/s
		client_max_body_size 1000m; #请求数据大小限制的
		log_not_found off;#是否在error_log中记录不存在的错误。默认是
		access_log off;
		expires 1d;#缓存1天
	}
#rewrite配置
	#格式：
	rewrite regex replacement [flag]

	#正则：regex
	. ： 匹配除换行符以外的任意字符
	? ： 重复0次或1次
	+ ： 重复1次或更多次
	* ： 重复0次或更多次
	\d ：匹配数字
	^ ： 匹配字符串的开始
	$ ： 匹配字符串的介绍
	{n} ： 重复n次
	{n,} ： 重复n次或更多次
	[c] ： 匹配单个字符c
	[a-z] ： 匹配a-z小写字母的任意一个
	
	#变量赋值
	set $hostx ""; 
	set $addrs ""; 

	#用作if判断的全局变量replacement
	$args ： 			这个变量等于请求行中的参数，同$query_string
	$content_length ： 	请求头中的Content-length字段。
	$content_type ： 	请求头中的Content-Type字段。
	$document_root ： 	当前请求在root指令中指定的值。
	$host ： 			请求主机头字段，否则为服务器名称。
	$http_user_agent ： 客户端agent信息
	$http_cookie ： 	客户端cookie信息
	$limit_rate ： 		这个变量可以限制连接速率。
	$request_method ： 	客户端请求的动作，通常为GET或POST。
	$remote_addr ： 	客户端的IP地址。
	$remote_port ： 	客户端的端口。
	$remote_user ： 	已经经过Auth Basic Module验证的用户名。
	$request_filename ：当前请求的文件路径，由root或alias指令与URI请求生成。
	$scheme ： 			HTTP方法（如http，https）。
	$server_protocol ： 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。
	$server_addr ： 	服务器地址，在完成一次系统调用后可以确定这个值。
	$server_name ： 	服务器名称。
	$server_port ： 	请求到达服务器的端口号。
	$request_uri ： 	包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。
	$uri ： 			不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”。
	$document_uri ： 	与$uri相同。

	#标签flag
	last : 相当于Apache的[L]标记，表示完成rewrite
	break : 停止执行当前虚拟主机的后续rewrite指令集
	redirect : 返回302临时重定向，地址栏会显示跳转后的地址
	permanent : 返回301永久重定向，地址栏会显示跳转后的地址
	因为301和302不能简单的只返回状态码，还必须有重定向的URL，这就是return指令无法返回301,302的原因了。这里 last 和 break 区别有点难以理解：
	#注
	last一般写在server和if中，而break一般使用在location中
	last不终止重写后的url匹配，即新的url会再从server走一遍匹配流程，而break终止重写后的匹配
	break和last都能组织继续执行后面的rewrite指令
			
#https-ssl相关配置
	方法一：
	server {
		listen 80;
		server_name bjubi.com;#你的域名
		rewrite ^(.*)$ https://$host$1 permanent;#把http的域名请求转成https
	}
	server {
	　　listen 443; #监听端口
	　　server_name bjubi.com;
		root html；

	　　ssl on; #开启ssl
	　　ssl_certificate /ls/app/nginx/conf/mgmtxiangqiankeys/server.crt; #服务的证书
	　　ssl_certificate_key /ls/app/nginx/conf/mgmtxiangqiankeys/server.key; #服务端key
	　　ssl_client_certificate /ls/app/nginx/conf/mgmtxiangqiankeys/ca.crt; #客户端证书
	　　ssl_session_timeout 5m; #session超时时间
	　　ssl_verify_client on; # 开户客户端证书验证 
	　　ssl_protocols SSLv2 SSLv3 TLSv1; #允许SSL协议 
	　　ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP; #加密算法
	　　ssl_prefer_server_ciphers on; #启动加密算法
	location / {
        index index.html index.htm;
    }
	}
	方法二：
	if ($scheme != https) { # 强制 HTTP 跳转至 HTTPS
		# host 与 server_name 等价, redirect/permanent 分别为临时跳转/永久跳转
		rewrite ^(.*)$  https://$host$1 permanent; 
	}
	
#防盗链
	#防止别人直接从你网站引用图片等链接，消耗了你的资源和网络流量，那么我们的解决办法由几种： 1：水印，品牌宣传，你的带宽，服务器足够 2：防火墙，直接控制，前提是你知道IP来源 3：防盗链策略下面的方法是直接给予404的错误提示
	location ~* .*\.(gif|jpg|png|jpeg)$ {
		expires     30d;
		valid_referers *.hugao8.com www.hugao8.com m.hugao8.com *.baidu.com *.google.com;
		if ($invalid_referer) {
		#rewrite ^/ http://ww4.sinaimg.cn/bmiddle/051bbed1gw1egjc4xl7srj20cm08aaa6.jpg;
		return 404;
	}
	}
	#第一行：其中“gif|jpg|jpeg|png|bmp|swf”设置防盗链文件类型，自行修改，每个后缀用“|”符号分开！
	#第三行：就是白名单，允许文件链出的域名白名单，自行修改成您的域名！*.it300.com这个指的是子域名，域名与域名之间使用空格隔开！
	#第五行：这个图片是盗链返回的图片，也就是替换盗链网站所有盗链的图片。这个图片要放在没有设置防盗链的网站上，因为防盗链的作用，这个图片如果也放在防盗链网站上就会被当作防盗链显示不出来了，盗链者的网站所盗链图片会显示X符号。
	#第六行：直接返回404

#后端服务器获取真实IP设置
	nginx代理设置
		proxy_set_header Host $http_host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	后端服务器设置（在http或server设置，需要加载模块--with-http_realip_module）
		real_ip_header X-Forwarded-For;
		#X-Forwarded-For获取代理IP段的第一个IP为真实IP
		set_real_ip_from 192.168.0.0/16;	
		#定义从哪台服务器获取代理IP段
		real_ip_recursive on;
		#real_ip_recursive 是否递归地排除直至得到用户ip

#跨域设置
	#跨域其实就是访问不同网站的内容，可以直接使用rewrite重写实现
	#以下是反向代理实现跨域
	location /apis {
	rewrite  ^/apis/(.*)$ /$1 break;
	proxy_pass   http://localhost:82;
	}
		
实例：
user  www www;
worker_processes  4;
worker_cpu_affinity 0001 0010 0100 1000;
error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
worker_rlimit_nofile 10240;
pid        logs/nginx.pid;
events {
    use epoll;
    worker_connections  4096;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"'
                      '"$upstream_cache_status"';
access_log  logs/access.log  main;
server_tokens off;
    sendfile        on;
    #tcp_nopush     on;
    #keepalive_timeout  0;
    keepalive_timeout  65;
    #Compression Settings
    gzip on;
    gzip_comp_level 6;
    gzip_http_version 1.1;
    gzip_proxied any;
    gzip_min_length 1k;
    gzip_buffers 16 8k;
    gzip_types text/plain text/css text/javascript application/json application/javascript application/x-javascript application/xml;
    gzip_vary on;
    #end gzip
    # http_proxy Settings
    client_max_body_size   10m;
    client_body_buffer_size   128k;
    proxy_connect_timeout   75;
    proxy_send_timeout   75;
    proxy_read_timeout   75;
    proxy_buffer_size   4k;
    proxy_buffers   4 32k;
    proxy_busy_buffers_size   64k;
	proxy_temp_file_write_size  64k;
	proxy_buffering on;
    proxy_temp_path /usr/local/nginx1.10/proxy_temp;
    proxy_cache_path /usr/local/nginx1.10/proxy_cache levels=1:2 keys_zone=my-cache:100m max_size=1000m inactive=600m max_size=2g;
    #load balance Settings
    upstream backend {
        sticky;
        server 192.168.31.141:80 weight=1 max_fails=2 fail_timeout=10s;
        server 192.168.31.250:80 weight=1 max_fails=2 fail_timeout=10s;
    }
    #virtual host Settings
    server {
        listen       80;
        server_name  localhost;
        charset utf-8;
        location  ~/purge(/.*) {
           allow 127.0.0.1;
           allow 192.168.31.0/24;
           deny all;
           proxy_cache_purge my-cache $host$1$is_args$args;
        }
        location / {
            index  index.php index.html index.htm;
            proxy_pass        http://backend;
            proxy_redirect off;
            proxy_set_header  Host  $host;
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_ignore_headers Set-Cookie;
proxy_hide_header Set-Cookie;
            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        }
        location ~ .*\.(gif|jpg|png|html|htm|css|js|ico|swf|pdf)(.*) {
           proxy_pass  http://backend;
           proxy_redirect off;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
           proxy_cache my-cache;
           add_header Nginx-Cache $upstream_cache_status;
           proxy_cache_valid 200 304 301 302 8h;
           proxy_cache_valid 404 1m;
           proxy_cache_valid any 1d;
           proxy_cache_key $host$uri$is_args$args;
           expires 30d;
        }
        location /nginx_status {
            stub_status on;
            access_log off;
            allow 192.168.31.0/24;
            deny all;
        }
    }
}
```



### Nginx变量机制

#### 1. 内置变量

Nginx在启动时，http\_core模块会注册一些内部变量，如下：

```
$arg_name
请求中的的参数名，即“?”后面的arg_name=arg_value形式的arg_name

$args
请求中的参数值

$binary_remote_addr
客户端地址的二进制形式, 固定长度为4个字节

$body_bytes_sent
传输给客户端的字节数，响应头不计算在内；这个变量和Apache的mod_log_config模块中的“%B”参数保持兼容

$bytes_sent
传输给客户端的字节数 (1.3.8, 1.2.5)

$connection
TCP连接的序列号 (1.3.8, 1.2.5)

$connection_requests
TCP连接当前的请求数量 (1.3.8, 1.2.5)

$content_length
“Content-Length” 请求头字段

$content_type
“Content-Type” 请求头字段

$cookie_name
cookie名称

$document_root
当前请求的文档根目录或别名

$document_uri
同 $uri

$host
优先级如下：HTTP请求行的主机名>”HOST”请求头字段>符合请求的服务器名

$hostname
主机名

$http_name
匹配任意请求头字段； 变量名中的后半部分“name”可以替换成任意请求头字段，如在配置文件中需要获取http请求头：“Accept-Language”，那么将“－”替换为下划线，大写字母替换为小写，形如：$http_accept_language即可。

$https
如果开启了SSL安全模式，值为“on”，否则为空字符串。

$is_args
如果请求中有参数，值为“?”，否则为空字符串。

$limit_rate
用于设置响应的速度限制，详见 limit_rate。

$msec
当前的Unix时间戳 (1.3.9, 1.2.6)

$nginx_version
nginx版本

$pid
工作进程的PID

$pipe
如果请求来自管道通信，值为“p”，否则为“.” (1.3.12, 1.2.7)

$proxy_protocol_addr
获取代理访问服务器的客户端地址，如果是直接访问，该值为空字符串。(1.5.12)

$query_string
同 $args

$realpath_root
当前请求的文档根目录或别名的真实路径，会将所有符号连接转换为真实路径。

$remote_addr
客户端地址

$remote_port
客户端端口

$remote_user
用于HTTP基础认证服务的用户名

$request
代表客户端的请求地址

$request_body
客户端的请求主体
此变量可在location中使用，将请求主体通过proxy_pass, fastcgi_pass, uwsgi_pass, 和 scgi_pass传递给下一级的代理服务器。

$request_body_file
将客户端请求主体保存在临时文件中。文件处理结束后，此文件需删除。如果需要之一开启此功能，需要设置client_body_in_file_only。如果将次文件传递给后端的代理服务器，需要禁用request body，即设置proxy_pass_request_body off，fastcgi_pass_request_body off, uwsgi_pass_request_body off, or scgi_pass_request_body off 。

$request_completion
如果请求成功，值为”OK”，如果请求未完成或者请求不是一个范围请求的最后一部分，则为空。

$request_filename
当前连接请求的文件路径，由root或alias指令与URI请求生成。

$request_length
请求的长度 (包括请求的地址, http请求头和请求主体) (1.3.12, 1.2.7)

$request_method
HTTP请求方法，通常为“GET”或“POST”

$request_time
处理客户端请求使用的时间 (1.3.9, 1.2.6); 从读取客户端的第一个字节开始计时。

$request_uri
这个变量等于包含一些客户端请求参数的原始URI，它无法修改，请查看$uri更改或重写URI，不包含主机名，例如：”/cnphp/test.php?arg=freemouse”。

$scheme
请求使用的Web协议, “http” 或 “https”

$sent_http_name
可以设置任意http响应头字段； 变量名中的后半部分“name”可以替换成任意响应头字段，如需要设置响应头Content-length，那么将“－”替换为下划线，大写字母替换为小写，形如：$sent_http_content_length 4096即可。

$server_addr
服务器端地址，需要注意的是：为了避免访问linux系统内核，应将ip地址提前设置在配置文件中。

$server_name
服务器名，www.cnphp.info

$server_port
服务器端口

$server_protocol
服务器的HTTP版本, 通常为 “HTTP/1.0” 或 “HTTP/1.1”

$status
HTTP响应代码 (1.3.2, 1.2.2)

$tcpinfo_rtt, $tcpinfo_rttvar, $tcpinfo_snd_cwnd, $tcpinfo_rcv_space
客户端TCP连接的具体信息

$time_iso8601
服务器时间的ISO 8610格式 (1.3.12, 1.2.7)

$time_local
服务器时间（LOG Format 格式） (1.3.12, 1.2.7)

$uri
请求中的当前URI(不带请求参数，参数位于$args)，可以不同于浏览器传递的$request_uri的值，它可以通过内部重定向，或者使用index指令进行修改，$uri不包含主机名，如”/foo/bar.html”。

$realpath_root
真实的目录地址而不是链接

$proxy_add_x_forwarded_for
变量包含客户端请求头中的"X-Forwarded-For"，与$remote_addr两部分，他们之间用逗号分开。

$proxy_host
该变量获取的是upstream的上游代理名称，例如upstream backend 

$proxy_port
该变量表示的是要代理到的端口

$proxy_protocol_addr

$upstream_addr
代理到上游的服务器地址信息

$upstream_cache_status
代理到上游的服务器地址信息

$upstream_response_length
上游服务器响应的时间

$upstream_status
上游服务器响应的状态码
```

#### 2. rewrite模块的set指令

除了内置变量，可以使用rewrite模块的set指令声明变量。使用set指令声明的变量，在nginx启动并读取配置文件时注册，得到真正的值的时间是在请求处理的rewrite阶段。

#### 3. 内部机制

通过set指令注册的变量与内置变量没有本质的区别，都是在http\_core模块的main结构体的variables\_keys字段中加入一组key value对，其中key是变量名，value则是该变量的元数据，定义了该变量的get,set方法等，其类型为ngx\_http\_variable\_s。

```cpp
struct ngx_http_variable_s {
    ngx_str_t                     name;   /* must be first to build the hash */
    ngx_http_set_variable_pt      set_handler;
    ngx_http_get_variable_pt      get_handler;
    uintptr_t                     data;
    ngx_uint_t                    flags;
    ngx_uint_t                    index;
};

typedef void (*ngx_http_set_variable_pt) (ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
typedef ngx_int_t (*ngx_http_get_variable_pt) (ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
```
get和set方法是一对相反的作用，get一般是从请求r中得到内容放入v中，set一般是将v的内容放入r中，其中data是该变量value中的data。

- 注册一个变量
```cpp
ngx_http_variable_t *
ngx_http_add_variable(ngx_conf_t *cf, ngx_str_t *name, ngx_uint_t flags)
```
其中cf描述了当前解析的配置，cf->ctx则是当前解析到的http上下文(http{} 或 server {} 或 location{})。name是变量名，flags对应了上述http_variable_s的flags，其值是由下面的几个属性组合而成:

```cpp
#define NGX_HTTP_VAR_CHANGEABLE   1
#define NGX_HTTP_VAR_NOCACHEABLE  2
#define NGX_HTTP_VAR_INDEXED      4
#define NGX_HTTP_VAR_NOHASH       8
NGX_HTTP_VAR_CHANGEABLE
NGX_HTTP_VAR_NOCACHEABLE表示这个变量每次都要去取值，而不是直接返回上次cache的值(配合对应的接口).
NGX_HTTP_VAR_INDEXED表示这个变量是用索引读取的.
NGX_HTTP_VAR_NOHASH表示这个变量不需要被hash.
```
其中indexed标识，用户一般不能显式的使用，利用set指令定义的变量总是indexed的。nocacheable标识则表明了，indexed类型的变量不用缓存(只有indexed的变量会有缓存，不需要每次都调用get方法获取变量值)。

- 注册一个可被索引的变量
```cpp
ngx_int_t
ngx_http_get_variable_index(ngx_conf_t *cf, ngx_str_t *name)
```
在使用上述的add函数注册一个变量后，如果要使变量变为可被索引的(indexed)，需要执行ngx_http_get_variable_index，获得或生成一个该变量的索引。

- 获取变量值
```cpp
ngx_http_variable_value_t *
ngx_http_get_variable(ngx_http_request_t *r, ngx_str_t *name, ngx_uint_t key)
```
    - 对于不可索引的变量，该函数每次调用get方法获得变量值
    - 对于可索引变量，若该变量是nocache的则每次用get方法获取，否则使用缓存在
    r->variable数组中的值。
    
- 设置变量值
变量的set方法，仅会在set指令中触发。简单来说，如果该变量没有set方法时，使用set指令会给出一个默认的set方法，否则使用变量自定义的set方法，对变量自定义的get方法同理。**注意：**不同于get方法，set方法仅会在rewrite阶段执行。



## 使用

nginx 基于模块化的构建方式

比较重要的

1. nginx内核模块
2. nginx事件模块。估摸着，类似netty
3. nginx http内核模块

你观察nginx配置文件，也可以知道nginx的配置项是按照这个顺序组织的。

## nginx http内核模块配置项

	http{
		server{
			server_name www.domain1.com
			location /hello{
				proxy_pass http://ip:port/...
			}
			localtion /world{
			}
		}
		server{
			server_name www.domain2.com
			localtion{
				
			}
		}
	}

用户在浏览器中键入`http://www.domain1.com/hello`

1. 客户端将`www.domain1.com`解析成ip
2. 访问`ip:80`，`www.domain1.com`会作为host header设置在http请求中
3. nginx 接到请求，根据Host header每个server的server_name进行匹配，选取server。
4. 然后根据uri的剩余部分对localtion进行匹配

## upstream是负载均衡模块的配置项
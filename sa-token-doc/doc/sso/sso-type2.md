# SSO模式二 URL重定向传播会话

如果我们的多个系统：部署在不同的域名之下，但是后端可以连接同一个Redis，那么便可以使用 **`[URL重定向传播会话]`** 的方式做到单点登录。


### 0、解题思路

首先我们再次复习一下，多个系统之间为什么无法同步登录状态？

1. 前端的`Token`无法在多个系统下共享。
2. 后端的`Session`无法在多个系统间共享。

关于第二点，我们已在 "SSO模式一" 章节中阐述，使用 [Alone独立Redis插件](/plugin/alone-redis) 做到权限缓存直连 SSO-Redis 数据中心，在此不再赘述。

而第一点，才是我们解决问题的关键所在，在跨域模式下，意味着 "共享Cookie方案" 的失效，我们必须采用一种新的方案来传递Token。

1. 用户在 子系统 点击 `[登录]` 按钮。
2. 用户跳转到子系统登录接口 `/sso/login`，并携带 `back参数` 记录初始页面URL。
	- 形如：`http://{sso-client}/sso/login?back=xxx` 
3. 子系统检测到此用户尚未登录，再次将其重定向至SSO认证中心，并携带`redirect参数`记录子系统的登录页URL。
	- 形如：`http://{sso-server}/sso/auth?redirect=xxx?back=xxx` 
4. 用户进入了 SSO认证中心 的登录页面，开始登录。
5. 用户 输入账号密码 并 登录成功，SSO认证中心再次将用户重定向至子系统的登录接口`/sso/login`，并携带`ticket码`参数。
	- 形如：`http://{sso-client}/sso/login?back=xxx&ticket=xxxxxxxxx`
6. 子系统根据 `ticket码` 从 `SSO-Redis` 中获取账号id，并在子系统登录此账号会话。
7. 子系统将用户再次重定向至最初始的 `back` 页面。

整个过程，除了第四步用户在SSO认证中心登录时会被打断，其余过程均是自动化的，当用户在另一个子系统再次点击`[登录]`按钮，由于此用户在SSO认证中心已有会话存在，
所以第四步也将自动化，也就是单点登录的最终目的 —— 一次登录，处处通行。

下面我们按照步骤依次完成上述过程：

### 1、准备工作 
首先修改hosts文件`(C:\windows\system32\drivers\etc\hosts)`，添加以下IP映射，方便我们进行测试：
``` url
127.0.0.1 sa-sso-server.com
127.0.0.1 sa-sso-client1.com
127.0.0.1 sa-sso-client2.com
127.0.0.1 sa-sso-client3.com
```


### 2、搭建 SSO-Server 认证中心

> 搭建示例在官方仓库的 `/sa-token-demo/sa-token-demo-sso2-server/`，如遇到难点可结合源码进行测试学习

#### 2.1、创建 SSO-Server 端项目
创建 SpringBoot 项目 `sa-token-demo-sso-server`，引入依赖：

``` xml
<!-- Sa-Token 权限认证, 在线文档：http://sa-token.dev33.cn/ -->
<dependency>
	<groupId>cn.dev33</groupId>
	<artifactId>sa-token-spring-boot-starter</artifactId>
	<version>${sa.top.version}</version>
</dependency>

<!-- Sa-Token 整合 Redis (使用 jackson 序列化方式) -->
<dependency>
	<groupId>cn.dev33</groupId>
	<artifactId>sa-token-dao-redis-jackson</artifactId>
	<version>${sa.top.version}</version>
</dependency>
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-pool2</artifactId>
</dependency>
```

#### 2.2、开放认证接口
一个完整的SSO认证中心应该至少包含以下接口：
- `/sso/auth`：单点登录统一认证地址。
- `/sso/doLogin`：RestAPI 登录接口，根据账号密码进行登录。
- `/sso/logout`：统一单点注销地址，一次注销，全端下线。

别急，这里不需要你亲自完成这些接口 —— Sa-Token 已经为你封装了实现。你要做的，就是提供一个访问入口，接入 Sa-Token 的方法。

``` java
/**
 * Sa-Token-SSO Server端 Controller 
 */
@RestController
public class SsoServerController {

	/*
	 * SSO-Server端：处理所有SSO相关请求 
	 * 		http://{host}:{port}/sso/auth           -- 单点登录授权地址，接受参数：redirect=授权重定向地址 
	 * 		http://{host}:{port}/sso/doLogin        -- 账号密码登录接口，接受参数：name、pwd 
	 * 		http://{host}:{port}/sso/checkTicket    -- Ticket校验接口（isHttp=true时打开），接受参数：ticket=ticket码、ssoLogoutCall=单点注销回调地址 [可选] 
	 * 		http://{host}:{port}/sso/logout         -- 单点注销地址（isSlo=true时打开），接受参数：loginId=账号id、secretkey=接口调用秘钥 
	 */
	@RequestMapping("/sso/*")
	public Object ssoRequest() {
		return SaSsoHandle.serverRequest();
	}
	
	// 配置SSO相关参数 
	@Autowired
	private void configSso(SaTokenConfig cfg) {
		cfg.sso
			// 配置：未登录时返回的View 
			.setNotLoginView(() -> {
				String msg = "当前会话在SSO-Server端尚未登录，请先访问"
						+ "<a href='/sso/doLogin?name=sa&pwd=123456' target='_blank'> doLogin登录 </a>"
						+ "进行登录之后，刷新页面开始授权";
				return msg;
			})
			// 配置：登录处理函数 
			.setDoLoginHandle((name, pwd) -> {
				// 此处仅做模拟登录，真实环境应该查询数据进行登录 
				if("sa".equals(name) && "123456".equals(pwd)) {
					StpUtil.login(10001);
					return SaResult.ok("登录成功！");
				}
				return SaResult.error("登录失败！");
			})
			;
	}
	
}
```
注意：在`setDoLoginHandle`函数里如果要获取name, pwd以外的参数，可通过`SaHolder.getRequest().getParam("xxx")`来获取 

#### 2.3、application.yml配置
``` yml
# 端口
server:
    port: 9000

# Sa-Token配置
sa-token: 
	# SSO-相关配置
	sso: 
		# Ticket有效期 (单位: 秒)，默认五分钟 
		ticket-timeout: 300
		# 所有允许的授权回调地址 （此处为了方便测试配置为*，线上生产环境一定要配置为详细地地址）
		allow-url: "*"
        
spring: 
    # Redis配置 
    redis:
        # Redis数据库索引（默认为0）
        database: 1
        # Redis服务器地址
        host: 127.0.0.1
        # Redis服务器连接端口
        port: 6379
        # Redis服务器连接密码（默认为空）
        password: 
```
注意点：`allow-url`为了方便测试配置为`*`，线上生产环境一定要配置为详细URL地址 （之后的章节我们会详细阐述此配置项）

#### 2.4、创建SSO-Server端启动类
``` java 
@SpringBootApplication
public class SaSsoServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(SaSsoServerApplication.class, args);
		System.out.println("\nSa-Token-SSO 认证中心启动成功");
	}
}
```


### 3、搭建 SSO-Client 应用端 

> 搭建示例在官方仓库的 `/sa-token-demo/sa-token-demo-sso2-client/`，如遇到难点可结合源码进行测试学习

#### 3.1、创建SSO-Client端项目
创建一个 SpringBoot 项目 `sa-token-demo-sso-client`，引入依赖：
``` xml
<!-- Sa-Token 权限认证, 在线文档：http://sa-token.dev33.cn/ -->
<dependency>
	<groupId>cn.dev33</groupId>
	<artifactId>sa-token-spring-boot-starter</artifactId>
	<version>${sa.top.version}</version>
</dependency>

<!-- Sa-Token 整合redis (使用jackson序列化方式) -->
<dependency>
	<groupId>cn.dev33</groupId>
	<artifactId>sa-token-dao-redis-jackson</artifactId>
	<version>${sa.top.version}</version>
</dependency>
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-pool2</artifactId>
</dependency>

<!-- Sa-Token插件：权限缓存与业务缓存分离 -->
<dependency>
	<groupId>cn.dev33</groupId>
	<artifactId>sa-token-alone-redis</artifactId>
	<version>${sa.top.version}</version>
</dependency>
```


#### 3.2、创建 SSO-Client 端认证接口

同 SSO-Server 一样，Sa-Token 为 SSO-Client 端所需代码也提供了完整的封装，你只需提供一个访问入口，接入 Sa-Token 的方法即可。

``` java

/**
 * Sa-Token-SSO Client端 Controller 
 */
@RestController
public class SsoClientController {

	// 首页 
	@RequestMapping("/")
	public String index() {
		String str = "<h2>Sa-Token SSO-Client 应用端</h2>" + 
					"<p>当前会话是否登录：" + StpUtil.isLogin() + "</p>" + 
					"<p><a href=\"javascript:location.href='/sso/login?back=' + encodeURIComponent(location.href);\">登录</a> " + 
					"<a href='/sso/logout?back=self'>注销</a></p>";
		return str;
	}
	
	/*
	 * SSO-Client端：处理所有SSO相关请求 
	 * 		http://{host}:{port}/sso/login          -- Client端登录地址，接受参数：back=登录后的跳转地址 
	 * 		http://{host}:{port}/sso/logout         -- Client端单点注销地址（isSlo=true时打开），接受参数：back=注销后的跳转地址 
	 * 		http://{host}:{port}/sso/logoutCall     -- Client端单点注销回调地址（isSlo=true时打开），此接口为框架回调，开发者无需关心
	 */
	@RequestMapping("/sso/*")
	public Object ssoRequest() {
		return SaSsoHandle.clientRequest();
	}

}
```

##### 3.3、配置SSO认证中心地址 
你需要在 `application.yml` 配置如下信息：
``` yml
# 端口
server:
    port: 9001
	
# sa-token配置 
sa-token: 
	# SSO-相关配置
	sso: 
		# SSO-Server端 统一认证地址 
		auth-url: http://sa-sso-server.com:9000/sso/auth
        # 是否打开单点注销接口
        is-slo: true
	
	# 配置Sa-Token单独使用的Redis连接 （此处需要和SSO-Server端连接同一个Redis）
	alone-redis: 
		# Redis数据库索引 (默认为0)
		database: 1
		# Redis服务器地址
		host: 127.0.0.1
		# Redis服务器连接端口
		port: 6379
		# Redis服务器连接密码（默认为空）
		password: 
```
注意点：`sa-token.alone-redis` 的配置需要和SSO-Server端连接同一个Redis（database也要一样）

#### 3.4、写启动类
``` java
@SpringBootApplication
public class SaSsoClientApplication {
	public static void main(String[] args) {
		SpringApplication.run(SaSsoClientApplication.class, args);
		System.out.println("\nSa-Token-SSO Client端启动成功");
	}
}
```
启动项目 


### 4、测试访问 

(1) 依次启动 `SSO-Server` 与 `SSO-Client`，然后从浏览器访问：[http://sa-sso-client1.com:9001/](http://sa-sso-client1.com:9001/)

![sso-client-index.png](https://oss.dev33.cn/sa-token/doc/sso/sso-client-index.png 's-w-sh')

(2) 首次打开，提示当前未登录，我们点击**`登录`** 按钮，页面会被重定向到登录中心

![sso-server-auth.png](https://oss.dev33.cn/sa-token/doc/sso/sso-server-auth.png 's-w-sh')

(3) SSO-Server提示我们在认证中心尚未登录，我们点击 **`doLogin登录`**按钮进行模拟登录

![sso-server-dologin.png](https://oss.dev33.cn/sa-token/doc/sso/sso-server-dologin.png 's-w-sh')

(4) SSO-Server认证中心登录成功，我们回到刚才的页面刷新页面

![sso-client-index-ok.png](https://oss.dev33.cn/sa-token/doc/sso/sso-client-index-ok.png 's-w-sh')

(5) 页面被重定向至`Client`端首页，并提示登录成功，至此，`Client1`应用已单点登录成功！ 

(6) 我们再次访问`Client2`：[http://sa-sso-client2.com:9001/](http://sa-sso-client2.com:9001/)

![sso-client2-index.png](https://oss.dev33.cn/sa-token/doc/sso/sso-client2-index.png 's-w-sh')

(7) 提示未登录，我们点击**`登录`**按钮，会直接提示登录成功

![sso-client2-index-ok.png](https://oss.dev33.cn/sa-token/doc/sso/sso-client2-index-ok.png 's-w-sh')

(8) 同样的方式，我们打开`Client3`，也可以直接登录成功：[http://sa-sso-client3.com:9001/](http://sa-sso-client3.com:9001/)

![sso-client3-index-ok.png](https://oss.dev33.cn/sa-token/doc/sso/sso-client3-index-ok.png 's-w-sh')

至此，测试完毕！

可以看出，除了在`Client1`端我们需要手动登录一次之外，在`Client2端`和`Client3端`都是可以无需验证，直接登录成功的。

我们可以通过 F12控制台 Netword跟踪整个过程

![sso-genzong](https://oss.dev33.cn/sa-token/doc/sso/sso-genzong.png 's-w-sh')


### 5、运行官方仓库 

以上示例，虽然完整的复现了单点登录的过程，但是页面还是有些简陋，我们可以运行一下官方仓库的示例，里面有制作好的登录页面

> 下载官方示例，依次运行：
> - `/sa-token-demo/sa-token-demo-sso2-server/`
> - `/sa-token-demo/sa-token-demo-sso2-client/`
> 
> 然后访问：
> - [http://sa-sso-client1.com:9001/](http://sa-sso-client1.com:9001/)

![sso-server-login-hua](https://oss.dev33.cn/sa-token/doc/sso/sso-server-login-hua.png 's-w-sh')

默认测试密码：`sa / 123456`，其余流程保持不变 




### 6、跨Redis的单点登录
以上流程解决了跨域模式下的单点登录，但是后端仍然采用了共享Redis来同步会话，如果我们的架构设计中Client端与Server端无法共享Redis，又该怎么完成单点登录？

这就要采用模式三了，且往下看：[SSO模式三：Http请求获取会话](/sso/sso-type3)


<!-- 
### 6、如何单点注销？
由于Server端与所有Client端都是在共用同一套会话，因此只要一端注销，即可全端下线，达到单点注销的效果 

在`SsoClientController`中添加以下代码：
``` java
// SSO-Client端：单点注销 (所有端一起下线)
@RequestMapping("logout")
public SaResult logout() {
	StpUtil.logout();
	return SaResult.ok();
}
``` -->









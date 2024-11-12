# 怎么在 Java 中限制用户访问频率


我们当然要限制用户访问频率，因为用户可能生气，并狂点我们的网站或应用。

他也可能很坏，使用一些爬虫试图拖垮我们的服务器。

所以怎么实现呢？

本文使用 springboot，并将用户的信息和访问频率记录到 redis 中，如果你没有使用 redis，也不影响，你可以参考着自己实现，比如存储到内存或数据库中。

## 想想这个需求，从第一性原理出发
用户可能没有登录，或者已经登录了。

如果用户登录了，我们就根据用户名来限制，否则，就根据IP或者其它设备码来限制，本文假设使用IP。

我们希望它足够简单，可以在多个方法上使用，而不需要编写额外的代码。所以我们要使用接口切面。

## 接口
```java
public @interface RequestRateLimit {

	/**
	 * 限流的key，比如限制用户注册，限制用户发送邮件，等等，一般是方法名
	 * @return
	 */
	String key() default &#34;&#34;;

	/**
	 * 限流模式,默认单机
	 * @return
	 */
	RateType type() default RateType.PER_CLIENT;

	/**
	 * 限流速率，1次/分钟
	 * @return
	 */
	long rate() default 1;

	/**
	 * 限流速率，每分钟
	 * @return
	 */
	long rateInterval() default 60 * 1000;

	/**
	 * 限流速率单位
	 * @return
	 */
	RateIntervalUnit timeUnit() default RateIntervalUnit.MILLISECONDS;

}

```

## 切面
你可以直接拷贝这些代码并测试。

```java
public class RequestRateLimitAspect {

	private RedissonClient redisson;
	private final UserService userService;

	/**
	 * 根据自定义注解获取切点
	 *
	 * @param RequestRateLimit 注解接口
	 */
	@Pointcut(&#34;@annotation(RequestRateLimit)&#34;)
	public void findAnnotationPointCut(RequestRateLimit RequestRateLimit) {
	}

	@Around(value = &#34;findAnnotationPointCut(requestRateLimit)&#34;, argNames = &#34;joinPoint,requestRateLimit&#34;)
	public Object around(ProceedingJoinPoint joinPoint, RequestRateLimit requestRateLimit) throws Throwable {
		UserEntity user = userService.getCurrentRequestUser(); // 只是封装了 SecurityContextHolder.getContext().getAuthentication().getPrincipal();
		String realIp = &#34;&#34;;
		if (user == null) {
			RequestAttributes ra = RequestContextHolder.getRequestAttributes();
			ServletRequestAttributes sra = (ServletRequestAttributes) ra;
			if (null != sra) {
				HttpServletRequest request = sra.getRequest();
				realIp = request.getHeader(&#34;His-Real-IP&#34;);
				if (notValidIp(realIp)) {
					realIp = request.getHeader(&#34;His-Real-IP2&#34;);
					if (notValidIp(realIp)) {
						realIp = request.getRemoteAddr();
					}
				}
			}
		}
		if (user == null &amp;&amp; notValidIp(realIp)) {
			return R.failed(EMPTY_USER, &#34;未找到您的任何登录信息&#34;);
		}
		// 限流拦截器
		String key = user == null || StrUtil.isBlank(user.getUserName()) ? realIp : user.getUserName();
		key = key &#43; &#34;::&#34; &#43; joinPoint.getSignature().getName();
		RRateLimiter limiter = getRateLimiter(requestRateLimit, key);
		if (limiter.tryAcquire(1)) {
			return joinPoint.proceed();
		} else {
			log.info(&#34;rate-limit: {} {} {}&#34;, user == null ? &#34;&#34; : user.getUserName(), realIp, joinPoint.getSignature());
			return R.failed(REACH_REQUEST_LIMIT, String.format(&#34;请求过于频繁，请于以下时间后重试：%s %s&#34;, requestRateLimit.rateInterval(), requestRateLimit.timeUnit().name().toLowerCase()));
		}
	}

	private boolean notValidIp(String ip) {
		return StrUtil.isBlank(ip) || ip.startsWith(&#34;172.1&#34;); // docker bridge ip
	}

	/**
	 * 获取限流拦截器
	 *
	 * @param limit  在要限流的方法上的配置
	 * @param defaultKey 在redis中的存储的key
	 * @return 限流器
	 */
	private RRateLimiter getRateLimiter(RequestRateLimit limit, String defaultKey) {
		RRateLimiter rRateLimiter = redisson.getRateLimiter(StrUtil.isBlank(limit.key()) ? RATE_LIMITER &#43; &#34;::&#34; &#43; defaultKey : limit.key()); // RATE_LIMITER 随意起名，比如可以使用你的项目名称，只是为了在redis中好区分
		// 设置限流
		if (rRateLimiter.isExists()) {
			RateLimiterConfig existed = rRateLimiter.getConfig();
			// 判断配置是否更新，如果更新，重新加载限流器配置
			if (!Objects.equals(limit.rate(), existed.getRate())
					|| !Objects.equals(limit.timeUnit().toMillis(limit.rateInterval()), existed.getRateInterval())
					|| !Objects.equals(limit.type(), existed.getRateType())) {
				rRateLimiter.delete();
				rRateLimiter.trySetRate(limit.type(), limit.rate(), limit.rateInterval(), limit.timeUnit());
				expireByConfig(rRateLimiter, limit);
			}
		} else {
			rRateLimiter.trySetRate(limit.type(), limit.rate(), limit.rateInterval(), limit.timeUnit());
			expireByConfig(rRateLimiter, limit);
		}

		return rRateLimiter;
	}

	private void expireByConfig(RRateLimiter rRateLimiter, RequestRateLimit limit) {
		// ttl 设置为 rateLimit 配置时间 &#43; 5s
		long limitDuration = limit.timeUnit().toMillis(limit.rateInterval()) &#43; 5000;
		// 设置过期时间，从现在算起 &#43; 以上计算的时间。超时时间到后会删除一下几个key
		// 1) &#34;{rr_limiter::username}:value:***********&#34;
		// 2) &#34;{rr_limiter::username}:permits:***********&#34;
		// 3) &#34;rr_limiter::username&#34;
		rRateLimiter.expire(Instant.now().plusMillis(limitDuration));
	}
}

```

## 使用
```java
    @GetMapping(&#34;/info&#34;)
	@RequestRateLimit(rate = 2, rateInterval = 1, timeUnit = RateIntervalUnit.MINUTES) // 1 分钟允许请求 2 次
	public R getInfo() {
        // ...
    }
```

当用户请求 /info 接口的时候，redis 中就会存储一个 RATE_LIMITER::his_user_name::com.package.getInfo 这样的 key。当该用户在1分钟内请求该接口超过2次，那么他将会收到报错，并且 getInfo 方法并不会执行。

注意该注解无法作用于 @Cacheable 注释的方法上。

## 更多
我们可以实现一个自定义的频率限制，可以限制任意的方法，比如发送给运维人员的紧急邮件，如果同一主题发送过了，在5分钟内不要再次发送。

```java
public @interface CustomRateLimit {
    /**
     * key 的前缀，用于一组相同功能限流的标记
     * @return
     */
    String prefix() default &#34;&#34;;

    /**
     * 限流的 key，要求不为空，支持从参数中读取
     * @return
     */
    String key() default &#34;#key&#34;;

    /**
     * 限流模式,默认单机
     * @return
     */
    RateType type() default RateType.PER_CLIENT;

    /**
     * 限流速率，1次/分钟
     * @return
     */
    long rate() default 1;

    /**
     * 限流速率，每分钟
     * @return
     */
    long rateInterval() default 60 * 1000;

    /**
     * 限流速率单位
     * @return
     */
    RateIntervalUnit timeUnit() default RateIntervalUnit.MILLISECONDS;

}

public class CustomRateLimitAspect {

	private final RedissonClient redisson;
	/**
	 * 根据自定义注解获取切点
	 *
	 * @param CustomRateLimit 注解接口
	 */
	@Pointcut(&#34;@annotation(CustomRateLimit)&#34;)
	public void findAnnotationPointCut(CustomRateLimit CustomRateLimit) {
	}

	@Around(value = &#34;findAnnotationPointCut(customRateLimit)&#34;, argNames = &#34;joinPoint,customRateLimit&#34;)
	public Object around(ProceedingJoinPoint joinPoint, CustomRateLimit customRateLimit) throws Throwable {
		// 限流拦截器
		String key = getKey(joinPoint, customRateLimit);
		RRateLimiter limiter = getRateLimiter(customRateLimit, key);
		if (limiter.tryAcquire(1)) {
			return joinPoint.proceed();
		} else {
			log.info(&#34;skip method cause violate rate limit, key is {}&#34;, key);
			return R.failed(REACH_REQUEST_LIMIT, String.format(&#34;请求过于频繁，请于以下时间后重试：%s %s&#34;, customRateLimit.rateInterval(), customRateLimit.timeUnit().name().toLowerCase()));
		}
	}

	/**
	 * 获取限流拦截器
	 *
	 * @param limit  在要限流的方法上的配置
	 * @return 限流器
	 */
	private RRateLimiter getRateLimiter(CustomRateLimit limit, String key) {
		RRateLimiter rRateLimiter = redisson.getRateLimiter(CUSTOM_RATE_LIMITER_PREFIX &#43; &#34;::&#34; &#43; limit.prefix() &#43; &#34;::&#34; &#43; key);
		// 设置限流
		if (rRateLimiter.isExists()) {
			RateLimiterConfig existed = rRateLimiter.getConfig();
			// 判断配置是否更新，如果更新，重新加载限流器配置
			if (!Objects.equals(limit.rate(), existed.getRate())
					|| !Objects.equals(limit.timeUnit().toMillis(limit.rateInterval()), existed.getRateInterval())
					|| !Objects.equals(limit.type(), existed.getRateType())) {
				rRateLimiter.delete();
				rRateLimiter.trySetRate(limit.type(), limit.rate(), limit.rateInterval(), limit.timeUnit());
				expireByConfig(rRateLimiter, limit);
			}
		} else {
			rRateLimiter.trySetRate(limit.type(), limit.rate(), limit.rateInterval(), limit.timeUnit());
			expireByConfig(rRateLimiter, limit);
		}

		return rRateLimiter;
	}

	private void expireByConfig(RRateLimiter rRateLimiter, CustomRateLimit limit) {
		long limitDuration = limit.timeUnit().toMillis(limit.rateInterval()) &#43; 5000;
		rRateLimiter.expire(Instant.now().plusMillis(limitDuration));
	}

	// el表达式支持
	private String getKey(JoinPoint joinPoint, CustomRateLimit customRateLimit) {
		ExpressionParser expressionParser = new SpelExpressionParser();
		Expression expression = expressionParser.parseExpression(customRateLimit.key());
		CodeSignature methodSignature = (CodeSignature) joinPoint.getSignature();
		String[] sigParamNames = methodSignature.getParameterNames();
		EvaluationContext context = new StandardEvaluationContext();
		Object[] args = joinPoint.getArgs();
		for (int i = 0; i &lt; sigParamNames.length; i&#43;&#43;) {
			context.setVariable(sigParamNames[i], args[i]);
		}
		return (String) expression.getValue(context);
	}
}
```

### 使用
```java
	@Override
	@CustomRateLimit(prefix = Constants.Cache.EMAIL_RATE_LIMITER, rateInterval = 5, timeUnit = RateIntervalUnit.MINUTES) // 5分钟最多一次
	public void sendToMaintainersWithFrequencyLimit(String key, String subject, String... content) {
		sendToMaintainers(&#34;[紧急通知]&#34;,  subject, content);
	}
```


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/07/request-limit/  


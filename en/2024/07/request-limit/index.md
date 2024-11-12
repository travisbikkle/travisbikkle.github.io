# How to Limit Request Frequency in Java



Certainly we want to limit the frequency of user visits, because users could get angry, and angry people will click on our app like crazy.

They could also be very bad and use some crawler to try to bring down our server.

So how to do that?

In this article, we use springboot and log the user&#39;s information and access frequency into redis, you can also store it into memory or database.

## Think about this story, from first principles
The user may not be logged in, or may already be logged in.

If the user is logged in, we limit him based on the username, otherwise, we use IP or some unique device code. 

Ip is used in this article.

We want it to be simple enough to be used on multiple methods without writing additional code. So we are going to use interface cutouts.

## The @interface
```java
public @interface RequestRateLimit {

	/**
	 * the key, we use it to distinguish user request objects, like /info1 /info2 
	 * @return
	 */
	String key() default &#34;&#34;;

	/**
	 * just ignore this for now
	 * @return
	 */
	RateType type() default RateType.PER_CLIENT;

	/**
	 * frequency, like once per minute
	 * @return
	 */
	long rate() default 1;

	/**
	 * interval, like every one minute
	 * @return
	 */
	long rateInterval() default 60 * 1000;

	/**
	 * interval unit, like minute
	 * @return
	 */
	RateIntervalUnit timeUnit() default RateIntervalUnit.MILLISECONDS;

}

```

## The Aspect

You can just copy this code and test it.

```java
public class RequestRateLimitAspect {

	private RedissonClient redisson;
	private final UserService userService;
    
	@Pointcut(&#34;@annotation(RequestRateLimit)&#34;)
	public void findAnnotationPointCut(RequestRateLimit RequestRateLimit) {
	}

	@Around(value = &#34;findAnnotationPointCut(requestRateLimit)&#34;, argNames = &#34;joinPoint,requestRateLimit&#34;)
	public Object around(ProceedingJoinPoint joinPoint, RequestRateLimit requestRateLimit) throws Throwable {
		UserEntity user = userService.getCurrentRequestUser(); // just SecurityContextHolder.getContext().getAuthentication().getPrincipal();
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
			return R.failed(EMPTY_USER, &#34;login info not found&#34;);
		}
		// the real logic
		String key = user == null || StrUtil.isBlank(user.getUserName()) ? realIp : user.getUserName();
		key = key &#43; &#34;::&#34; &#43; joinPoint.getSignature().getName();
		RRateLimiter limiter = getRateLimiter(requestRateLimit, key);
		if (limiter.tryAcquire(1)) {
			return joinPoint.proceed();
		} else {
			log.info(&#34;rate-limit: {} {} {}&#34;, user == null ? &#34;&#34; : user.getUserName(), realIp, joinPoint.getSignature());
			return R.failed(REACH_REQUEST_LIMIT, String.format(&#34;too often, please retry after:%s %s&#34;, requestRateLimit.rateInterval(), requestRateLimit.timeUnit().name().toLowerCase()));
		}
	}

	private boolean notValidIp(String ip) {
		return StrUtil.isBlank(ip) || ip.startsWith(&#34;172.1&#34;); // docker bridge ip
	}

	private RRateLimiter getRateLimiter(RequestRateLimit limit, String defaultKey) {
		RRateLimiter rRateLimiter = redisson.getRateLimiter(StrUtil.isBlank(limit.key()) ? RATE_LIMITER &#43; &#34;::&#34; &#43; defaultKey : limit.key()); // RATE_LIMITER is a constant, change it to whatever you want
		if (rRateLimiter.isExists()) {
			RateLimiterConfig existed = rRateLimiter.getConfig();
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
		long limitDuration = limit.timeUnit().toMillis(limit.rateInterval()) &#43; 5000;
		rRateLimiter.expire(Instant.now().plusMillis(limitDuration));
	}
}

```

## Usage
```java
    @GetMapping(&#34;/info&#34;)
	@RequestRateLimit(rate = 2, rateInterval = 1, timeUnit = RateIntervalUnit.MINUTES) // Maximum of two requests per minute
	public R getInfo() {
        // ...
    }
```

When a user requests the /info method, a key such as `RATE_LIMITER::his_user_name::com.package.getInfo` is stored in redis. 

if the user requests the /info method more than twice in a minute, he will receive an error and the getInfo method will not execute.

Note that this annotation does not work on methods annotated with @Cacheable.

## More

We can implement a customized frequency limit that can restrict arbitrary methods, 
such as urgent emails sent to Ops staff, if the same subject has been sent, don&#39;t send it again within 5 minutes.

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

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/07/request-limit/  


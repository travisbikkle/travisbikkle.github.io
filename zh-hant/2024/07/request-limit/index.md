# 怎麼在 Java 中限制用戶訪問頻率


我們當然要限制用戶訪問頻率，因爲用戶可能生氣，並狂點我們的網站或應用。

他也可能很壞，使用一些爬蟲試圖拖垮我們的服務器。

所以怎麼實現呢？

本文使用 springboot，並將用戶的信息和訪問頻率記錄到 redis 中，如果你沒有使用 redis，也不影響，你可以參考着自己實現，比如存儲到內存或數據庫中。

## 想想這個需求，從第一性原理出發
用戶可能沒有登錄，或者已經登錄了。

如果用戶登錄了，我們就根據用戶名來限制，否則，就根據IP或者其它設備碼來限制，本文假設使用IP。

我們希望它足夠簡單，可以在多個方法上使用，而不需要編寫額外的代碼。所以我們要使用接口切面。

## 接口
```java
public @interface RequestRateLimit {

	/**
	 * 限流的key，比如限制用戶註冊，限制用戶發送郵件，等等，一般是方法名
	 * @return
	 */
	String key() default &#34;&#34;;

	/**
	 * 限流模式,默認單機
	 * @return
	 */
	RateType type() default RateType.PER_CLIENT;

	/**
	 * 限流速率，1次/分鐘
	 * @return
	 */
	long rate() default 1;

	/**
	 * 限流速率，每分鐘
	 * @return
	 */
	long rateInterval() default 60 * 1000;

	/**
	 * 限流速率單位
	 * @return
	 */
	RateIntervalUnit timeUnit() default RateIntervalUnit.MILLISECONDS;

}

```

## 切面
你可以直接拷貝這些代碼並測試。

```java
public class RequestRateLimitAspect {

	private RedissonClient redisson;
	private final UserService userService;

	/**
	 * 根據自定義註解獲取切點
	 *
	 * @param RequestRateLimit 註解接口
	 */
	@Pointcut(&#34;@annotation(RequestRateLimit)&#34;)
	public void findAnnotationPointCut(RequestRateLimit RequestRateLimit) {
	}

	@Around(value = &#34;findAnnotationPointCut(requestRateLimit)&#34;, argNames = &#34;joinPoint,requestRateLimit&#34;)
	public Object around(ProceedingJoinPoint joinPoint, RequestRateLimit requestRateLimit) throws Throwable {
		UserEntity user = userService.getCurrentRequestUser(); // 只是封裝了 SecurityContextHolder.getContext().getAuthentication().getPrincipal();
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
			return R.failed(EMPTY_USER, &#34;未找到您的任何登錄信息&#34;);
		}
		// 限流攔截器
		String key = user == null || StrUtil.isBlank(user.getUserName()) ? realIp : user.getUserName();
		key = key &#43; &#34;::&#34; &#43; joinPoint.getSignature().getName();
		RRateLimiter limiter = getRateLimiter(requestRateLimit, key);
		if (limiter.tryAcquire(1)) {
			return joinPoint.proceed();
		} else {
			log.info(&#34;rate-limit: {} {} {}&#34;, user == null ? &#34;&#34; : user.getUserName(), realIp, joinPoint.getSignature());
			return R.failed(REACH_REQUEST_LIMIT, String.format(&#34;請求過於頻繁，請於以下時間後重試：%s %s&#34;, requestRateLimit.rateInterval(), requestRateLimit.timeUnit().name().toLowerCase()));
		}
	}

	private boolean notValidIp(String ip) {
		return StrUtil.isBlank(ip) || ip.startsWith(&#34;172.1&#34;); // docker bridge ip
	}

	/**
	 * 獲取限流攔截器
	 *
	 * @param limit  在要限流的方法上的配置
	 * @param defaultKey 在redis中的存儲的key
	 * @return 限流器
	 */
	private RRateLimiter getRateLimiter(RequestRateLimit limit, String defaultKey) {
		RRateLimiter rRateLimiter = redisson.getRateLimiter(StrUtil.isBlank(limit.key()) ? RATE_LIMITER &#43; &#34;::&#34; &#43; defaultKey : limit.key()); // RATE_LIMITER 隨意起名，比如可以使用你的項目名稱，只是爲了在redis中好區分
		// 設置限流
		if (rRateLimiter.isExists()) {
			RateLimiterConfig existed = rRateLimiter.getConfig();
			// 判斷配置是否更新，如果更新，重新加載限流器配置
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
		// ttl 設置爲 rateLimit 配置時間 &#43; 5s
		long limitDuration = limit.timeUnit().toMillis(limit.rateInterval()) &#43; 5000;
		// 設置過期時間，從現在算起 &#43; 以上計算的時間。超時時間到後會刪除一下幾個key
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
	@RequestRateLimit(rate = 2, rateInterval = 1, timeUnit = RateIntervalUnit.MINUTES) // 1 分鐘允許請求 2 次
	public R getInfo() {
        // ...
    }
```

當用戶請求 /info 接口的時候，redis 中就會存儲一個 RATE_LIMITER::his_user_name::com.package.getInfo 這樣的 key。當該用戶在1分鐘內請求該接口超過2次，那麼他將會收到報錯，並且 getInfo 方法並不會執行。

注意該註解無法作用於 @Cacheable 註釋的方法上。

## 更多
我們可以實現一個自定義的頻率限制，可以限制任意的方法，比如發送給運維人員的緊急郵件，如果同一主題發送過了，在5分鐘內不要再次發送。

```java
public @interface CustomRateLimit {
    /**
     * key 的前綴，用於一組相同功能限流的標記
     * @return
     */
    String prefix() default &#34;&#34;;

    /**
     * 限流的 key，要求不爲空，支持從參數中讀取
     * @return
     */
    String key() default &#34;#key&#34;;

    /**
     * 限流模式,默認單機
     * @return
     */
    RateType type() default RateType.PER_CLIENT;

    /**
     * 限流速率，1次/分鐘
     * @return
     */
    long rate() default 1;

    /**
     * 限流速率，每分鐘
     * @return
     */
    long rateInterval() default 60 * 1000;

    /**
     * 限流速率單位
     * @return
     */
    RateIntervalUnit timeUnit() default RateIntervalUnit.MILLISECONDS;

}

public class CustomRateLimitAspect {

	private final RedissonClient redisson;
	/**
	 * 根據自定義註解獲取切點
	 *
	 * @param CustomRateLimit 註解接口
	 */
	@Pointcut(&#34;@annotation(CustomRateLimit)&#34;)
	public void findAnnotationPointCut(CustomRateLimit CustomRateLimit) {
	}

	@Around(value = &#34;findAnnotationPointCut(customRateLimit)&#34;, argNames = &#34;joinPoint,customRateLimit&#34;)
	public Object around(ProceedingJoinPoint joinPoint, CustomRateLimit customRateLimit) throws Throwable {
		// 限流攔截器
		String key = getKey(joinPoint, customRateLimit);
		RRateLimiter limiter = getRateLimiter(customRateLimit, key);
		if (limiter.tryAcquire(1)) {
			return joinPoint.proceed();
		} else {
			log.info(&#34;skip method cause violate rate limit, key is {}&#34;, key);
			return R.failed(REACH_REQUEST_LIMIT, String.format(&#34;請求過於頻繁，請於以下時間後重試：%s %s&#34;, customRateLimit.rateInterval(), customRateLimit.timeUnit().name().toLowerCase()));
		}
	}

	/**
	 * 獲取限流攔截器
	 *
	 * @param limit  在要限流的方法上的配置
	 * @return 限流器
	 */
	private RRateLimiter getRateLimiter(CustomRateLimit limit, String key) {
		RRateLimiter rRateLimiter = redisson.getRateLimiter(CUSTOM_RATE_LIMITER_PREFIX &#43; &#34;::&#34; &#43; limit.prefix() &#43; &#34;::&#34; &#43; key);
		// 設置限流
		if (rRateLimiter.isExists()) {
			RateLimiterConfig existed = rRateLimiter.getConfig();
			// 判斷配置是否更新，如果更新，重新加載限流器配置
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

	// el表達式支持
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
	@CustomRateLimit(prefix = Constants.Cache.EMAIL_RATE_LIMITER, rateInterval = 5, timeUnit = RateIntervalUnit.MINUTES) // 5分鐘最多一次
	public void sendToMaintainersWithFrequencyLimit(String key, String subject, String... content) {
		sendToMaintainers(&#34;[緊急通知]&#34;,  subject, content);
	}
```


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/07/request-limit/  


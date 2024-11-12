# Oauth2 限制登录一个客户端


有时候我们希望用户只能在一台设备登录账号（我们太吝啬了）。

使用 springboot oauth2 怎么实现呢？

注意本文不会带你使用 spring security 实现 oauth2 登录，仅仅是讨论我们那个吝啬的需求。

假设我们有这样一个自定义的认证实现类：

```java
public class RedisOAuth2AuthorizationService implements OAuth2AuthorizationService {

	private final static Long TIMEOUT = 10L;

	private static final String AUTHORIZATION = &#34;token&#34;;

	private final RedisTemplate&lt;String, Object&gt; redisTemplate;

	@Override
	public void save(OAuth2Authorization authorization) {
        // is refresh token mode or code mode
        // ...
        // is access token mode
		if (isAccessToken(authorization)) {
			OAuth2AccessToken accessToken = authorization.getAccessToken().getToken();
			long between = ChronoUnit.SECONDS.between(accessToken.getIssuedAt(), accessToken.getExpiresAt());
			redisTemplate.setValueSerializer(RedisSerializer.java());
			redisTemplate.opsForValue()
				.set(buildKey(OAuth2ParameterNames.ACCESS_TOKEN, accessToken.getTokenValue()), authorization, between,
						TimeUnit.SECONDS);
		}
	}

	@Override
	public void remove(OAuth2Authorization authorization) {
        // is refresh token mode or code mode
        // ...
        // is access token mode
		if (isAccessToken(authorization)) {
			OAuth2AccessToken accessToken = authorization.getAccessToken().getToken();
			keys.add(buildKey(OAuth2ParameterNames.ACCESS_TOKEN, accessToken.getTokenValue()));
		}
		redisTemplate.delete(keys);
	}
	@Override
	@Nullable
	public OAuth2Authorization findByToken(String token, @Nullable OAuth2TokenType tokenType) {
		Assert.hasText(token, &#34;token cannot be empty&#34;);
		Assert.notNull(tokenType, &#34;tokenType cannot be empty&#34;);
		redisTemplate.setValueSerializer(RedisSerializer.java());
		return (OAuth2Authorization) redisTemplate.opsForValue().get(buildKey(tokenType.getValue(), token));
	}

    private String buildKey(String type, String id) {
        return String.format(&#34;%s::%s::%s&#34;, AUTHORIZATION, type, id);
    }
    // ...
}
```

它将 token 存储到 redis 中。key 是下面这种格式：
```text
token::access_token::xxxxxxxxxxxxxxxxxxxxxxxx
```

当用户登录的时候，findByToken 会被 spring security 调用，从而找出用户信息。

因为我们很吝啬，我们希望一个用户只能登录一次，也就是说，一个用户只有一个 token。

redis 的 key 不能再使用 token 了，而应该改成用户名。

```text
token::access_token::that_annoying_user
```

但是这样 findByToken 又如何根据 token 找出用户名呢？采用这种 key，我们就没有一个效率比较高的方法，能够在数百万用户中找出该用户。

那么这样的 key 怎么样？

```text
token::access_token::xxxxxxxxxxxxxxxxxxxxxxxx::that_annoying_user
```
既有 token，又有用户信息。

假设用户已经登录，获得了 `xxxxxxxxxxxxxxxxxxxxxxxx` 的 token。

他又来登录我们的应用了，我们让他登录前，找出该用户名下的旧的 token，并删除。

```text
keys token::access_token::*::that_annoying_user
```

这样配合应用中的检查 token，就可以踢出他原来的登录会话了。

```java
public class RedisOAuth2AuthorizationService implements OAuth2AuthorizationService {
	@Override
	public void save(OAuth2Authorization authorization) {
        // is refresh token mode or code mode
        // ...
        // is access token mode
		if (isAccessToken(authorization)) {
			OAuth2AccessToken accessToken = authorization.getAccessToken().getToken();
			long between = ChronoUnit.SECONDS.between(accessToken.getIssuedAt(), accessToken.getExpiresAt());
			redisTemplate.setValueSerializer(RedisSerializer.java());

			// 删除该用户的旧的 token
			// token::access_token::*::userName
            // 确保你的用户名不允许 * 的存在
			Set&lt;String&gt; keys  = redisTemplate.keys(buildKey(OAuth2ParameterNames.ACCESS_TOKEN, &#34;*&#34;, authorization.getPrincipalName()));
			if (! CollectionUtils.isEmpty(keys)) {
				redisTemplate.delete(keys);
			}

			redisTemplate.opsForValue()
				.set(buildKey(OAuth2ParameterNames.ACCESS_TOKEN, accessToken.getTokenValue(), authorization.getPrincipalName()), authorization, between,
						TimeUnit.SECONDS);
		}
	}

	@Override
	public void remove(OAuth2Authorization authorization) {
        // is refresh token mode or code mode
        // ...
        // is access token mode
		if (isAccessToken(authorization)) {
			OAuth2AccessToken accessToken = authorization.getAccessToken().getToken();
			keys.add(buildKey(OAuth2ParameterNames.ACCESS_TOKEN, accessToken.getTokenValue(), authorization.getPrincipalName()));
		}
		redisTemplate.delete(keys);
	}

	@Override
	@Nullable
	public OAuth2Authorization findById(String id) {
		throw new UnsupportedOperationException();
	}

	@Override
	@Nullable
	public OAuth2Authorization findByToken(String token, @Nullable OAuth2TokenType tokenType) {
		Assert.hasText(token, &#34;token cannot be empty&#34;);
		Assert.notNull(tokenType, &#34;tokenType cannot be empty&#34;);
		redisTemplate.setValueSerializer(RedisSerializer.java());

		// token::access_token::tokenValue::*
		Set&lt;String&gt; keys = redisTemplate.keys(buildKey(tokenType.getValue(), token, &#34;*&#34;));
		if (CollectionUtils.isEmpty(keys)) {
			return null;
		}

		List&lt;Object&gt; saved = redisTemplate.opsForValue().multiGet(keys);
		if (CollectionUtils.isEmpty(saved)) {
			return null;
		}

		return (OAuth2Authorization) saved.get(0);
	}

    private String buildKey(String type, String id, String principle) { // 增加了用户名
        return String.format(&#34;%s::%s::%s::%s&#34;, AUTHORIZATION, type, id, principle);
    }
    // ...
}
```



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/07/oauth2-single-login/  


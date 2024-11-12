# Oauth2 Single Login


Sometimes we want users to be able to log in to their accounts on only one device (we&#39;re too stingy).

How can we achieve this using springboot security and oauth2?

Note that this article will not take you through the implementation of oauth2 login using spring security, but merely discuss our miserly requirement.

Suppose we have a custom authentication implementation class like this:

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

It stores the token in redis. The key is in the following format:

```text
token::access_token::xxxxxxxxxxxxxxxxxxxxxxxx
```

When a user logs in, findByToken is called by spring security to find out information about the user.

Because we&#39;re stingy, we want a user to log in only once, i.e., only one token per user.

The redis key should no longer use a token, but rather a username;

```text
token::access_token::that_annoying_user
```

But how does findByToken find the username based on the token? 

With this kind of key, we don&#39;t have an efficient way to find the user out of millions of users.

What about a key like this?

```text
token::access_token::xxxxxxxxxxxxxxxxxxxxxxxx::that_annoying_user
```

We got the token, and the username.

Let&#39;s say a user has logged in and got the token `xxxxxxxxxxxxxxxxxxxxxxxx.

He wants to log into our application again, that&#39;s fine. We just find the old token under that username and delete it(them) before logging him in.

```text
keys token::access_token::*::that_annoying_user
```

By that, in conjunction with checking for tokens in the app, he will be kicked out of his original login session.

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

            // delete old tokens of this user
            // make sure * or :: or other special character is not allowed in the username
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

> Author: [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/en/2024/07/oauth2-single-login/  


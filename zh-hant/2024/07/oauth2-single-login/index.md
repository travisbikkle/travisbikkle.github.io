# Oauth2 限制登錄一個客戶端


有時候我們希望用戶只能在一臺設備登錄賬號（我們太吝嗇了）。

使用 springboot oauth2 怎麼實現呢？

注意本文不會帶你使用 spring security 實現 oauth2 登錄，僅僅是討論我們那個吝嗇的需求。

假設我們有這樣一個自定義的認證實現類：

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

它將 token 存儲到 redis 中。key 是下面這種格式：
```text
token::access_token::xxxxxxxxxxxxxxxxxxxxxxxx
```

當用戶登錄的時候，findByToken 會被 spring security 調用，從而找出用戶信息。

因爲我們很吝嗇，我們希望一個用戶只能登錄一次，也就是說，一個用戶只有一個 token。

redis 的 key 不能再使用 token 了，而應該改成用戶名。

```text
token::access_token::that_annoying_user
```

但是這樣 findByToken 又如何根據 token 找出用戶名呢？採用這種 key，我們就沒有一個效率比較高的方法，能夠在數百萬用戶中找出該用戶。

那麼這樣的 key 怎麼樣？

```text
token::access_token::xxxxxxxxxxxxxxxxxxxxxxxx::that_annoying_user
```
既有 token，又有用戶信息。

假設用戶已經登錄，獲得了 `xxxxxxxxxxxxxxxxxxxxxxxx` 的 token。

他又來登錄我們的應用了，我們讓他登錄前，找出該用戶名下的舊的 token，並刪除。

```text
keys token::access_token::*::that_annoying_user
```

這樣配合應用中的檢查 token，就可以踢出他原來的登錄會話了。

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

			// 刪除該用戶的舊的 token
			// token::access_token::*::userName
            // 確保你的用戶名不允許 * 的存在
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

    private String buildKey(String type, String id, String principle) { // 增加了用戶名
        return String.format(&#34;%s::%s::%s::%s&#34;, AUTHORIZATION, type, id, principle);
    }
    // ...
}
```



---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/07/oauth2-single-login/  


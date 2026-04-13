```
public class AccessToken extends BaseToken {
    private RefreshToken refreshToken;

    public AccessToken() {
        super();
        this.refreshToken = new RefreshToken();
    }
}
```

```
public class BaseToken implements Serializable {
    protected String token;

    protected BaseToken() {
        this.token = RandomStringUtils.random(ConstantPool.TOKEN_LENGTH, ConstantPool.TOKEN_CHARS);
    }
}
```

```
public class RefreshToken extends BaseToken {
    public RefreshToken() {
        super();
    }
}
```

```java
public class TokenService {
    private static final Integer REDIS_DELETE_RETRY_TIMES = 3;

    @Qualifier("redis-token-template")
    private final RedisTemplate<String, Object> redisTemplate;
    private final PlatformRepository platformRepository;
    private final AccountRepository accountRepository;

    @RemotePropertyValue("enable-super-token")
    private Boolean enableSuperToken;
    @RemotePropertyValue("access-token-expiration")
    private Duration accessTokenExpiration;
    @RemotePropertyValue("refresh-token-expiration")
    private Duration refreshTokenExpiration;

    public List<AuthenticatedUser> getAuthenticatedUsers(Long accountId) {
        String accountAccessTokenKey = RedisUtils.computeKey(ConstantPool.ACCOUNT_ACCESS_TOKEN_KEY_PREFIX, accountId.toString());
       Set<Object> accessTokens = SetUtils.emptyIfNull(redisTemplate.opsForSet().members(accountAccessTokenKey));
       Set<String> keys = accessTokens.stream()
                .map(accessToken -> RedisUtils.computeKey(ConstantPool.ACCESS_TOKEN_ACCOUNT_KEY_PREFIX, ((AccessToken) accessToken).getToken()))
             .filter(key -> BooleanUtils.isTrue(redisTemplate.hasKey(key)))
             .collect(Collectors.toSet());
       return ListUtils.emptyIfNull(redisTemplate.opsForValue().multiGet(keys))
             .stream()
             .filter(Objects::nonNull)
             .map(AuthenticatedUser.class::cast)
             .toList();
    }

    public AccessToken generateToken(AccountDO accountDO, PlatformDO platformDO, LoginMode loginMode, boolean superToken) {
       AccessToken accessToken = new AccessToken();

       AuthenticatedUser authenticatedUser = new AuthenticatedUser();
       authenticatedUser.setId(accountDO.getId());
       authenticatedUser.setPlatformId(platformDO.getId());
       authenticatedUser.setPlatform(platformDO.getName());
       authenticatedUser.setToken(accessToken);
       authenticatedUser.setLoginMode(loginMode);

        Date loginTime = new Date();
       authenticatedUser.setLoginTime(loginTime);
        authenticatedUser.setExpireTime(DateUtils.addMinutes(loginTime, (int) accessTokenExpiration.toMinutes()));

        if (!platformDO.getMultiDeviceLogin()) {
            removeToken(accountDO.getId());
        }
       if (!(enableSuperToken && superToken)) {
          storeRefreshToken(accessToken.getRefreshToken(), authenticatedUser);
       }
       storeAccessToken(accessToken, authenticatedUser, superToken);

       return accessToken;
    }

    public String generateCode(String token) {
       token = StringUtils.substringAfter(token, ConstantPool.TOKEN_PREFIX);
       AuthenticatedUser authenticatedUser = getAuthenticatedUser(token);
        if (Objects.isNull(authenticatedUser)) {
            throw new InvalidTokenException();
        }

        String code = RandomStringUtils.random(16, ConstantPool.TOKEN_CHARS);
        String storeKey = RedisUtils.computeKey(ConstantPool.CODE_ACCOUNT_KEY_PREFIX, code);
       redisTemplate.opsForValue().set(storeKey, authenticatedUser.getId(), accessTokenExpiration.getSeconds(), TimeUnit.SECONDS);
       return code;
    }

    public AccessToken generateToken(String code, boolean superToken) {
        String codeKey = RedisUtils.computeKey(ConstantPool.CODE_ACCOUNT_KEY_PREFIX, code);
       Long accountId = (Long) redisTemplate.opsForValue().get(codeKey);
       if (Objects.isNull(accountId)) {
            throw new AuthenticationExpireException("无效的token");
       }
       redisTemplate.delete(codeKey);

       AccountDO accountDO = accountRepository.getById(accountId);
       PlatformDO platformDO = platformRepository.getById(accountDO.getPlatformId());
       if (!platformDO.getMultiDeviceLogin()) {
          removeToken(accountId);
            String accountTokenStoreKey = RedisUtils.computeKey(ConstantPool.ACCOUNT_ACCESS_TOKEN_KEY_PREFIX, accountId.toString());
          Set<Object> accessTokens = redisTemplate.opsForSet().members(accountTokenStoreKey);
          if (CollectionUtils.isEmpty(accessTokens)) {
             return generateToken(accountDO, platformDO, LoginMode.CODE, superToken);
          }
          return (AccessToken) accessTokens.iterator().next();
       }
       return generateToken(accountDO, platformDO, LoginMode.CODE, superToken);
    }

    public AuthenticatedUser getAuthenticatedUser(String accessToken) {
        String key = RedisUtils.computeKey(ConstantPool.ACCESS_TOKEN_ACCOUNT_KEY_PREFIX, accessToken);
        return (AuthenticatedUser) redisTemplate.opsForValue().get(key);
    }

    public void removeToken(String accessToken) {
       accessToken = StringUtils.substringAfter(accessToken, ConstantPool.TOKEN_PREFIX);
       AuthenticatedUser authenticatedUser = getAuthenticatedUser(accessToken);
        if (Objects.nonNull(authenticatedUser)) {
            removeToken(authenticatedUser);
        }
    }

    public void removeToken(AuthenticatedUser authenticatedUser) {
        String accountTokenKey = RedisUtils.computeKey(ConstantPool.ACCOUNT_ACCESS_TOKEN_KEY_PREFIX, authenticatedUser.getId().toString());
        String accessTokenKey = RedisUtils.computeKey(ConstantPool.ACCESS_TOKEN_ACCOUNT_KEY_PREFIX, authenticatedUser.getToken().getToken());
        String refreshTokenKey = RedisUtils.computeKey(ConstantPool.REFRESH_TOKEN_ACCOUNT_KEY_PREFIX, authenticatedUser.getToken().getRefreshToken().getToken());

        redisTemplate.delete(Set.of(accessTokenKey, refreshTokenKey));
        redisTemplate.opsForSet().remove(accountTokenKey, authenticatedUser.getToken());
    }

    public void removeToken(Long accountId) {
       Set<String> keys = getKeys(accountId);
        deleteRedisKeys(keys);
    }

    public void removeTokens(List<Long> accountIds) {
       if (accountIds.isEmpty()) {
          return;
       }
       Set<String> keys = ListUtils.emptyIfNull(accountIds)
             .stream()
             .map(this::getKeys)
             .flatMap(Set::stream)
             .collect(Collectors.toSet());
        deleteRedisKeys(keys);
    }

    public AccessToken refreshToken(String refreshToken) {
        if (!RegExUtils.matches(TokenValidator.PATTERN, refreshToken)) {
            throw new AuthenticationExpireException("无效的token");
        }
        refreshToken = StringUtils.substringAfter(refreshToken, ConstantPool.TOKEN_PREFIX);
        String refreshTokenKey = RedisUtils.computeKey(ConstantPool.REFRESH_TOKEN_ACCOUNT_KEY_PREFIX, refreshToken);
       AuthenticatedUser authenticatedUser = (AuthenticatedUser) redisTemplate.opsForValue().get(refreshTokenKey);
       if (Objects.isNull(authenticatedUser)) {
            throw new AuthenticationExpireException("无效的token");
       }

       AccountDO accountDO = accountRepository.getById(authenticatedUser.getId());
       PlatformDO platformDO = platformRepository.getById(authenticatedUser.getPlatformId());

       redisTemplate.delete(refreshTokenKey);
       String accessTokenKey = RedisUtils.computeKey(ConstantPool.ACCESS_TOKEN_ACCOUNT_KEY_PREFIX, authenticatedUser.getToken().getToken());
       redisTemplate.delete(accessTokenKey);
       String accountTokenKey = RedisUtils.computeKey(ConstantPool.ACCOUNT_ACCESS_TOKEN_KEY_PREFIX, authenticatedUser.getId().toString());
       redisTemplate.opsForSet().remove(accountTokenKey, authenticatedUser.getToken());

       return generateToken(accountDO, platformDO, authenticatedUser.getLoginMode(), false);
    }

    private void storeAccessToken(AccessToken baseToken, AuthenticatedUser authenticatedUser, boolean superToken) {
        String tokenSecurityAccountKey = RedisUtils.computeKey(ConstantPool.ACCESS_TOKEN_ACCOUNT_KEY_PREFIX, baseToken.getToken());
       if (enableSuperToken && superToken) {
          redisTemplate.opsForValue().set(tokenSecurityAccountKey, authenticatedUser);
       } else {
          redisTemplate.opsForValue().set(tokenSecurityAccountKey, authenticatedUser, accessTokenExpiration.getSeconds(), TimeUnit.SECONDS);
       }
       redisTemplate.opsForSet().add(RedisUtils.computeKey(ConstantPool.ACCOUNT_ACCESS_TOKEN_KEY_PREFIX, authenticatedUser.getId().toString()), baseToken);
    }

    private void storeRefreshToken(RefreshToken baseToken, AuthenticatedUser authenticatedUser) {
        String tokenSecurityAccountKey = RedisUtils.computeKey(ConstantPool.REFRESH_TOKEN_ACCOUNT_KEY_PREFIX, baseToken.getToken());
       redisTemplate.opsForValue().set(tokenSecurityAccountKey, authenticatedUser, refreshTokenExpiration.getSeconds(), TimeUnit.SECONDS);
    }

    private Set<String> getKeys(Long accountId) {
        String accountTokenKey = RedisUtils.computeKey(ConstantPool.ACCOUNT_ACCESS_TOKEN_KEY_PREFIX, accountId.toString());
       Set<String> tokenKeys = SetUtils.emptyIfNull(redisTemplate.opsForSet().members(accountTokenKey))
             .stream()
             .map(token -> {
                AccessToken accessToken = (AccessToken) token;
                    String accessTokenKey = RedisUtils.computeKey(ConstantPool.ACCESS_TOKEN_ACCOUNT_KEY_PREFIX, accessToken.getToken());
                    String refreshTokenKey = RedisUtils.computeKey(ConstantPool.REFRESH_TOKEN_ACCOUNT_KEY_PREFIX, accessToken.getRefreshToken().getToken());
                return Set.of(accessTokenKey, refreshTokenKey);
             })
             .flatMap(Set::stream)
             .collect(Collectors.toSet());
       tokenKeys.add(accountTokenKey);
       return tokenKeys;
    }

    private void deleteRedisKeys(Collection<String> keys) {
        if (CollectionUtils.isNotEmpty(keys)) {
          long deleteCount = ObjectUtils.defaultIfNull(redisTemplate.delete(keys), 0L);
            if (deleteCount < keys.size()) {
                long count = keys.size() - deleteCount;
                long times = 0;
                while (times <= REDIS_DELETE_RETRY_TIMES && count > 0) {
                long tmpCount = redisTemplate.delete(keys);
                ++times;
                count -= tmpCount;
                deleteCount += tmpCount;
             }
            }
        }
    }
}
```
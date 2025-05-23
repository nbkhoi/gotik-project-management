# Login Rate Limiting

This document describes the implementation of login rate limiting in the API Gateway to prevent brute force attacks.

## Overview

The login rate limiting feature restricts the number of failed login attempts per username within a specified time window. After exceeding the maximum number of attempts, the user is temporarily locked out.

## Implementation

### Configuration

- Maximum allowed failed attempts: 5
- Lockout period: 15 minutes
- In-memory caching with Caffeine

```java
// Configuration constants in LoginRateLimitFilter
private static final int MAX_ATTEMPTS = 5;
private static final int LOCKOUT_MINUTES = 15;

// Cache configuration
private final Cache<String, AtomicInteger> attemptCache = Caffeine.newBuilder()
    .expireAfterWrite(24, TimeUnit.HOURS)
    .build();
    
private final Cache<String, LocalDateTime> lockCache = Caffeine.newBuilder()
    .expireAfterWrite(LOCKOUT_MINUTES, TimeUnit.MINUTES)
    .build();
```

### Components

1. **LoginRateLimitFilter**: Implements the rate limiting mechanism as a Spring Cloud Gateway filter.
2. **CognitoAuthService Integration**: Records failed attempts and resets counters on successful authentication.

### Filter Operation

1. The filter intercepts all requests to the `/auth/login` endpoint.
2. It extracts the username from the request body.
3. For failed login attempts, it increments a counter in the cache.
4. If the counter reaches the maximum allowed attempts, the user is locked out.
5. During lockout, the filter returns a 429 Too Many Requests response with a Retry-After header.

```java
// Rate limiting response in LoginRateLimitFilter
if (username != null && isRateLimited(username)) {
    // Return 429 Too Many Requests
    ServerHttpResponse response = exchange.getResponse();
    response.setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
    response.getHeaders().add("Retry-After", String.valueOf(LOCKOUT_MINUTES * 60));
    
    try {
        String errorJson = String.format(
            "{\"error\":\"RATE_LIMITED\",\"message\":\"Too many login attempts. Please try again later.\",\"retryAfterSeconds\":%d}",
            LOCKOUT_MINUTES * 60
        );
        byte[] bytes = errorJson.getBytes(StandardCharsets.UTF_8);
        DataBuffer buffer = response.bufferFactory().wrap(bytes);
        return response.writeWith(Mono.just(buffer));
    } catch (Exception e) {
        log.error("Error writing rate limit response", e);
        return response.setComplete();
    }
}

### Integration Points

1. **API Gateway Configuration**: The filter is applied to the `/auth/login` route.
2. **Authentication Service**: Records failed login attempts in the CognitoAuthService class.

```java
// Integration with CognitoAuthService
@Override
public InitiateAuthResponse signIn(String username, String password, PortalType portalType) {
    try {
        // Set up authentication parameters
        // ...
        
        // Successful login
        InitiateAuthResponse response = getCognitoClient(portalType).initiateAuth(authRequest);
        // Reset any failed attempts upon successful login
        loginRateLimitFilter.resetAttempts(username);
        return response;
    } catch (NotAuthorizedException e) {
        // Record failed login attempt
        loginRateLimitFilter.recordFailedAttempt(username);
        throw new CognitoAuthException(ResponseCode.AUTH_INVALID_CREDENTIALS);
    } catch (/* Other exceptions */) {
        // Other exception handling
    }
}
```

3. **Gateway Route Configuration**: The login rate limiting filter is applied specifically to the login endpoint in `GatewayConfig.java`:

```java
@Bean
public RouteLocator authRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
            .route("auth-login", r -> r
                    .path("/auth/login")
                    .filters(f -> f
                            .filter(loginRateLimitFilter.apply(new LoginRateLimitFilter.Config())))
                    .uri(serviceLoadBalanceProperties.getAuthService()))
            // Other routes
            .build();
}
```

## Monitoring

Metrics are collected for:

- Failed login attempts (`auth.login.failed`)
- Rate limit activations (`auth.rate_limit.triggered`)
- Rate limit blocks (`auth.rate_limit.blocked`)
- Counter resets (`auth.rate_limit.reset`)

## Security Considerations

1. The rate limiting is based on the username (email) to prevent password guessing attacks.
2. The filter applies before authentication to prevent resource exhaustion attacks.
3. All rate limiting actions are logged for security monitoring.
4. Rate limiting information is stored in memory and expires automatically after the specified period.

## Testing

Tests include:
1. Unit tests for the LoginRateLimitFilter
2. Integration tests with actual authentication flow
3. Load tests to verify behavior under concurrent attempts
4. Security tests to verify effectiveness against brute force attempts

## Future Enhancements

1. Replace in-memory caching with a distributed cache for cluster support
2. Add progressive lockout periods for repeat offenders
3. Implement IP-based rate limiting in addition to username-based limiting
4. Add administrative interface for managing locked accounts

## Implementation Summary

The login rate limiting feature has been successfully implemented with the following components:

1. **LoginRateLimitFilter**: A Gateway filter that tracks failed login attempts and blocks excessive attempts.
2. **CognitoAuthService Integration**: The authentication service now records failed attempts and resets counters.
3. **API Gateway Configuration**: The filter is properly applied to the login endpoint.
4. **Rate Limiting Logic**: Implements a simple counter-based rate limiting with appropriate timeouts.
5. **Error Responses**: Returns standardized 429 Too Many Requests responses with Retry-After headers.
6. **Metrics**: Captures metrics for monitoring failed attempts, blocks, and resets.
7. **Unit Tests**: Test cases created for the LoginRateLimitFilter to verify functionality.
8. **Documentation**: Comprehensive documentation provided for future reference and maintenance.

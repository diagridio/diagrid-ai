# Java

Package: `io.diagrid:diagrid-spring-ai-identity` on Maven Central, 0.4.0 or later, on Java
17 or later. 0.4.0 is built for Spring Boot 4.0 (Spring 7, Jakarta servlet). It brings the
core identity module and its one dependency, `nimbus-jose-jwt`. Use the latest release
and put that version in the `pom.xml`:

```xml
<dependency>
  <groupId>io.diagrid</groupId>
  <artifactId>diagrid-spring-ai-identity</artifactId>
  <version>0.4.0</version>
</dependency>
<dependency>
  <groupId>io.modelcontextprotocol.sdk</groupId>
  <artifactId>mcp</artifactId>
  <version>2.0.0</version>
</dependency>
```

No BOM manages `mcp`, so give its version. 2.0.0 is the one Spring AI 2.0.0 is built
against.

## Verify the caller

A servlet filter. There is no auto-configuration, so declare it as a bean and Spring Boot
registers it:

```java
import io.diagrid.ai.identity.OAuthConfig;
import io.diagrid.springai.identity.OAuthFilter;

@Bean
OAuthFilter diagridOAuthFilter() {
  return new OAuthFilter(new OAuthConfig());   // issuer, audience and keys are discovered
}
```

In a controller:

```java
import io.diagrid.ai.identity.VerifiedUser;

VerifiedUser user = OAuthFilter.verifiedUser(request).orElseThrow();
```

`VerifiedUser` has `subject()`, `tenant()`, `scopes()`, `claims()` and `issuerId()`.

## Call MCP tools as the caller

The filter keeps the caller's token in a `ThreadLocal`. The MCP Java SDK sends from its
own threads, where that `ThreadLocal` is empty. So read the token on the request thread
and hand it to the sending thread through the MCP transport context:

```java
import io.diagrid.ai.identity.IdentityContext;
import io.modelcontextprotocol.client.McpClient;
import io.modelcontextprotocol.client.McpSyncClient;
import io.modelcontextprotocol.client.transport.HttpClientStreamableHttpTransport;
import io.modelcontextprotocol.common.McpTransportContext;
import java.util.Map;

static final String TOKEN_KEY = "diagrid.userToken";

McpSyncClient connect() {
  var transport = HttpClientStreamableHttpTransport.builder(System.getenv("DAPR_HTTP_ENDPOINT"))
      .endpoint("/v1.0/diagrid/mcp/crm-mcp")
      .httpRequestCustomizer((request, method, uri, body, context) -> {
        request.header("dapr-api-token", System.getenv("DAPR_API_TOKEN"));
        Object token = context == null ? null : context.get(TOKEN_KEY);
        if (token instanceof String raw && !raw.isEmpty()) {
          request.header(IdentityContext.USER_TOKEN_HEADER, IdentityContext.BEARER_PREFIX + raw);
        }
      })
      .build();
  McpSyncClient client = McpClient.sync(transport)
      .transportContextProvider(() -> {
        String token = IdentityContext.currentUserToken();   // runs on the request thread
        return token == null || token.isEmpty()
            ? McpTransportContext.EMPTY
            : McpTransportContext.create(Map.of(TOKEN_KEY, token));
      })
      .build();
  client.initialize();
  return client;
}
```

This is the one place the header is set in app code, because the identity package cannot
reach the MCP SDK's threads. It uses the package's own constants and token, and does
nothing else with the token. Build the client on the first request that needs it, never
at startup, because `initialize()` is itself an MCP request and needs a caller. Then share
it: each call captures its own caller.

For calls through Spring's `RestClient`, such as a sub-agent, use the package's
`IdentityClientHttpRequestInterceptor`. For a plain `java.net.http.HttpClient`, use
`IdentityHttpClient.newHttpClient()`. Both read the `ThreadLocal`, so they work only on
the request thread.

## Traps

- The token does not reach an executor thread, a Reactor scheduler or
  `ChatClient.stream()`. A call from there goes out with no user and no error. Keep the
  model turn and its tool calls on the request thread, with a blocking `ChatClient.call()`.
- Do not add `diagrid-spring-ai-starter` to an app that needs the user on its tool calls.
  The starter runs tool calls as workflow activities, after the request has ended.
- Without a Diagrid connection the first request with a token answers 503
  `oauth.not_configured`. Start the app with its connection and tunnel, as
  `diagrid-app-tunnels` describes.

Reference: the `agents/spring-ai/enterprise-identity` quickstart in
`diagridio/catalyst-quickstarts`.

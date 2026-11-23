# FSL SSE — inline publishing POC (FSL 0.0.9)

## Design selected for this POC

This package adapts the earlier `fsl-sse-m1` framework draft to the agreed in-process design. **There is no separate SSE microservice in this milestone.** The reusable capability is a module in FSL Framework, and the supplied `Integration.zip` sample application hosts the stream and publishes events in the same Spring application/JVM.

```text
Postman / UI
     |
     | GET /fsl/sse/stream
     v
+-----------------------------------+
| Existing Integration microservice |
|                                   |
| FSL Framework: com-fsl-framework-sse
|   SseConnectionController         |
|   SseConnectionManager            |
|   SseEventPublisher               |
|                                   |
| Integration business services    |
|   inject SseEventPublisher        |
+-----------------------------------+
                  |
                  +---- in-process event dispatch to registered SseEmitter(s)
```

The **only framework HTTP endpoint for SSE is the GET streaming endpoint**. Publishing is a Java API (`SseEventPublisher`) injected into business services. The old internal HTTP publish controller and publisher-authenticator extension point were removed from this version because the selected POC does not need an HTTP publishing hop.

## Important boundary: same application instance

`SseConnectionManager` is an in-memory registry. An `SseEventPublisher` injected in the Integration microservice dispatches only to connections registered in that same application instance.

If a different microservice (for example, Login) injects its own local `SseEventPublisher`, it will have a different manager and **will not reach clients connected to Integration**. For the first POC, keep both the stream and the event-producing code in Integration. Cross-microservice publishing is a later enhancement and will require a remote publisher adapter over the existing FSL/TCS HTTP integration path (or another agreed transport).

## Files in this package

- `framework-overlay/com-fsl-framework-PARENT/com-fsl-framework-sse/`: new reusable framework module.
- `framework-overlay/com-fsl-framework-PARENT/pom-vt.xml`: adds the module to the FSL 0.0.9 parent build.
- `framework-overlay/com-fsl-framework-dependency-PARENT/pom-vt.xml`: adds dependency management for the new artifact.
- `patches/FSL-FRAMEWORK*.yml`: 0.0.9 CI workflow changes for the module build.
- `integration-poc/Integration/`: copy of the provided `Integration.zip`, updated to depend on the framework SSE module and containing a local Postman trigger endpoint.
- `postman/SSE-Inline-POC.postman_collection.json`: GET stream and POST test trigger.

## What was read from the supplied source archives

1. **`Integration.zip`** has the three-module sample layout: `com-fsl-microservice-filteruserbank`, `com-fsl-microservice-filteruserbank-model`, and `com-fsl-outbound-filteruserbank-model`. Its controller extends `MWRestController`; its service extends `FSLEAIGWIntegrationService`; its outbound call uses the existing integration request/response mapping pattern. The POC adds the SSE dependency only to the runnable microservice module; the model modules are unchanged.
2. **FSL Framework 0.0.9 source** uses `com-fsl-framework-PARENT/pom-vt.xml` for the 0.0.9 module aggregator and `com-fsl-framework-dependency-PARENT/pom-vt.xml` for 0.0.9 dependency management. The starter scans `com.fsl.*`, so `com.fsl.fwk.sse` is within its component-scan packages.
3. **TCS 0.0.9 source** shows `MyFilter` wraps the complete servlet response in `ContentCachingResponseWrapper` and copies the body after the filter chain. That is not suitable for an open-ended SSE stream. The module disables the servlet registration of this filter when SSE is enabled and installs a replacement that preserves the allowed-hostname check and normal response copy behaviour, while bypassing response caching only for the SSE path.
4. **FSL user security source** shows the standard `FSLRestController` / `FSLControllerHelper` flow reads FSL headers, resolves a `User`, loads `FSLUserProfile`, and checks device/profile/session conditions. A custom streaming `@GetMapping` does not automatically execute that MWRestController security flow. Therefore the included header authenticator is explicitly a local POC adapter, not a production substitute for the normal FSL security process.

## Framework module integration

Copy the complete directory:

`framework-overlay/com-fsl-framework-PARENT/com-fsl-framework-sse`

to the same location in the FSL Framework repository.

Replace the FSL repository's `com-fsl-framework-PARENT/pom-vt.xml` and `com-fsl-framework-dependency-PARENT/pom-vt.xml` with the corresponding patched files in `framework-overlay`. Apply the workflow changes from `patches/FSL-FRAMEWORK-SIT.yml` and/or `patches/FSL-FRAMEWORK.yml` to the matching FSL repo workflows. Only the **0.0.9 / `pom-vt.xml`** track is modified. The 0.0.7 and 0.0.8 tracks remain untouched.

Build it in the internal FSL workspace, after the required 0.0.9 FSL dependencies are installed:

```bash
mvn clean install --file ./com-fsl-framework-PARENT/com-fsl-framework-sse/pom-vt.xml
```

The module has an FSL 0.0.9 parent and depends on `com-fsl-framework-baseservice`, Spring MVC, Spring Boot autoconfiguration, and the Servlet API.

## Integrate the sample microservice

The supplied Integration sample's application POM now adds:

```xml
<dependency>
    <groupId>com-fsl-framework-PARENT</groupId>
    <artifactId>com-fsl-framework-sse</artifactId>
</dependency>
```

No dependency was added to either model module.

The sample development properties enable the POC:

```properties
fsl.sse.enabled=true
fsl.sse.stream-path=/fsl/sse/stream
fsl.sse.max-connections=100
fsl.sse.max-connections-per-user=5
fsl.sse.heartbeat-interval=15s
fsl.sse.connection-timeout=0s
FSL.SSE.POC.ENABLED=true
FSL.SSE.POC.SECRET=${FSL_SSE_POC_SECRET}
```

Set the environment variable before starting the Integration microservice:

```bash
export FSL_SSE_POC_SECRET='a-local-only-random-value'
```

The POC-only adapter uses these existing FSL/TCS header names for the simulated identity: `userid`, `userid_type`, `deviceid`, and `channel`. It also requires `X-SSE-POC-KEY`. The test `sessionId` is derived from `deviceid:channel`; that is a demo key, not a verified bank login-session ID.

### Postman sequence

1. Import `postman/SSE-Inline-POC.postman_collection.json`.
2. Set collection variables `baseUrl` (the Integration service base URL) and `pocKey` (same value as `FSL_SSE_POC_SECRET`).
3. Run **1 - Open SSE stream** first. Set `userId`, `deviceId`, and `channel` to sample values (defaults in the collection are `user-101`, `device-101`, and `IMOBILE`). Keep this request open. You should receive a `connected` event with a `connectionId` and then periodic heartbeat comments.
4. While the GET stream remains open, run **2 - Publish demo event**. The endpoint is `POST /sse-poc/events`; its controller calls `SsePocEventService`, which injects `SseEventPublisher` and publishes to that same `userId`.
5. Verify that `DEMO_NOTIFICATION` appears in the open stream. The POST response contains `matchingConnections`, `writeSuccessCount`, and `writeFailureCount`.

The `POST /sse-poc/events` route is a **local POC trigger only**, guarded by the POC shared key and enabled by `FSL.SSE.POC.ENABLED=true`. Remove/disable it before any shared or production deployment. Actual business services should inject `SseEventPublisher` directly; they should not call this POC endpoint.

## Using the API from business services

```java
@Service
public class SomeBusinessService {

    private final SseEventPublisher sseEventPublisher;

    public SomeBusinessService(SseEventPublisher sseEventPublisher) {
        this.sseEventPublisher = sseEventPublisher;
    }

    public void completeBusinessAction(String userId) {
        // Existing business logic remains unchanged.

        SsePublishRequest event = new SsePublishRequest(
                null,
                "BUSINESS_ACTION_COMPLETED",
                "SomeBusinessService",
                null,
                Instant.now(),
                SseTarget.user(userId),
                Map.of("message", "Action completed"));

        sseEventPublisher.publish(event);
    }
}
```

Use `SseTarget.user(userId)`, `SseTarget.session(userId, sessionId)`, or `SseTarget.connection(connectionId)`. The session target uses the exact `(userId, sessionId)` pair to avoid collisions between users with reused session identifiers.

Publish only after the relevant business outcome is known. For transactional work, do not emit a success event before the transaction commits. Delivery is synchronous in this milestone, so load-test fan-out and slow-client behaviour before using it on latency-sensitive paths.

## Security and deployment notes

- `SseClientAuthenticator` is mandatory when `fsl.sse.enabled=true`. If the host application does not provide one, startup fails rather than exposing the stream without authentication.
- The sample authenticator verifies a shared POC key and reads identity headers. This is only for local Postman testing. It does not load/validate `FSLUserProfile`, device block state, concurrent-session termination, hash/replay protections, or the complete standard FSL security checks.
- Before production, provide an adapter that authenticates the SSE GET through the approved gateway/session context, derives a stable user and session ID, and applies the required FSL user/device/session policy. Do not trust client-controlled `userid` or `deviceid` headers unless the gateway strips incoming copies and injects verified values.
- Native browser `EventSource` does not support arbitrary Authorization headers. Align with the real UI authentication route (typically the trusted same-site gateway/session mechanism) rather than exposing tokens in query strings.
- One JVM / one replica only for this POC. The connection map is in memory; do not assume a second application instance can publish to the first instance's clients.
- Heartbeat is 15 seconds by default. Ensure gateway, ingress and load-balancer idle timeouts allow the stream to remain open; disable response buffering on the SSE path.
- `MyFilter` replacement is a high-priority integration test. Verify that non-SSE requests still preserve the required hostname restriction and content-length behaviour in the real application and filter chain.
- No durable event persistence/replay, browser acknowledgement, cross-instance routing, or automatic reconnect replay is implemented.

## Validation status

I inspected the provided FSL framework, TCS 0.0.9 source, and Integration sample and performed static checks on the generated source and POMs. **I could not run Maven compilation or the Spring application here**: Maven is not installed in this environment, and the required internal FSL/TCS artifact repository is not available. Build the new module and Integration sample in the internal FSL 0.0.9 workspace before treating this as runtime-verified.

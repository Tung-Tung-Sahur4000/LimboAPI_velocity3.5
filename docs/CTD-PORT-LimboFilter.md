# Porting LimboFilter to Velocity-CTD

Guide for making **LimboFilter** build and run against the
[Velocity-CTD](https://github.com/GemstoneGG/Velocity-CTD) fork, matching the
CTD-compatible LimboAPI fork in this repository.

> Run this in a Claude Code session **scoped to your fork of LimboFilter**, not
> the LimboAPI repo. Fork `Elytrium/LimboFilter` into your account first.

---

## 1. Background

LimboFilter is built on LimboAPI. It compiles against Velocity's internal
`velocity-proxy` artifact, so — like LimboAPI — it is sensitive to the fork.
In practice the stock Elytrium LimboFilter jar already *runs* on CTD once a
CTD-compatible LimboAPI is installed (it goes through LimboAPI for the
fork-divergent code paths). This guide produces a build that is **compiled**
against the CTD artifacts so it is guaranteed-correct and version-matched.

**Good news:** LimboFilter `master` is already on `velocityVersion=3.5.0-SNAPSHOT`
and `limboapiVersion=1.1.27`, so this is almost entirely a build-system switch.

## 2. What "CTD support" means here

Same recipe applied to LimboAPI in this repo:

1. Pull Velocity from the CTD Maven repo instead of PaperMC.
2. Use the `com.velocityctd` artifact group instead of `com.velocitypowered`.
3. Compile, then fix only the (few, if any) source spots that hit CTD-divergent
   internal APIs — see the reference table in §5.

## 3. Build changes

### 3a. `build.gradle` — add the CTD Maven repository

In the `repositories { ... }` block, add the CTD repo **before** the
papermc-repo entry:

```groovy
    maven {
        setName("velocityctd-repo")
        setUrl("https://repo.velocityctd.com/snapshots/")
    }
```

Keep `elytrium-repo` (needed for `velocity-proxy` and `net.elytrium.limboapi:api`)
and the rest.

### 3b. `build.gradle` — switch the Velocity dependency group

```diff
-    compileOnly("com.velocitypowered:velocity-api:$velocityVersion")
-    annotationProcessor("com.velocitypowered:velocity-api:$velocityVersion")
-    compileOnly("com.velocitypowered:velocity-proxy:$velocityVersion")
+    compileOnly("com.velocityctd:velocity-api:$velocityVersion")
+    annotationProcessor("com.velocityctd:velocity-api:$velocityVersion")
+    compileOnly("com.velocityctd:velocity-proxy:$velocityVersion")
```

### 3c. `gradle.properties` — confirm versions

Already correct on `master`, but verify:

```properties
limboapiVersion=1.1.27
velocityVersion=3.5.0-SNAPSHOT
```

`velocityVersion` here is the **CTD** `3.5.0-SNAPSHOT` published at
`repo.velocityctd.com` (group `com.velocityctd`). Confirm it resolves:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  https://repo.velocityctd.com/snapshots/com/velocityctd/velocity-proxy/3.5.0-SNAPSHOT/maven-metadata.xml
# expect: 200
```

## 4. The LimboAPI dependency

LimboFilter compiles against `net.elytrium.limboapi:api:$limboapiVersion`. The
**api** module's public surface only uses `velocity-api` (stable across the
fork), so you do **not** need a special CTD build of the api to compile.

- **Compile time:** if `net.elytrium.limboapi:api:1.1.27-SNAPSHOT` is not on
  `maven.elytrium.net`, build the api locally from this LimboAPI fork and
  publish it to your local Maven cache:
  ```bash
  # in the LimboAPI fork
  ./gradlew :api:publishToMavenLocal
  ```
  then add `mavenLocal()` to LimboFilter's `repositories { }` (first entry).
- **Runtime:** drop the **CTD LimboAPI plugin jar** built from this repo
  (`limboapi-1.1.27-SNAPSHOT.jar`) into `plugins/` next to LimboFilter.

## 5. Source-adaptation reference (apply only if compile fails)

LimboFilter mostly delegates to LimboAPI, so it likely needs **no** source
changes. If `./gradlew build` reports errors, these are the CTD-divergent APIs
and the exact fixes used in the LimboAPI port (commit history of this repo):

| Symptom (compile error / `NoSuchMethodError`) | CTD-correct fix |
|---|---|
| `LoginEvent(player)` constructor not found | `new LoginEvent(player, null)` |
| `VelocityServer#registerConnection` returns `CompletableFuture<Boolean>` not `boolean` | consume it with `.whenCompleteAsync((registered, err) -> { ... }, eventLoop)` |
| `VelocityServer#canRegisterConnection` missing | remove the call (CTD has no pre-check; treat as `true`) |
| `com.velocitypowered.proxy.config.PlayerInfoForwarding` missing | import `com.velocitypowered.api.proxy.server.PlayerInfoForwarding` |
| Tab list ctor wants `VelocityServer` not `ProxyServer` | change field/param type to `com.velocitypowered.proxy.VelocityServer` |
| `ConnectedPlayer#markLoginEventFired` / `getPlayerRegistry().finalizeLogin` needed | call them where LimboAPI does (login finalize path) |
| `VelocityConfiguration#getProxyBrandCustom()` etc. missing on your CTD build | do **not** call CTD-only brand methods; send a plain `MC|Brand` plugin message |

> Rule of thumb: never hard-depend on a method only present in *newer* CTD
> builds. If a CTD-only API may be absent on the build operators run, guard it
> or fall back to the vanilla behaviour (this is exactly why LimboAPI's brand
> code was simplified).

## 6. Build & verify

```bash
./gradlew build
```

A green build produces the shaded jar under `plugin`/`build/libs/` (or the
project's `build/libs/`). The repo's CI (if present, like LimboAPI's
`build.yml`) will build on push and can publish a prerelease jar.

Runtime smoke test on a Velocity-CTD proxy:

1. `plugins/`: CTD `limboapi-1.1.27-SNAPSHOT.jar` + your new LimboFilter jar.
2. Start the proxy. Expect LimboFilter to load and
   `Captcha generated in N ms` with **no** `NoSuchMethodError` /
   `ReflectionException` during `ProxyInitializeEvent`.

## 7. Checklist

- [ ] Forked `Elytrium/LimboFilter`, session scoped to the fork
- [ ] Added `velocityctd-repo` to `repositories`
- [ ] Switched `com.velocitypowered` → `com.velocityctd` (api + proxy)
- [ ] `gradle.properties`: `velocityVersion=3.5.0-SNAPSHOT`, `limboapiVersion=1.1.27`
- [ ] LimboAPI api available at compile (elytrium-repo or `mavenLocal()`)
- [ ] `./gradlew build` green (apply §5 fixes only if needed)
- [ ] Runtime test on CTD passes (loads, captcha generates, no errors)

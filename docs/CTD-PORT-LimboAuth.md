# Porting LimboAuth to Velocity-CTD

Guide for making **LimboAuth** build and run against the
[Velocity-CTD](https://github.com/GemstoneGG/Velocity-CTD) fork, matching the
CTD-compatible LimboAPI fork in this repository.

> Run this in a Claude Code session **scoped to your fork of LimboAuth**, not
> the LimboAPI repo. Fork `Elytrium/LimboAuth` into your account first.

---

## 1. Background

LimboAuth is built on LimboAPI. It compiles against Velocity's internal
`velocity-proxy` artifact, so — like LimboAPI — it is sensitive to the fork.
In practice the stock Elytrium LimboAuth jar already *runs* on CTD once a
CTD-compatible LimboAPI is installed (it goes through LimboAPI for the
fork-divergent code paths; your proxy log showed LimboAuth 1.1.14 loading fine,
the only crash was LimboAPI's brand code). This guide produces a build that is
**compiled** against the CTD artifacts so it is guaranteed-correct and
version-matched.

> ⚠️ Unlike LimboFilter, LimboAuth `master` is **behind**: it pins
> `velocityVersion=3.4.0-SNAPSHOT` and `limboapiVersion=1.1.26`. Part of this
> task is bumping it to the 3.5 / 1.1.27 line so it matches the CTD LimboAPI.

## 2. What "CTD support" means here

Same recipe applied to LimboAPI in this repo:

1. Pull Velocity from the CTD Maven repo instead of PaperMC.
2. Use the `com.velocityctd` artifact group instead of `com.velocitypowered`.
3. Bump to the `3.5.0-SNAPSHOT` / LimboAPI `1.1.27` line.
4. Compile, then fix only the source spots that hit CTD-divergent internal APIs
   — see the reference table in §5.

## 3. Build changes

### 3a. `build.gradle` — add the CTD Maven repository

In the `repositories { ... }` block, add the CTD repo **before** papermc-repo:

```groovy
    maven {
        setName("velocityctd-repo")
        setUrl("https://repo.velocityctd.com/snapshots/")
    }
```

Keep `elytrium-repo`, `papermc-repo`, and `opencollab-snapshot` (Floodgate).
Resulting block:

```groovy
repositories {
    mavenCentral()
    maven {
        setName("velocityctd-repo")
        setUrl("https://repo.velocityctd.com/snapshots/")
    }
    maven {
        setName("elytrium-repo")
        setUrl("https://maven.elytrium.net/repo/")
    }
    maven {
        setName("papermc-repo")
        setUrl("https://repo.papermc.io/repository/maven-public/")
    }
    maven {
        setName("opencollab-snapshot")
        setUrl("https://repo.opencollab.dev/maven-snapshots")
    }
}
```

### 3b. `build.gradle` — switch the Velocity dependency group

```diff
-    compileOnly("com.velocitypowered:velocity-api:$velocityVersion")
-    annotationProcessor("com.velocitypowered:velocity-api:$velocityVersion")
-    compileOnly("com.velocitypowered:velocity-proxy:$velocityVersion")
+    compileOnly("com.velocityctd:velocity-api:$velocityVersion")
+    annotationProcessor("com.velocityctd:velocity-api:$velocityVersion")
+    compileOnly("com.velocityctd:velocity-proxy:$velocityVersion")
```

### 3c. `gradle.properties` — bump versions

```diff
-limboapiVersion=1.1.26
-velocityVersion=3.4.0-SNAPSHOT
+limboapiVersion=1.1.27
+velocityVersion=3.5.0-SNAPSHOT
```

`velocityVersion` is the **CTD** `3.5.0-SNAPSHOT` (group `com.velocityctd`).
Confirm it resolves:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  https://repo.velocityctd.com/snapshots/com/velocityctd/velocity-proxy/3.5.0-SNAPSHOT/maven-metadata.xml
# expect: 200
```

> Bumping 3.4 → 3.5 may surface API drift unrelated to CTD (Velocity moved a
> few internals between 3.4 and 3.5). Treat any such compile errors the same
> way as §5 — adjust to the 3.5 signature. The LimboAPI fork in this repo is a
> working reference for the 3.5 internal API shapes.

## 4. The LimboAPI dependency

LimboAuth compiles against `net.elytrium.limboapi:api:$limboapiVersion`. The
**api** module's public surface only uses `velocity-api` (stable across the
fork), so you do **not** need a special CTD build of the api to compile.

- **Compile time:** if `net.elytrium.limboapi:api:1.1.27-SNAPSHOT` is not on
  `maven.elytrium.net`, build the api locally from this LimboAPI fork and
  publish it to your local Maven cache:
  ```bash
  # in the LimboAPI fork
  ./gradlew :api:publishToMavenLocal
  ```
  then add `mavenLocal()` to LimboAuth's `repositories { }` (first entry).
- **Runtime:** drop the **CTD LimboAPI plugin jar** built from this repo
  (`limboapi-1.1.27-SNAPSHOT.jar`) into `plugins/` next to LimboAuth.

## 5. Source-adaptation reference (apply only if compile fails)

LimboAuth is mostly high-level (BCrypt/TOTP/ORMLite over LimboAPI), so it likely
needs **no** CTD-specific source changes — but the 3.4→3.5 bump may. If
`./gradlew build` reports errors, these are the CTD-/3.5-divergent APIs and the
exact fixes used in the LimboAPI port (commit history of this repo):

| Symptom (compile error / `NoSuchMethodError`) | CTD/3.5-correct fix |
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
> code was simplified after it crashed on CTD build b604).

## 6. Build & verify

```bash
./gradlew build
```

A green build produces the shaded jar under `build/libs/`. The repo's CI (if
present, like LimboAPI's `build.yml`) will build on push and can publish a
prerelease jar.

Runtime smoke test on a Velocity-CTD proxy:

1. `plugins/`: CTD `limboapi-1.1.27-SNAPSHOT.jar` + your new LimboAuth jar.
2. Start the proxy. Expect LimboAuth to load and complete
   `ProxyInitializeEvent` with **no** `NoSuchMethodError` / `ReflectionException`.
3. Join the server and confirm the auth (register/login) limbo flow works.

## 7. Checklist

- [ ] Forked `Elytrium/LimboAuth`, session scoped to the fork
- [ ] Added `velocityctd-repo` to `repositories`
- [ ] Switched `com.velocitypowered` → `com.velocityctd` (api + proxy)
- [ ] `gradle.properties`: `velocityVersion=3.5.0-SNAPSHOT`, `limboapiVersion=1.1.27`
- [ ] LimboAPI api available at compile (elytrium-repo or `mavenLocal()`)
- [ ] `./gradlew build` green (apply §5 fixes for any CTD/3.5 drift)
- [ ] Runtime test on CTD passes (loads, register/login flow works, no errors)

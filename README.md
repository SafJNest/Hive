# Hive

Hive is a JDK-only, in-memory job runtime. It can be used without Discord,
Spring, Riot, Mongo or Redis.

## Build

Requires JDK 21 or newer and Maven.

```sh
mvn clean verify
```

The artifact is built in `target/hive-1.0-alpha-1.jar`. Each consumer creates a
`Hive` instance with its own configuration, registers its local schedulers,
submits work through its `Router` and closes the runtime. Register schedulers
against that runtime's `Registry`. Loading Hive classes does not start
application schedulers.

```java
import java.util.concurrent.TimeUnit;

import com.hive.Hive;
import com.hive.HiveConfig;
import com.hive.Registry;
import com.hive.job.JobPriority;
import com.hive.scheduler.AbstractScheduler;

public class HiveExample {
    public static void main(String[] args) throws Exception {
        try (Hive hive = new Hive(HiveConfig.defaults("example"))) {
            hive.register(new DemoScheduler(hive.registry()));
            Integer value = hive.router().submit(DemoScheduler.class, "reports",
                "report:42", "Example report", JobPriority.NORMAL, Integer.class,
                job -> 42).get(5, TimeUnit.SECONDS);
            System.out.println(value);
        }
    }

    private static final class DemoScheduler extends AbstractScheduler<String> {
        private DemoScheduler(Registry registry) {
            super("reports", "Reports stopped", registry);
        }

        @Override
        protected String routeName(String route) { return route; }

        @Override
        protected String workerThreadName(String route) { return "reports-" + route + "-"; }

        @Override
        protected String routeForJob(Object route) {
            if (route instanceof String name && !name.isBlank()) return name;
            throw new IllegalArgumentException("A report route is required");
        }
    }
}
```

Use `ResultType.list(Element.class)` for typed list results. Deduplication checks
result contracts and uses scheduler, logical route and key, independently of
physical placement. Independent runtimes share no dedup entries or parent
context.

`HiveConfig` controls runtime identity, Clock, queued/status snapshot limits,
shutdown timeout and observation. Scheduler constructors/hooks control queued or
direct execution, route validation, physical placement and names. Direct mode
runs virtual threads without inventing a physical queue. Snapshot records live
in `com.hive.status`; applications interpret their own operational run keys.

Register contextual continuations with `hive.thenApply`, `hive.thenCompose` or
`hive.whenComplete` while running the parent, so callbacks retain its lifecycle
and restore context. Explicit `retain/resume` is available when integrating a
different callback API. Avoid waiting synchronously for a job on the same serial
worker occupied by its parent; move orchestration to direct execution or
compose it asynchronously.

`hive.termination(future)` returns a read-only stage for real logical
termination, separate from the result view. Use it for cleanup that must wait
for actual completion even when the caller cancels its view.

Cancelling a result future does not stop the body. Explicit job cancellation and
runtime shutdown manage logical lifecycle and interruption separately. Bodies
that ignore interruption may outlive the configured shutdown wait. Jobs and
queues are volatile: no persistent backlog, remote execution or distributed
deduplication.

## Publish and consume

The Maven coordinates are `com.hive:hive:1.0-alpha-1`. Publishing uses GitHub
Packages for this repository. Create the tag `v1.0-alpha-1` to run the publish
workflow, or publish locally with `mvn deploy` after configuring GitHub
credentials in Maven settings.

Add this repository and dependency to a consumer such as Beebot:

```xml
<repositories>
  <repository>
    <id>github</id>
    <url>https://maven.pkg.github.com/SafJNest/Hive</url>
  </repository>
</repositories>

<dependency>
  <groupId>com.hive</groupId>
  <artifactId>hive</artifactId>
  <version>1.0-alpha-1</version>
</dependency>
```

For local Maven use, configure `~/.m2/settings.xml` with a GitHub personal
access token (classic) that has package read access. Use package write access
when publishing manually. The GitHub Actions workflow uses `GITHUB_TOKEN` with
package write permission:

```xml
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>${env.GITHUB_ACTOR}</username>
      <password>${env.GITHUB_TOKEN}</password>
    </server>
  </servers>
</settings>
```

The publish workflow uses the repository's `GITHUB_TOKEN` and requires package
write permission. No Beebot classes or third-party runtime dependencies are
included in the Hive artifact.

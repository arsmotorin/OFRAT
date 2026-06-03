[![JitPack](https://jitpack.io/v/arsmotorin/ofrat.svg)](https://jitpack.io/#arsmotorin/ofrat)
[![License](https://img.shields.io/github/license/arsmotorin/OFRAT)](https://github.com/arsmotorin/OFRAT/blob/master/LICENSE)
[![Discord](http://img.shields.io/discord/1456716690879676501?label=Discord&style=flat&logo=discord)](https://discord.gg/uwMMZ2KWk6)

<div align="center"> <img src="https://i.imgur.com/xn18HzJ.png" alt="OFRAT"> </div>
One file rules all targets!

> [!WARNING]
> This is an experimental project and API can be changed in the future. You can use it, but it's not recommended.

## The situation

You're writing a Minecraft server plugin — and it has to run on both Fabric and Paper.
The logic is identical. The same storage format, same commands, same packet handling, etc.
But Fabric says `ServerLevel`, Paper says `World`. Fabric says `BlockPos`, Paper says `Location`.

So you end up with two codebases that do the same thing. **OFRAT breaks that loop.**

## How it works

Annotate declarations by platform. OFRAT's Kotlin compiler plugin removes the ones that don't belong — before a single byte is written to disk.

```kotlin
object Scheduler {
    @FabricOnly fun scheduleAsync(task: Runnable) {
        Thread(task, "async-worker").also { it.isDaemon = true }.start()
    }

    @PaperOnly fun scheduleAsync(task: Runnable) {
        Bukkit.getScheduler().runTaskAsync(plugin, task)
    }
}
```

Compile for Paper? The `@FabricOnly` block is gone.

```sh
javap -p build/libs/my-paper.jar 'com.example.Scheduler'
# scheduleAsync() <- Paper version only. The Fabric one never made it to bytecode.
```

And there is no runtime overhead too, and it's the same file for both platforms.

## Quick start

### 1. Add JitPack

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        maven("https://jitpack.io")
    }
}
```

### 2. Add dependencies

```kotlin
// build.gradle.kts
dependencies {
    // Annotation classes
    compileOnly("com.github.arsmotorin.ofrat:ofrat-annotations:1.0.0")

    // Compiler plugin
    "kotlinCompilerPluginClasspath"("com.github.arsmotorin.ofrat:ofrat-plugin:1.0.0")

    // Both platform APIs as compileOnly, so types inside annotated blocks resolve
    compileOnly("net.fabricmc:fabric-api:...")
    compileOnly("io.papermc.paper:paper-api:...")
}
```

### 3. Set the target platform

```kotlin
// build.gradle.kts
tasks.withType<KotlinCompile>().configureEach {
    compilerOptions.freeCompilerArgs.addAll(
        "-P", "plugin:io.github.arsmotorin.ofrat:platform=paper"
    )
}
```

### 4. Write

```kotlin
import io.github.arsmotorin.ofrat.FabricOnly
import io.github.arsmotorin.ofrat.PaperOnly

object WorldHelper {

    // No annotation? Compiled into every JAR
    fun packCoords(x: Int, y: Int, z: Int): Long =
        (x.toLong() shl 38) or (z.toLong() shl 12) or y.toLong()

    // Fabric only? Stripped from the Paper build
    @FabricOnly fun getLevel(server: MinecraftServer, key: String): ServerLevel? {
        val rl = Identifier.parse(key)
        return server.getLevel(ResourceKey.create(Registries.DIMENSION, rl))
    }

    // Paper only
    @PaperOnly fun getWorld(name: String): World? = Bukkit.getWorld(name)
}
```

## Annotations

| Annotation      | Kept when building for | Stripped when building for      |
|-----------------|------------------------|---------------------------------|
| `@FabricOnly`   | `fabric`               | Paper, Neoforge, anything else  |
| `@PaperOnly`    | `paper`                | Fabric, Neoforge, anything else |
| `@NeoForgeOnly` | `neoforge`             | Fabric, Paper, anything else    |

## Custom platform annotations

OFRAT isn't limited to the three built-in annotations. Define your own in one line:

```kotlin
import io.github.arsmotorin.ofrat.PlatformOnly

@PlatformOnly("spigot")
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class SpigotOnly
```

It picks it up automatically. You don't need to add it to the compiler plugin classpath.

## One thing to know

Two declarations with the **exact same name and parameter types** in the same class are a
Kotlin compile error — the type-checker sees both before the IR phase where OFRAT runs.

```kotlin
// Kotlin frontend duplicate error, OFRAT never even gets to run
@FabricOnly val configDir: String = "config/mod"
@PaperOnly  val configDir: String = "plugins/Mod"
```

This is intentional and **will not be fixed** — it's actually a feature.

If you need the same name on both platforms, that's a signal the concept belongs behind
an interface. Define it once in shared code, implement it per platform:

```kotlin
// Shared. No annotation, compiled into every JAR
interface PlatformPaths {
    val configDir: String
}

// Compiled into the Fabric JAR only:
@FabricOnly
object FabricPaths : PlatformPaths {
    override val configDir = "config/mod"
}

// Compiled into the Paper JAR only:
@PaperOnly
object PaperPaths : PlatformPaths {
    override val configDir = "plugins/Mod"
}
```

Shared code talks to `PlatformPaths`. Each JAR ships exactly one implementation.

## What's the difference comparing to Architectury?

Both OFRAT and Architectury solve the same problem, but they solve it in different ways.

|                          | Architectury            | OFRAT                       |
|--------------------------|-------------------------|-----------------------------|
| Model                    | Abstraction API         | Annotation-driven stripping |
| Code location            | Separate source modules | Single file                 |
| Native platform APIs     | Wrapped                 | Direct                      |
| Paper support            | No                      | Yes                         |
| Runtime overhead         | Delegation layer        | Zero                        |
| Custom platforms         | No                      | Yes                         |
| Compile-time elimination | No                      | Yes                         |


## Donate us

OFRAT is developed with passion and dedication by me and my friends. If you enjoy our mod and want to support
further development, consider making a [donation](https://www.paypal.com/paypalme/frogdream).

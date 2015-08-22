## Injekt/config-typesafe

Injekt supports injection from [Typesafe Config](https://github.com/typesafehub/config) configuration via bound objects into the Injekt registry, then you may inject your configuration objects as you would any singleton.

The configuration system allows you to define within modules: the configuration objects and a section of the configuration 
tree that is to be used.  Then at the main module of your application you import the module's configurables and injectables
delegating the registation to the submodules.

A few other notes:  Jackson Data Binding is used to bind configuration into objects.  (later we will replace with a lighterweight binder).  And we use the [Klutter/config-typesafe](https://github.com/klutter/klutter/tree/master/config-typesafe) wrapper to control and load configuration in the samples below, although you may still use Typesafe Config's load and parse methods directly.  All of the extension methods from [Klutter/config-typesafe](https://github.com/klutter/klutter/tree/master/config-typesafe) are available for use if you use the configuration directly.

This module is `uy.kohesive.injekt:injekt-config-typesafe` for the lastest JDK (currently JDK 8).  For earlier versions of Java, use:

* JDK 6 - `uy.kohesive.injekt:injekt-config-typesafe-jdk6`  
* JDK 7 - `uy.kohesive.injekt:injekt-config-typesafe-jdk7`  
* JDK 8 - `uy.kohesive.injekt:injekt-config-typesafe-jdk8` (or as above)  

Each of these depends on the same JDK level for [Klutter/config-typesafe](https://github.com/klutter/klutter/tree/master/config-typesafe) 

### This modules is best shown by a quick example:

Using a configuration file, `myConfig.conf`
```json
{
    "http": {
        "httpPort": 8080,
        "workerThreads": 16
    },
    "data": {
        "bucket": "com.test.bucket",
        "region": "us-east"
    },
    "other": {
        "name": "frisbee"
    }
}
```

We then have a sub module that wants the `http` configuration bound into this data class:

```kotlin
data class HttpConfig(val httpPort: Int, val workerThreads: Int)
```

The module creates an importable module that registers a configuration class, and any injectables, and code that uses the configuration object later via injection:

```kotlin
public object ServerModuleInjectables : KonfigModule, InjektModule {
    override fun KonfigRegistrar.registerConfigurables() {
        // register our HttpConfig object to be bound from the root of our section of the configuration file
        bindClassAtConfigRoot<HttpConfig>()
    }

    override fun InjektRegistrar.registerInjectables() {
        // register injections as normal, and those can use configuration objects since they are available for injection
        addFactory { MyHttpServer() }
    }
}

// Our server will receive the configuration via injection
class MyHttpServer(cfg: HttpConfig = Injekt.get()) { ... }
```

Now the main controlling class, the Application creates something similar to an `InjektMain`, but instead is a `ConfigAndInjektMain` such as:

```kotlin
class MyApp {
    companion object : KonfigAndInjektMain() {
        // my app starts here with a static main()
        platformStatic public fun main(args: Array<String>) {
            MyApp().run()
        }

        // override and load the configuration
        override fun configFactory(): Config {
            return loadConfig(SystemPropertiesConfig(), ApplicationConfig(), ReferenceConfig(), EnvironmentVariablesConfig())
        }
        
        // register any configuration bindings, and import our server module
        override fun KonfigRegistrar.registerConfigurables() {
            bindClassAtConfigPath<S3Config>("data")
            bindClassAtConfigPath<OtherConfig>("other")
            
            importModule("http", ServerModuleInjectables) // use the http:{} section of the configuration
        }

        
        // register injections as normal importing our server module
        override fun InjektRegistrar.registerInjectables() {
            // add my singletons, factories, keyed factories, per-thread factories, ...
            importModule(JacksonWithKotlinInjektables)
            importModule(ServerModuleInjectables)  // our server can be injected, and it itself has the config injected
        }
    }
    
    data class DataConfig(val bucket: String, val region: String)
    data class OtherConfig(val name: String)
    
    // rest of class ...
}
```

Using this system, you have a modularized way to define configuration and services (injectables) that can be managed by the Injekt registry.  Each module defines what it needs, and what it can provide.  Then the main control point of the app decides where in its overall configuration each module resides, and which modules are included.

### Roadmap (random order)

* Allow binding an object that has a 1 parameter constructor, of type Config that does not use binding, but rather loads the config directly using Typesafe Config or Klutter/config-typesafe methods.  See issue #9
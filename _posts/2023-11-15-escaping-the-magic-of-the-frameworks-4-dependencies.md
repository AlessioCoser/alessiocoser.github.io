---
layout: post
title:  "Escaping the magics of the frameworks: 4. Dependencies"
author: alessio
categories: [ frameworks, springboot, jvm, kotlin, annotations, dependencies, ioc, 'depdendency injection' ]
image: assets/images/escaping-frameworks-magics-4.jpg
---
<script src="https://platform.linkedin.com/in.js" type="text/javascript">lang: en_US</script>

The way Spring handles the dependencies is one of its mightiest magics. You can define the dependencies in different ways, even through an XML file. You have annotations and properties files to configure the way SpringBoot behaves. It's so generic that it **adds a lot of complexity** to manage every scenario an application could need, and it does it at runtime, because it has to parse some files and to process annotations through reflection.

... but, most probably, you will always need only one of those scenarios at a time.

<center>Are you sure you want all this complexity just to avoid writing some lines of code to instantiate your own classes?</center>

This is the fourth out of five articles about how and why to escape the framework's magic.

In this series of articles I'm **replacing, magic by magic, some framework's features** until I have no more sorceries to wield. I'm doing it in **baby steps**, explaining the reasons why, and **keeping my application working** all the time.

You can read about **frameworks**, **what's wrong with magics**, **configuration as code** and **explicit routing** in the previous articles. I also addressed how to decouple the tests from the framework.

> [Escaping the magics of the frameworks: 1. Configuration](/escaping-the-magic-of-the-frameworks-1-configuration/)
>
> [Escaping the magics of the frameworks: 2. HTTP Routing](/escaping-the-magic-of-the-frameworks-2-http-routing/)
>
> [Escaping the magics of the frameworks: 3. Tests](/escaping-the-magic-of-the-frameworks-3-tests/)

From now on I will continue where I left off with the code examples tackling the **refactoring of the Spring dependency injection mechanism**.

Please <script type="IN/Share" alt="share" data-url="{{site.url}}{{page.url}}"></script> **this article on LinkedIn and tag me** if you want to start a public conversation on this topic or [**contact me**](/contact) if you prefer to talk about it privately.

## Spring IoC container

IoC stands for "Inversion of Control". This is a principle that aims to transfer the control of an object or an application to a container or a framework that will execute it.

Spring applies this principle with its IoC container. It manages the dependencies as beans. A bean is an object that is instantiated, assembled, and managed by the Spring IoC container. It's the base concept of a Spring application. The Spring IoC container knows what instance to inject at runtime searching for every single `@Component` annotated class (along with `@Service`, `@Controller`, and other annotations) and processing them as beans.

A bean can also be registered through a `@Configuration` annotated class. That class can have one or more `@Bean` methods and it's processed by the Spring IoC container to generate the bean definitions at runtime.

When I need to inject a dependency (a bean) into one component I can use the `@Autowired` annotation. It will tell Spring to search for a Spring bean that implements the required interface and place it automatically there.

This annotation can be placed over different statements of a class: constructor, field and setter method. Actually, I can omit the `@Autowired` annotation when I use the constructor injection.

## What's wrong with the Spring IoC?

What I'm doing with these annotations is **polluting the codebase** with the framework implementation details. This will probably result in a codebase that is more difficult to test in isolation.

Other than that, **I am moving the dependencies evaluation from compile-time to runtime**. This reduces the feedback loop frequency I have during development. I prefer to keep this type of feedback at compile time, because this way the IDE will tell me immediately, while I'm writing the code, if there is something wrong, and the application will not compile. It's safer.

I'm also hurting myself **making the code harder to understand and navigate**, because there is this magic involved that doesn't let me understand which instance I'm using in a specific component. I can know this only at runtime, especially on bigger applications. Doing a manual dependency injection helps me understand the application’s object graph, how it’s created and wired. Just looking at the code, and navigating it, it’s crystal clear what’s happening and when.

In any case I try to avoid the use of field or setters injections. By using them I'm **coupling the class to the framework injection mechanism** and I'm hiding a number of signals that could push me to improve the design of my code. It also drives me to build a **complex chain of dependencies** that makes writing tests more coupled to the code structure. All these things make refactoring the code a nightmare, so the application quality will degrade as time passes. **This is why I prefer the constructor injection**.

## @Service
I start with the `@Service` annotation from the `CounterService` class, because it's the leaf of the application dependency-graph.

I can remove the annotation by explicitly configuring the service as a `@Bean` in the configuration.

**AppConfig.kt**
```kotlin
@Configuration
class AppConfig : WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {

    @Bean
    fun properties() = Properties()

    @Bean
    fun counterService(props: Properties) = CounterService(props.exampleInitial)

    override fun customize(factory: ConfigurableServletWebServerFactory) {
        factory.setPort(properties().serverPort)
    }
}
```
([See the diff - tag `14-SERVICE`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/1ffb113e5f4...14-SERVICE?diff=unified))

## @Component
I can do almost the same also with the `@Component` annotation from the `GreetingController`.

I just define the controller inside the `AppConfig` and I use it in the router defined inside the `Application` class.

I don't need to annotate the greetingController method as a Bean, because I will not use it through the Spring IoC. I will use the `AppConfig` instance instead. I only need to declare it as a `config` field in the `Application` class.

**AppConfig.kt**
```kotlin
@Configuration
class AppConfig : WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {

    @Bean
    fun properties() = Properties()

    @Bean
    fun counterService(props: Properties) = CounterService(props.exampleInitial)

    fun greetingController() = GreetingController(counterService(properties()))

    override fun customize(factory: ConfigurableServletWebServerFactory) {
        factory.setPort(properties().serverPort)
    }
}
```

**Application.kt**
```kotlin
@Configuration
@SpringBootApplication
class Application: AutoCloseable {
    private val config: AppConfig = AppConfig()
    private var appContext: ConfigurableApplicationContext? = null
    private val app = SpringApplication(Application::class.java)

    @Bean
    fun router(): RouterFunction<ServerResponse> {
        return router {
            config.greetingController().routes(this)
        }
    }

    // [...]
}
```
([See the diff - tag `15-COMPONENT`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/14-SERVICE...15-COMPONENT?diff=unified))


## Port configuration
Putting the `Application` class aside for a moment, I am only missing the `Configuration` annotation of `AppConfig`.

To get rid of that I still need to find a way to define the port configuration in a more explicit way.

I can do that by updating the default properties of the `SpringApplication` instance using a [Scope Function](https://kotlinlang.org/docs/scope-functions.html) (see [.apply](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html)):

**Application.kt**
```kotlin
@Configuration
@SpringBootApplication
class Application: AutoCloseable {
    private val config: AppConfig = AppConfig()
    private var appContext: ConfigurableApplicationContext? = null
    private val app = SpringApplication(Application::class.java).apply {
        setDefaultProperties(mapOf("server.port" to config.properties().serverPort))
    }

    // [...]
}
```

Then, in the `AppConfig` class, I can remove both the override of the `customize` method and the extension of the generic class `WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>`. The result is way simpler:

**AppConfig.kt**
```kotlin
@Configuration
class AppConfig {

    @Bean
    fun properties() = Properties()

    @Bean
    fun counterService(props: Properties) = CounterService(props.exampleInitial)

    fun greetingController() = GreetingController(counterService(properties()))
}
```
([See the diff - tag `16-PORT-CONFIG`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/15-COMPONENT...16-PORT-CONFIG?diff=unified))

## AppConfig @Configuration
Now I'm able to approach the `@Configuration` annotation of `AppConfig`.

First of all I can remove all the `@Configuration` and `@Bean` annotations since I'm already retrieving the dependencies through the manually created instances.

The Spring Beans are, by default, instantiated as singleton. To have the same behavior I can change the methods into variables (it looks even prettier).

Then I extract the `Properties` from the `AppConfig`. It was strange that it was the `AppConfig` to instantiate them. Before, it was mandatory to keep the properties visible to the Spring IoC.

Now I don't use the Spring IoC so it does not need to be declared inside the `AppConfig`. In that way I can also avoid accessing the properties through the `AppConfig` class (this was violating the [law of demeter](https://en.wikipedia.org/wiki/Law_of_Demeter)).

**AppConfig.kt**
```kotlin
class AppConfig(props: Properties) {
    val counterService = CounterService(props.exampleInitial)
    val greetingController = GreetingController(counterService)
}
```

Since `CounterService` is not used anywhere else I could have set that variable as private.

**Application.kt**
```kotlin
@Configuration
@SpringBootApplication
class Application: AutoCloseable {
    private val props: Properties = Properties()
    private val config: AppConfig = AppConfig(props)
    private var appContext: ConfigurableApplicationContext? = null
    private val app = SpringApplication(Application::class.java).apply {
        setDefaultProperties(mapOf("server.port" to props.serverPort))
    }

    // [...]
}
```
([See the diff - tag `17-APPCONFIG`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/16-PORT-CONFIG...17-APPCONFIG?diff=unified))

The way I handle all the dependencies in a single file is just one approach. It's more similar to the way I configure the Beans in Spring. This is why I chose that method.

I could have removed the `AppConfig` class completely and created all the dependencies in the main `Application` class. Or I could have chosen other approaches with way less magic than the Spring way.

I used the `AppConfig` class to demonstrate that is perfectly possible to have the same sort of `bean` definition `@Configuration` with some advantages:
- It's code
- It does not use reflection
- The dependency graph is evaluated at compile time
- I can still have dependencies that depend on others
- I can still decide to do a lazy initialization for some dependencies using the [by lazy](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/lazy.html) kotlin [delegated property](https://kotlinlang.org/docs/delegated-properties.html)

## Router Bean
Let's tackle the `Application` class now and remove its `@Configuration` annotation along with the `@Bean` definition for the router.

Unfortunately it seems that the Spring routing mechanism is tightly coupled to the Spring IoC, so I am forced to define the router as a Bean.

Luckily Spring also provides a Kotlin DSL to define the beans in a less magical way. Instead of using these annotations I can add a custom initializer to the `SpringApplication` and I can define all the beans I need for the application. Yes, it is the router only, because the rest of the dependencies are manually provided.

**Application.kt**
```kotlin
@SpringBootApplication
class Application: AutoCloseable {
    private val props: Properties = Properties()
    private val config: AppConfig = AppConfig(props)
    private var appContext: ConfigurableApplicationContext? = null
    private val app = SpringApplication(Application::class.java).apply {
        setDefaultProperties(mapOf("server.port" to props.serverPort))
        addInitializers(beans {
            bean {
                router {
                    config.greetingController.routes(this)
                }
            }
        })
    }

    // [...]
}
```
([See the diff - tag `18-ROUTER-BEAN`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/17-APPCONFIG...18-ROUTER-BEAN?diff=unified))

In any case the router is now defined in the Application entrypoint, it is defined as code, it is straightforward to understand. So, it's enough for me at the moment.

## Configurable Application
I may need to configure my Application in the tests in order to test the code branches I need.

I just moved the `props` field inside the constructor. It is like to have an `application.properties` file inside the test classpath, but with less magic.

**Application.kt**
```kotlin
@SpringBootApplication
class Application(private val props: Properties = Properties()): AutoCloseable {
    private val config: AppConfig = AppConfig(props)
    // [...]
}
```

Doing so I can, for example, change the application port with a random number in each test. I only need to change the runApp helper method and configure the Application and the client with the same port.

**Application.kt**
```kotlin
class GreetingControllerTests {

    // tests

    private fun runApp(test: (client: Http) -> Unit) {
        val port = Random.nextInt(35000, 65000)
        val client = Http("http://localhost:${port}")
        Application(Properties(serverPort = port))
            .start(emptyArray())
            .use { test(client) }
    }
}
```
([See the diff - tag `19-CONFIGURABLE-APP`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/18-ROUTER-BEAN...19-CONFIGURABLE-APP?diff=unified))

## Clean up

At this point I have two classes: `AppConfig` and `Properties`. Now I don't think the names are clearly expressing their intent. So I choose to do two simple renames.

**AppConfig --> AppDependencies**
... since its duty is to prepare and retrieve (statically) the dependencies of the application.

**Properties --> Config**
... since its duty is to provide the configuration variables to the application.

([See the diff - tag `20-CLEANUP`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/19-CONFIGURABLE-APP...20-CLEANUP?diff=unified))

## Conclusions

I avoided magics in the dependencies instantiation, definition and retrieval. It's only code. It can be verified at compile-time. The IDE helps me immediately while I'm writing the code.

Of course there are more lines of code to write, even more if I create the dependencies straight inside the `Application` class.

**But has this boilerplate code to be avoided?**<br />

I don't think so, on the contrary I think it is a more explicit way to define the dependency graph.

The **object graph** is the way the application builds and instantiates the dependencies starting from the main entrypoint. **It's like the network of our application**. It tells me how the different classes are related. It speeds up the codebase reading and understanding. When I'm lost I can always go back to the application entrypoint and navigate down through the dependency graph.

**Spring Boot is still there**<br />

As I'm sure you noticed, I couldn't get rid of all the Spring beans. I still need the router bean that tells Spring which routes are registered but at least, I managed to put that logic in the main `Application` class. I think it is the best place for the routing, it's quite near to the entrypoint, it eases the controllers instantiation and the code understanding.

I cannot even remove the `@SpringBootApplication` annotation on the `Application` class. **It's still Spring Boot after all**.

Anyway I think I reduced the magic performed by the framework to a minimum and I still managed to have a lighter and more expressive application structure.

**What's next?**<br />

What about switching frameworks? Is it possible to change Spring Boot with a less magical framework? What will need to be changed?

**Stay tuned for the last episode :)**

Please let me know your opinion <br /><script type="IN/Share" alt="share" data-url="{{site.url}}{{page.url}}"></script> this article on LinkedIn and tag me if you find it interesting.
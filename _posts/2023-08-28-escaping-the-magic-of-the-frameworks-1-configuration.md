---
layout: post
title:  "Escaping the magics of the frameworks: 1. Configuration"
author: alessio
categories: [ frameworks, springboot, jvm, kotlin, annotations, configuration ]
image: assets/images/escaping-frameworks-magics.jpg
---
<script src="https://platform.linkedin.com/in.js" type="text/javascript">lang: en_US</script>

**When you adopt a framework you choose to give it control of your application**. It is the framework that decides what part of your app to call based on its logic.

What is sure is that **we are surrounded by frameworks**; even the Java platform could be considered a framework because it is an abstraction on top of the operating system that has control over your application.

> The ability to get everything we need “magically” working is quite tempting and satisfactory, even if at the cost of some flexibility, navigability and understanding.

Whether or not to use a framework is a **controversial topic**, with fairly good arguments on both sides. I am neither saying **"don't use a framework"** nor **"use a framework"**. They are both very valid approaches in different contexts and teams for different reasons. My aim is to try, by means of a practical exercise, to clarify if and how these magics really simplify the development of an application.

**With this series of articles I want to try to replace, magic by magic, some framework's features until I have no more sorceries to wield**. I could have replaced all the magics in one go, but I think I would have missed an opportunity to explain the motivation that made me want to avoid these magics.

In that way we can analyse the different features a bit more, and see what change if we avoid the use of that magic. It's better? It's worse? It's something in the middle?

At the end of this series we will see the code where we started and we will compare it with the less magic implementation considering the pros and cons.

Please <script type="IN/Share" data-url="{{site.url}}{{page.url}}"></script> **this article on LinkedIn and tag me** if you want to start a public conversation on this topic or [**contact me**](/contact) if you prefer to talk about it privately.

## What's wrong with "magics"

I see **magics** as things happening in my application **that cannot be reached by code navigation**. I don't know why, where or in which order they happen: It's magic!

Thus, it becomes **difficult to understand** the flow in the code, sometimes the IDE cannot know if a method is used or not and this **feedback is postponed** at the start of the application: i.e. at runtime.

Typically, it's necessary to study and **know the execution flow of the framework** thoroughly to understand what is really going on. Sometimes you don't need to know it, and that's okay, until you have to do something specific or you have problems on that part.

In many teams I have seen the senior people orally handing down to the juniors the effects of the framework magics. **You can't figure it out by navigating the code**, so every time you have to remember how that magic works. From my point of view **this greatly increases the cognitive load** instead of reducing it.

This is why, before choosing a **framework** or any of its **magical features**, we should think through the pros and cons and consider whether its use gives us a real benefit:
- Does it really help me to do things in a cleaner way and save me a lot of boilerplate code?
- Does a delayed feedback at compile time really help me?
- Do I have to reinvent the wheel if I don't use it or is it enough to use another simpler library?
- Does it really help me reduce cognitive load or does it increase it?

## The configuration

One of the framework's magical features is the **configuration file**. Usually you can find it in your repo as an `xml`, `yml` or a `.properties` file. The framework then, **at runtime**, parses the file and injects the configuration into the application.

I like to follow the [12 factor app](https://12factor.net/config) guidelines about configuration. It promotes a **strict separation of config from code**, because the code doesn't change between environments, but the config does.

Thus, configuration is a set of variables that change from one environment to another (staging, production, testing, and so on). Too many times **other variables** are also placed in the config file along with the configurations, but they do not change from environment to environment, so they are not true configurations.

These other variables **should be coded and kept as close as possible to where they are used**. This approach helps us greatly improve the discoverability and readability of our code.

> The more a variable is explicit and near to the usage, the more easy to read and understand becomes my code.

So, what I prefer is to have a **small set of configurations** (only those that change) that are given as **environment variables** independently of the language, the framework and the OS.

Then I keep these variables in one place in the code. This takes the **logic of reading and parsing variables away from the application logic**, and the code can be navigated back and forth, discovered and used by the IDE to make suggestions.

## The Magical approach

From now on I will take a [really simple web application](https://github.com/AlessioCoser/escaping-frameworks-magics/tree/01-START) written using the [Spring Boot](https://spring.io/projects/spring-boot) framework as an example. This doesn't mean that Spring Boot sucks, it is a great ecosystem. I could have used any other magical framework.

Let's focus on the configurations (aka properties) then.

**application.properties**
```
server.port=8080
example.initial=${INITIAL:0}
```

Wherever I want to use those properties I need to add an annotation to let the framework inject the value, and I also need to instrument the injection to parse that value (a string) as an integer.

**CounterService.kt**
```kotlin
@Service
class CounterService {

    @Value("#{new Integer('\${example.initial}')}")
    private val initial: Int? = null
    private val atomic: AtomicLong = AtomicLong((initial ?: 0).toLong())

    fun incrementAndGet(): Long {
        return atomic.incrementAndGet()
    }
}
```
([See this version - tag `01-START`](https://github.com/AlessioCoser/escaping-frameworks-magics/tree/01-START/src/main/kotlin/com/example))

I see a lot of magic in these lines, all of which is handled and checked at runtime.
- **Is `example.initial` defined in the configuration?** The IDE can't tell me, maybe there is a plugin for this or I have to pay for this IDE feature.
- **Did I write the conversion logic right?** Again, the IDE can not tell me. If it were not in the annotation string, the IDE could have suggested something to me while I was writing.
- **Can I easily reach the code line where the `example.initial` configuration is set?** Once again, the IDE cannot help me, but if I pay maybe...

I generally see the need to install an IDE plugin for a framework as a sign that the magic involved in the framework is a bit too powerful for my IDE/compiler. It means that a lot of things happen at runtime, and I need to pollute my classes with `null` fields even when those fields should never be `null`.

Besides magics, I'm also coupling the `CounterService` class with the property reading and transformation.

## The plain approach

My application should be **environment agnostic**. A single artifact that can be used anywhere, without the need to build it again, changing only the `System ENV vars`. It will have no reference to any environment. It will not know whether it is staging, production, or anything else. With this approach we are moving in the direction described by [12 factor app](https://12factor.net/config).

So I can simply define a `Properties` class that reads values from the system environment, and I register it using the `@Service` annotation.

**Properties.kt**
```kotlin
@Service
data class Properties(
    val serverPort: Int = 8080,
    val exampleInitial: Int = (getenv("INITIAL") ?: "0").toInt()
)
```

This way I can extract the logic that retrieves the `initial` property from `CounterService` and I can inject the dependency using the Springboot constructor injection.

**CounterService.kt**
```kotlin
@Service
class CounterService(props: Properties) {
    private val atomic = AtomicLong(props.exampleInitial.toLong())

    fun incrementAndGet(): Long {
        return atomic.incrementAndGet()
    }
}
```
([See this version - tag `02-CONFIG`](https://github.com/AlessioCoser/escaping-frameworks-magics/tree/02-CONFIG/src/main/kotlin/com/example))

**I did not write more code, actually I wrote less code and I have no more `nulls` in the service**. Besides, I have the great help of the IDE/compiler to find the property I need or to **navigate back and forth between properties and their usages**. I also transform the value from string to number directly in the code, which can be tested or extracted whenever I feel the need.

Now, things start to get a bit weird when I try to manage the framework's default properties like `server.port`. For that I needed to create an `AppConfig` class where I register the `Properties` bean and override the `server.port` property using that bean.

This is needed only because I want to handle the configuration of `server.port` in the same way as the other properties. The framework in this case works against us a bit in terms of conciseness and readability of the code (and this is fair). **We will see in the next articles how we can evolve this part**.

```kotlin
@Configuration
class AppConfig : WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {

    @Bean
    fun properties() = Properties()

    override fun customize(factory: ConfigurableServletWebServerFactory) {
        factory.setPort(properties().serverPort)
    }
}
```
([See this version - tag `03-CONFIG-PORT`](https://github.com/AlessioCoser/escaping-frameworks-magics/tree/03-CONFIG-PORT/src/main/kotlin/com/example))

With `@Bean fun properties()` I register the Properties class as an injectable dependency. The effect is similar to the `@Service` annotation. So I can remove that annotation from the `Properties` class.

Deep down I like having an `AppConfig` where to register all the dependencies. It feels a little less magic, because I'm explicitly wiring the dependencies. This helps me to be more aware of how I am designing the dependency graph and how many dependencies I have for a single bean.

**Anyway, I will cover more on this topic in the next article dedicated to the magical dependency injection.**

## Conclusions

It is definitely uncommon to see a Spring Boot application that does not use `application.properties`.

In fact I don't want you to do the same in your own projects. I do want you to realize that **sometimes we choose a framework to do things that can be done just fine (if not better) without the help of its magic tricks**.

Then it will be **up to you which framework to choose and whether to use it**. In doing so I hope that your next choice will take this series of articles into consideration.

Next month we will see how to get rid of another framework magical feature.

**Stay tuned :)**

Please let me know your opinion <br /><script type="IN/Share" data-url="{{site.url}}{{page.url}}"></script> this article on LinkedIn and tag me if you find it interesting.
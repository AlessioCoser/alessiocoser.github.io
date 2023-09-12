---
layout: post
title:  "Escaping the magics of the frameworks: 2. HTTP Routing"
author: alessio
categories: [ frameworks, springboot, jvm, kotlin, annotations, controllers ]
image: assets/images/escaping-frameworks-magics-2.jpg
---
<script src="https://platform.linkedin.com/in.js" type="text/javascript">lang: en_US</script>

Most of the time, in a framework-based application, when I want to handle an HTTP request on a path, I have to annotate the controller class and methods. This way the framework knows, only at runtime, how to call our methods **according to the annotated logic**.

<center>Is the routes registration mechanism so complicate? Why do we want to hide it? What happens if we make it explicit?</center>

In this series of articles I'm **replacing, magic by magic, some framework's features** until I have no more sorceries to wield. I'm doing it in **baby steps**, explaining the reasons why, and keeping **keeping my application working** all the time.

I know, what I'm doing is an **unconventional use of Spring Boot**, but I could have used any other framework. I chose this one because it's one of the most popular.

Also, it is interesting to see that **Spring Boot itself offers different ways to do the same thing**. So, let's see what changes when we remove the magics in terms of readability, simplicity, expressiveness, and verbosity.

You can read about **frameworks** and **what's wrong with magics** in the previous article. I also tackled the magical configuration. I recommend you read it, if you haven't already. From now on I will continue where I left off with the code examples tackling the refactoring of the controllers.

> [Escaping the magics of the frameworks: 1. Configuration](/escaping-the-magic-of-the-frameworks-1-configuration/)

Please <script type="IN/Share" alt="share" data-url="{{site.url}}{{page.url}}"></script> **this article on LinkedIn and tag me** if you want to start a public conversation on this topic or [**contact me**](/contact) if you prefer to talk about it privately.

## The Controller

The most common way to handle an HTTP request in Spring Boot is to use a Controller.

As you can see in the [example project](https://github.com/AlessioCoser/escaping-frameworks-magics/tree/03-CONFIG-PORT/src/main/kotlin/com/example) I listen for HTTP requests coming at the `/greeting` path through the `GreetingController`. The framework then parses the request, calls some logic to do the work, parses again the result and sends it as an HTTP response.

**GreetingController.kt**
```kotlin
@RestController
@RequestMapping("greeting")
class GreetingController(private val counterService: CounterService) {

    @GetMapping
    fun greeting(
        @RequestParam(value = "name", defaultValue = "World") name: String
    ): Greeting {
        return Greeting(counterService.incrementAndGet(), "Hello, ${name}!")
    }
}
```

Actually there is hardly a line without magic. So let's try refactoring the code to make the logic more explicit.

## @RequestParam
It is really necessary to have an annotation to get a parameter `name` from the request and use the default value `World` if it is not present? This is exactly what `@RequestParam` is doing.

What if there is a way to receive directly the request as parameter?

**GreetingController.kt**
```kotlin
    @GetMapping
    fun greeting(request: HttpServletRequest, response: HttpServletResponse): Greeting {
        val name = request.getParameter("name") ?: "World"
        return Greeting(counterService.incrementAndGet(), "Hello, ${name}!")
    }
```
([See the diff - tag `04-REQUEST-PARAM`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/03-CONFIG-PORT...04-REQUEST-PARAM?diff=unified))

Note that the request parameter already has all the methods I need. I can also use the code to fallback to a default value.

_"The code"_... what an innovation! I can compile it, test it, debug it, extract it, do wathever I want in order to access anything from the request.

## @RestController
This annotation is used to register the controller to the framework. It is a specialized version of `@Controller`. If you use it on the class you don't need to annotate every method/endpoint with `@ResponseBody` beacuse it will be automagically activated (I will write about it right below).

Now, to get rid of `@RestController` I can do a small step. I can explicitly use those hidden annotations: `@Controller` on the class and `@ResponseBody` on the greeting method.

([See the diff - tag `05-REST-CONTROLLER`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/04-REQUEST-PARAM...05-REST-CONTROLLER?diff=unified))

## @ResponseBody
What does `ResponseBody` mean? It means that the controller can return a class instance, because the framework takes that instance, wraps it in a ResponseEntity object, converts it to json, and puts it in the response body.

We can directly return the `ResponseEntity` class. Actually I found out there are a lot of helper methods to create a `ResponseEntity` in an expressive way.

**GreetingController.kt**
```kotlin
    @GetMapping
    fun greeting(request: HttpServletRequest, response: HttpServletResponse): ResponseEntity<Greeting> {
        val name = request.getParameter("name") ?: "World"
        return ok().body(Greeting(counterService.incrementAndGet(), "Hello, ${name}!"))
    }
```
([See the diff - tag `06-RESPONSE-BODY`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/05-REST-CONTROLLER...06-RESPONSE-BODY?diff=unified))

Doing this I can also choose which status I want to use as response. I used `ServerResponse.ok()` method to instantiate a `200 OK` response.

## @Controller & co.
Now I only left out the `@Controller`, the `@RequestMapping("greeting")` and the `@GetMapping`. These annotations are bound together so it is more practical to remove them all at once:

- I've already wrote about the `@Controller` annotation that registers the controller to the framework;
- The `@RequestMapping("greeting")` annotation, instead, tells the framework which route the controller is responsible for: `/greeting`;
- `@GetMapping`, at last, registers the annotated method as a controller route so that it will be called as soon as the service receives a `GET` request.

To reduce its magical grip on the code, I can transform the `@Controller` into a normal `@Component`, but doing so I need a more explicit way to register the routes.

Actually there is a way to register a router as a `@Bean` (dependency) and Spring Boot managed to provide `RouterFunctionDsl` class for it that also improves the readability.

**Application.kt**
```kotlin
@Configuration
@SpringBootApplication
class Application {
    @Bean
    fun router(greetingController: GreetingController): RouterFunction<ServerResponse> {
        return router {
            GET("/greeting") { greetingController.greeting(it) }
        }
    }
}
```

I prefer to have the routes registration in the main application component. In this way I can always start from the main and it is sure that I can be able to navigate down wherever I want to go. I'm able to **understand the dependency graph and the flow of the request easily**, starting from the main, following the path, looking at the controller and going down until I reach the core logic.

I also have the power of the IDE that suggests me what methods of the router I can use. Luckily Spring Boot, with `RouterFunctionDsl`, gives me also a simple way to define the routes: it's just a function `(ServerRequest) -> ServerResponse`. For me it is really short and expressive.

Therefore I have to change the controller's method signature to adapt it to the new one.

**GreetingController.kt**
```kotlin
@Component
class GreetingController(private val counterService: CounterService) {

    private fun greeting(request: ServerRequest): ServerResponse {
        val name = request.param("name").orElse("World")
        return ok().body(Greeting(counterService.incrementAndGet(), "Hello, ${name}!"))
    }
}
```

([See the diff - tag `07-ROUTING`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/f8e9d7f4ca8e56e5746add661434678562e8ed4b...07-ROUTING?diff=unified))

## Routing
Now I can choose whether to stick to this implementation or move inside the controller the knowledge on which path a controller has to handle. There are pros and cons in both cases, I choose the second option for three reasons:
1. It is a responsibility of the controller to know which base path to handle
2. In the router I only need to register the controllers without knowing which routes every controller has to handle
3. I want to have my application designed as similarly as possible to the classical SpringBoot application.

So I can delegate to the controller through the `routes` method the routes registration of a specific controller in this way:

**Application.kt**
```kotlin
    @Bean
    fun router(greeting: GreetingController): RouterFunction<ServerResponse> {
        return router {
            greeting.routes(this)
        }
    }
```

In the controller I can now add the routes method. In this way I have all the controller routes coded and discoverable in one single method. I can also make all the private routes enforcing the cohesion of the component.

**GreetingController.kt**
```kotlin
@Component
class GreetingController(private val counterService: CounterService) {

    fun routes(router: RouterFunctionDsl) {
        router.GET("/greeting", ::greeting)
    }

    private fun greeting(request: ServerRequest): ServerResponse {
        val name = request.param("name").orElse("World")
        return ok().body(Greeting(counterService.incrementAndGet(), "Hello, ${name}!"))
    }
}
```

([See the diff - tag `08-ROUTING`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/07-ROUTING...08-ROUTING?diff=unified))

What I've done here is changing the way I'm interacting with the already present router: from a framework-style (with annotations) to a library-style, so I used the router as a dependency, and I took the control over the registration of the routes.

## Conclusions

I'm almost halfway there, I reduced properties and controllers to just simple `@Component` (aka Beans or dependencies).

I kept an architecture similar to the Spring Boot one. So there is a central configuration accessible to all the components, and the controllers still have the responsibility of their routes definitions.

_Have I added more code?_<br />
Yes, but the code is more explicit, and the parts I really added are only about the server port configuration and the router definition. However, I'm on the way to change them again in the next articles.

_Have I added unnecessary boilerplate or complexity?_<br />
I don't think so. It actually seems less complex to me now. It makes the flow and the dependencies of the application clearer. I can read the code without disturbing my memory to remember some conventions because I can easily navigate it back and forth.

**What's next?**<br />
I want to deal with the core magic of the framework: the Beans. To do it in an incremental way, however, I need to decouple my tests from the way the application is run in the first place.

**Stay tuned :)**

Please let me know your opinion <br /><script type="IN/Share" alt="share" data-url="{{site.url}}{{page.url}}"></script> this article on LinkedIn and tag me if you find it interesting.
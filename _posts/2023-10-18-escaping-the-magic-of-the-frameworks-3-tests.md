---
layout: post
title:  "Escaping the magics of the frameworks: 3. Tests"
author: alessio
categories: [ frameworks, springboot, jvm, kotlin, annotations, tests ]
image: assets/images/escaping-frameworks-magics-3.jpg
---
<script src="https://platform.linkedin.com/in.js" type="text/javascript">lang: en_US</script>

When I want to test a **framework-based application**, I ran into some magics instructing the framework to run in a test environment. Usually I also need to receive from its dependency mechanism the client mock that simulates an http request.

<center>What if I treat the application as a black box in my tests?</center>

I could perform an http request and expect an http response. The tests would be **less coupled to the implementation details** and I would be able to refactor the application more easily.

This is why I am doing this now. I want to be able to refactor the application further and get rid of all the magic in the most comfortable way possible and **without breaking the tests**.

In this series of articles I'm **replacing, magic by magic, some framework's features** until I have no more sorceries to wield. I'm doing it in **baby steps**, explaining the reasons why, and **keeping my application working** all the time.

You can read about **frameworks** and **what's wrong with magics** in the previous articles. I also tackled the magical configuration and the power of explicitness in the routing. From now on I will continue where I left off with the code examples tackling the **refactoring** of the tests in order to decouple them from the implementation.

> [Escaping the magics of the frameworks: 1. Configuration](/escaping-the-magic-of-the-frameworks-1-configuration/)
>
> [Escaping the magics of the frameworks: 2. HTTP Routing](/escaping-the-magic-of-the-frameworks-2-http-routing/)

Please <script type="IN/Share" alt="share" data-url="{{site.url}}{{page.url}}"></script> **this article on LinkedIn and tag me** if you want to start a public conversation on this topic or [**contact me**](/contact) if you prefer to talk about it privately.

## Take back control of my application

The first step to decouple the tests from the Spring Boot implementation details is to take back control of my application.

I started from this situation:

**Application.kt**
```kotlin
fun main(args: Array<String>) {
    SpringApplication.run(Application::class.java, *args)
}

@Configuration
@SpringBootApplication
class Application {
    @Bean
    fun router(greeting: GreetingController): RouterFunction<ServerResponse> {
        return router {
            greeting.routes(this)
        }
    }
}
```

It sounds natural to delegate the application start and stop to the `Application` class itself. It's a behaviour that an app should have.

**Application.kt**
```kotlin
fun main(args: Array<String>) {
    Application().start(args)
}

@Configuration
@SpringBootApplication
class Application {
    private var appContext: ConfigurableApplicationContext? = null
    private val app = SpringApplication(Application::class.java)

    @Bean
    fun router(greeting: GreetingController): RouterFunction<ServerResponse> {
        return router {
            greeting.routes(this)
        }
    }

    fun start(args: Array<String>) {
        appContext = app.run(*args)
    }

    fun stop() {
        appContext?.close()
    }
}
```
([See the diff - tag `09-CONTROL-APP`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/4ab5eca136c...09-CONTROL-APP?diff=unified))

I just added two methods: `start()` and `stop()`. To make them effective, though, I have to keep the `SpringApplication` as a private field along with the `ConfigurableApplicationContext` that I instantiate when I start the app and I use when I need to stop the app.

## Decouple the test assertions
The `MockMvc` class mocks the requests made to the `SpringBootApplication` along with the responses, and provides a way to chain various assertions about the response. It can also check the json structure and values.

It's a Spring Boot tool that is injected at runtime and it is totally dependent on the framework magic: `@SpringBootTest`.

However I'm going to get rid of this magic, so I cannot rely on `MockMvc` anymore.

As a first step I can change the way I write the assertions. This will free them from the framework's grip.

I have this test file:

**GreetingControllerTests.kt**
```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class GreetingControllerTests {
  @Autowired
  private val mockMvc: MockMvc? = null

  @Test
  fun noParamGreetingShouldReturnDefaultMessage() {
      mockMvc!!.perform(MockMvcRequestBuilders
          .get("/greeting"))
          .andExpect(status().isOk())
          .andExpect(jsonPath("$.content").value("Hello, World!"))
  }
  // ...
}
```

I'm not a big fan of the json matchers, I think this kind of test should be as straightforward as possible. For example I could verify the whole body or check that the body contains a string like `"content": "Hello, World!"`.

However, I also see its usefulness. It helps to verify the json format correctness and simplifies the assertion of more complex json strings.

Now, I want to keep the same json matching feature, but I don't want to write one myself. Luckily I found a library that does exactly what I need: `json-unit-assertj`. So, I added it to the `build.gradle` file.

**GreetingControllerTests.kt**
```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class GreetingControllerTests {
  @Autowired
  private val mockMvc: MockMvc? = null

  @Test
  fun noParamGreetingShouldReturnDefaultMessage() {
      val response = mockMvc!!.perform(get("/greeting")).andReturn().response

      assertThat(response.status).isEqualTo(200)
      assertThatJson(response.contentAsString).inPath("$.content").isEqualTo("Hello, World!")
  }
  // ...
}
```
([See the diff - tag `10-ACT-ASSERT`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/09-CONTROL-APP...10-ACT-ASSERT?diff=unified))

Now I'm using the `mockMvc` only to perform the get request, in that way it will be easier to get rid of it later.

## Start the Application
The tests are still coupled to the Spring Boot internalities with some annotations and an injection.

Let's remove all the Spring Boot annotations `@SpringBootTest` and `@AutoConfigureMockMvc`, together with `MockMvc`.

To do so I have to change 2 things:

**1. I must have the application running**<br />
I create an `Application` instance in the test file, I start it before running all tests and stop it after all tests in the file are run. I used `@BeforeAll` and `@AfterAll` JUnit annotations.

**2. I must perform a real HTTP request**<br />
I use the built-in Spring Boot `RestTemplate` class to perform a GET request. I only need to change the way I retrieve the response body and the statusCode.

**GreetingControllerTests.kt**
```kotlin
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class GreetingControllerTests {
    private val application = Application()

    @Test
    fun noParamGreetingShouldReturnDefaultMessage() {
        val response = get("/greeting")
        assertThat(response.statusCode.value()).isEqualTo(200)
        assertThatJson(response.body).inPath("$.content").isEqualTo("Hello, World!")
    }

    @Test
    fun paramGreetingShouldReturnTailoredMessage() {
        val response = get("/greeting?name=Spring Community")
        assertThat(response.statusCode.value()).isEqualTo(200)
        assertThatJson(response.body).inPath("$.content").isEqualTo("Hello, Spring Community!")
    }

    @BeforeAll
    fun beforeAll() {
        application.start(emptyArray())
    }

    @AfterAll
    fun afterAll() {
        application.stop()
    }

    private fun get(uri: String): ResponseEntity<String> {
        return RestTemplate().getForEntity("http://localhost:8080$uri", String::class.java)
    }
}
```
([See the diff - tag `11-START-APP`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/10-ACT-ASSERT...11-START-APP?diff=unified))

I don't like so much the way `RestTemplate` performs the requests, it is not so expressive. For now I chose to create a private method to make the tests more readable, but I will replace it soon.

## Change HTTP Client
Some time ago I created a really simple HTTP client for Kotlin: [Topinambur](https://github.com/DaikonWeb/topinambur). I kept it as simple as possible to use and to maintain. It can be used statically or it can be instantiated with some specific defaults like the `baseUrl`.

I decided to replace `RestTemplate` with that library. Of course any other HTTP client will do, the important thing is to have the option to create an instance with a baseUrl without the need to repeat it on every request (it will be useful later).

**GreetingControllerTests.kt**
```kotlin
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class GreetingControllerTests {
    private val application = Application()
    private val client = Http(baseUrl = "http://localhost:8080")

    @Test
    fun noParamGreetingShouldReturnDefaultMessage() {
        val response = client.get("/greeting")
        assertThat(response.statusCode).isEqualTo(200)
        assertThatJson(response.body).inPath("$.content").isEqualTo("Hello, World!")
    }

    @Test
    fun paramGreetingShouldReturnTailoredMessage() {
        val response = client.get("/greeting?name=Spring Community")
        assertThat(response.statusCode).isEqualTo(200)
        assertThatJson(response.body).inPath("$.content").isEqualTo("Hello, Spring Community!")
    }

    @BeforeAll
    fun beforeAll() {
        application.start(emptyArray())
    }

    @AfterAll
    fun afterAll() {
        application.stop()
    }
}
```
([See the diff - tag `12-HTTP-CLIENT`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/11-START-APP...12-HTTP-CLIENT?diff=unified))

## Start and stop the app on each test
The tests as they are now could be ok. There is a **single app instance** that answers to all the tests (first approach). That's fine. However I would like to instantiate a **new application instance for each test** (second approach).

Both approaches are good and can be used in different contexts for different reasons.

I would choose the **first approach** when I can have the same application instance on every test and I need faster tests. The application is started at the beginning of the test file and closed at the end.

I prefer the **second approach** when I already have a fast Application startup (usually when the app doesn't inject dependencies at runtime) and when it is important to have the tests completely isolated. The application is started before and stopped after each single test.

Let's go for the second approach.

I thought that I could start and stop the application with `@BeforeEach` and  `@AfterEach` test annotations, but I would like to do it programmatically without annotations.

In Kotlin there is a way to automatically close a resource. It's like a `try-with-resources` Java statement, but way more expressive. It is [.use](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html).

To apply `.use` I need to have an `AutoCloseable` resource. So my `Application` needs to extend the `AutoCloseable` class and it needs to have an `override fun close()` method that replaces the previous `fun stop()` method.

**Application.kt**
```kotlin
class Application: AutoCloseable {

  // ...

  override fun close() {
      appContext?.close()
  }
}
```

Now the `Application` is ready to be closed automatically. Let's change the tests:

**GreetingControllerTests.kt**
```kotlin
class GreetingControllerTests {

    @Test
    fun noParamGreetingShouldReturnDefaultMessage() = runApp { client ->
        val response = client.get("/greeting")
        assertThat(response.statusCode).isEqualTo(200)
        assertThatJson(response.body).inPath("$.content").isEqualTo("Hello, World!")
    }

    @Test
    fun paramGreetingShouldReturnTailoredMessage() = runApp { client ->
        val response = client.get("/greeting?name=Spring Community")
        assertThat(response.statusCode).isEqualTo(200)
        assertThatJson(response.body).inPath("$.content").isEqualTo("Hello, Spring Community!")
    }

    private fun runApp(test: (client: Http) -> Unit) {
        val client = Http("http://localhost:8080")
        Application()
            .start(emptyArray())
            .use { test(client) }
    }
}
```
([See the diff - tag `13-RUN-APP`](https://github.com/AlessioCoser/escaping-frameworks-magics/compare/12-HTTP-CLIENT...13-RUN-APP?diff=unified))

I only needed to create a private `runApp` function that receives another function as the first argument. The argument is the test body that needs to be wrapped.

In `runApp` I instantiate a client, I start the application, and I run the test block inside the `.use` statement with the client as argument.

In that way I only need to use the `runApp` function on every test that needs a running Application.

The function starts the application, gives the configured HTTP client to the test block, and closes the server after each test block.

The interesting thing is that it's only a function, it's not an annotation. Not magic. Only Kotlin code.

## Conclusions

You have seen how I changed the way tests interact with the application.

Beforehand, I was using a framework tool that **simulates an http request**. The tests needed to know that it's a Spring Boot application, but they were quite fast even if the app had to instantiate the dependencies at runtime.

Now I'm starting the **real application** in the tests and I'm performing a **real HTTP request** on each test. The tests treat the Application as a black box, they are totally decoupled from the application, but they are a bit slower.

**I think it's a matter of needs**. For this series of articles I need to be free to change my application structure without breaking my safety net that checks if everything is still working.

**What's next?**<br />
Now I'm ready to make the greatest refactoring of the series. I will reduce the magic around the Spring Boot dependency manager. It's still out of my control. It's still magic.

**Stay tuned :)**

Please let me know your opinion <br /><script type="IN/Share" alt="share" data-url="{{site.url}}{{page.url}}"></script> this article on LinkedIn and tag me if you find it interesting.
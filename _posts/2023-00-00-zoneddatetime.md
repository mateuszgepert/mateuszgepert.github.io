---
layout: post
title: When you think you know how asserJ works
subtitle: about serialization of ZonedDateTime
date: 2023-03-12 13:51:13-0400
background: '/img/posts/zoneddatetime.jpg'
---

Lets say that you need to store particular `json` in database, you had newest spring, kotlin, mysql in version > 8. All
in place. Nothing easier simply create table `json_content`. For now forget about sql we will get back to that in next
episode ;)

Next you pick good old `jackon` and create `ObjectMapper` maybe even additional `JavaTimeModule()` and
disable `SerializationFeature.WRITE_DATES_AS_TIMESTAMPS` - yes smart head I do remember how to serialize timestamps!!
and I know it's kotlin won't forget about kotlin module as well.

```kotlin
private val objectMapper = ObjectMapper()
    .registerModule(JavaTimeModule())
    .registerModule(KotlinModule.Builder().build())
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
```

One of requirement would be to make a support for `ZonedDateTime`. Pfff don't make me fool I know how to do so. You had
unit tests support so quickly you create a perfect valid test with checking that such timestamps.

```kotlin
@Test
fun `should correctly serialize and deserialize dates with non-UTC timezone`() {
    val asString = "2023-03-08T18:29:45.621+08:00[Asia/Shanghai]"
    val withTimeZone = ZonedDateTime.parse(asString)

    val serialized = objectMapper.writeValueAsString(withTimeZone)
    val deserialized = objectMapper.readValue(serialized, ZonedDateTime::class.java)

    assertThat(withTimeZone).isEqualTo(deserialized)
}
```

Now what is wrong with that test? It's perfectly green

```bash
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.ZonedDateTimeSerializationDeserializationTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.377 s - in com.example.ZonedDateTimeSerializationDeserializationTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
```

So we're good ship it!!! But maybe some of you might think wait we shouldn't test datetime in isolation! Lets check the
actual `json`
So here is the test

```kotlin
@Test
fun `should correctly serialize and deserialize object with ZoneDateTime field`() {
    val withZonedDateTime = WithZonedDateTime(ZonedDateTime.now())

    val serialized = mapper.writeValueAsString(withZonedDateTime)
    val deserialized = mapper.readValue(serialized, WithZonedDateTime::class.java)

    assertThat(withZonedDateTime).isEqualTo(deserialized)
}
```

with following dummy data structure

```kotlin
data class WithZonedDateTime(val timestamp: ZonedDateTime)
```

and boom

```bash
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.ZonedDateTimeSerializationDeserializationTest
[ERROR] Tests run: 2, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.894 s <<< FAILURE! - in com.example.ZonedDateTimeSerializationDeserializationTest
[ERROR] should correctly serialize and deserialize object with ZoneDateTime field  Time elapsed: 0.252 s  <<< FAILURE!
org.opentest4j.AssertionFailedError:

expected: WithZonedDateTime(timestamp=2023-03-17T16:01:43.789816Z[UTC])
 but was: WithZonedDateTime(timestamp=2023-03-17T17:01:43.789816+01:00[Europe/Warsaw])
	at com.example.ZonedDateTimeSerializationDeserializationTest.should correctly serialize and deserialize object with ZoneDateTime field(ZonedDateTimeSerializationDeserializationTest.kt:36)

[INFO]
[INFO] Results:
[INFO]
[ERROR] Failures:
[ERROR]   ZonedDateTimeSerializationDeserializationTest.should correctly serialize and deserialize object with ZoneDateTime field:36
expected: WithZonedDateTime(timestamp=2023-03-17T16:01:43.789816Z[UTC])
 but was: WithZonedDateTime(timestamp=2023-03-17T17:01:43.789816+01:00[Europe/Warsaw])
[INFO]
[ERROR] Tests run: 2, Failures: 1, Errors: 0, Skipped: 0
```

Uff good that we decided to add that second test.

But on the other hand wait, WTF?!?!?! Why first assertion works and the second is not?
Well digging a bit deeper we might find out that `assertJ` is using `ChronoZonedDateTime` compare method which in the
end compares just `Long.compare(dateTime1.toLocalTime().getNano(), dateTime2.toLocalTime().getNano())` and since they
are in the same point of time it's perfectly valid.
But why our object comparison is falling? Well in nutshell for our class assertj decides to
use `StandardComparisonStrategy` where in the end `actual.equals(other)` is used.

Keeping that in mind lets try change assertion in our first test case to

```kotlin
assertThat(withTimeZone == deserialized).isTrue
```

And finally test will fail. The last thing is to tweak `ObjectMapper` with necessary features customisation

```kotlin
private val objectMapper = ObjectMapper()
    .registerModule(JavaTimeModule())
    .registerModule(KotlinModule.Builder().build())
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
    .enable(SerializationFeature.WRITE_DATES_WITH_ZONE_ID)
    .disable(DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE)
```

As lesson learned always test actual requirement!

Imagine that under the hood there is some conditional business logic that relies on such check.
That bugs are pretty hard to spot since the logic itself is somehow correct if objects are not the same then we're good
and no need to take any action.

It's first case from one weird issue I encountered so expect further updates about
that ;) 

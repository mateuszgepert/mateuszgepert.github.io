---
layout: post
title: "Deserialization of ZonedDateTime"
subtitle: "Liskov substitution principle at place"
date: 2023-03-12 18:32:13-0400
background: '/img/posts/hello-world.jpg'
---

You probably know the `Liskov substitution principle`. Either from book, school or interview. You probably remember even
the naive examples with lists, fruit classes or whatever.
But today I want to show you real life production issue that happened. And why mocking everything is not a good
practice.

Lets start with some problem background. Imagine that there is application that periodically sends notifications to
other systems at fixed time about what has happened in the origin system. But one of the requirement was to skip
duplicates if they occur. Consider such simple processor:

```kotlin
class NotificationsProcessor {

    private val repository: NotificationsRepository
    private val notificationsDeserializer: NotificationsDeserializer
    private val notifier: Notifier

    fun process(notification: Notification) =
        notification
            .takeIf { isDuplicate(it) }
            ?.let { notifier.notify(it) }

    private fun isDuplicate(notification: Notification?) =
        notification.takeIf { it != null }
            ?.let { repository.findLastNotification() }
            ?.let { notificationsDeserializer::deserialize }
            ?.isDuplicate(notification)
            ?: false

    // ...
}

```

As you can see it's pretty straightforward. In case of new notification we simply check if it's duplicate by fetching
last notification and checking if that's duplicate. If it's process should be skipped and nothing done in other way we
want to notify and save notification in DB.

From design perspective it's clear that responsibility of that class is just to determine if notification should be
processed. All is pretty well encapsulated behind dedicated objects. One can said that it's pretty neat design. 
I thought as well. All of those components are pretty well unit tested. 
But looking into setup method of that processor tests I see first red flag

```kotlin
class NotificationProcessorTest {

    private val repository: NotificationsRepository = mockk()
    private val notificationsDeserializer: NotificationsDeserializer = mockk()
    private val notifier: Notifier = mockk()
    private lateinit var notificationProcessor: NotificationProcessor

    @BeforeEach
    fun setup() {
        notificationProcessor = NotificationProcessorTest(repository, notificationsDeserializer, notifier)
        every { repository.findLastNotification() } returns "some notification"
        every { notificationsDeserializer.deserialize(eq("some notification")) } returns notification()
        every { notifier.notify(any()) } returns result
    }
    
    // all tests that covers skips non skips scenarios with correct notification equality check
}
```

and following `NotificationDeserializer` class test

```kotlin
class NotificationDeserializerTest {

    private val objectMapper: ObjectMapper = ApplicationConfig().objectMapper()
    private val deserializer: NotificationDeserializer = NotificationDeserializer(objcetMapper)

    // all tests that checks null, empty string, that invalid fields should not fail deserialization
}

```

and finally simplified representation of `Notification` class

```kotlin
class Notification(
    private val message: String,
    private val uuid: UUID,
    private val timestamp: ZonedDateTime,
) {

    fun isDuplicate(other: Notification) =
        this.copy(uuid = other.uuid) == other
}

```

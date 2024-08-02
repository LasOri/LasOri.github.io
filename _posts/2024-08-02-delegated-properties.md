---
title: Delegated properties in Kotlin
date: 2024-08-02 04:00:00 PM
categories: [dev]
tags: [kotlin, delegated properties, dev]
---

### Delegated Properties
Properties in Kotlin are fundamental building blocks of everyday software development. They are powerful on their own, and their setters and getters can be customized or, with a few keywords, initialized later or even loaded lazily.

Property delegation allows for even more customization. In a way, it's somewhat similar to Property Wrappers in Swift (without the projected value and annotation). Both solutions focus on easily adding logic to an existing property.

### A simple example
Let's say we have this class:
```kotlin
class PropertyDelegationExample {

    var number: Int = 0
    var text: String? = null

}
```
For some reasons, in another part of my code, I want to print to the console every time the number is even and every time the text contains the @ character. Of course, I have many options to do this, but let's assume for the sake of the example that I want to do it with property delegation. So, let's create a property delegator that implements a simple observer pattern to help us with this task.

We will need a callback that we can use to listen for value changes:
```kotlin
typealias Listener<Value> = (Value) -> Unit
```
And the property delegator itself, which has two necessary functions, `getValue` and `setValue`:
```kotlin
class ObserverDelegate {
    operator fun <Value>getValue(thisRef: Any?, property: KProperty<*>): Value {}
    operator fun <Value>setValue(thisRef: Any?, property: KProperty<*>, value: Value) {}
}
```
If we want to handle more than one type and more than one callback, we will need to store them:
```kotlin
class ObserverDelegate {

    private val listeners = mutableMapOf<String, MutableList<Listener<*>>>()
    private val values = mutableMapOf<String, Any?>()

    operator fun <Value>getValue(thisRef: Any?, property: KProperty<*>): Value {}
    operator fun <Value>setValue(thisRef: Any?, property: KProperty<*>, value: Value) {}
}
```
And of course, we need a way to add and remove listeners:
```kotlin
class ObserverDelegate {

    private val listeners = mutableMapOf<String, MutableList<Listener<*>>>()
    private val values = mutableMapOf<String, Any?>()

    operator fun <Value>getValue(thisRef: Any?, property: KProperty<*>): Value {}
    operator fun <Value>setValue(thisRef: Any?, property: KProperty<*>, value: Value) {}

    fun <Value>listen(property: KProperty<Value>, listener: Listener<Value>) {
        var listeners: MutableList<Listener<Value>>? = this.listeners[property.toString()] as MutableList<Listener<Value>>?
        if (listeners == null) {
            listeners = mutableListOf()
        }
        listeners.add(Listener)
        this.listeners[property.toString()] = listeners as MutableList<Listener<*>>
    }

    fun <Value>leave(property: KProperty<Value>, listener: Listener<Value>) {
        this.listeners[property.toString()]?.remove(Listener)
    }
}
```
Let's pause for a moment to understand what we've created here.

The listen function is generic over the Value type, taking two parameters: a property representation and a callback. The generic constraint on the callback ensures it's compatible with the property's type.

We use `property.toString()` as a key because the `KProperty` instances passed to `getValue`/`setValue`, `listen`/`leave` are different even for the same property. We can't rely on equality or access the property's receiver directly. It could be more elegant, Using the string representation is practical, and for now just simply good enough.

The core functionality is straightforward: we store callbacks in a list associated with the property key.

The `setValue` and `getValue` is also simple:
```kotlin
@Suppress("UNCHECKED_CAST")
class ObserverDelegate {

    private val listeners = mutableMapOf<String, MutableList<Listener<*>>>()
    private val values = mutableMapOf<String, Any?>()

    operator fun <Value>getValue(thisRef: Any?, property: KProperty<*>): Value {
        return values[property.toString()] as Value
    }

    operator fun <Value>setValue(thisRef: Any?, property: KProperty<*>, value: Value) {
        values[property.toString()] = value
        listeners[property.toString()]?.forEach { Listener ->
            (Listener as Listener<Value>)(value)
        }
    }

    fun <Value>listen(property: KProperty<Value>, listener: Listener<Value>) {
        var listeners: MutableList<Listener<Value>>? = this.listeners[property.toString()] as MutableList<Listener<Value>>?
        if (listeners == null) {
            listeners = mutableListOf()
        }
        listeners.add(Listener)
        this.listeners[property.toString()] = listeners as MutableList<Listener<*>>
    }

    fun <Value>leave(property: KProperty<Value>, listener: Listener<Value>) {
        this.listeners[property.toString()]?.remove(Listener)
    }
}
```
In the getter, we simply retrieve the current value of the property and return it. In the setter, we first set the new value and then iterate over all registered listeners, notifying them of the change by passing the new value.

We've added an unchecked cast suppression warning to the class. However, this should be safe because the property type remains consistent, and the callback type is tightly coupled to it, preventing type mismatches.

So let's use our delegate:
```kotlin
class PropertyDelegationExample {

    val observer = ObserverDelegate()
    var number: Int by observer
    var text: String? by observer
}
```
and listen to the value changes:
```kotlin
// ...
val propertyDelegationExample = PropertyDelegationExample()

propertyDelegationExample.observer.listen(propertyDelegationExample::number) {
    if (it % 2 == 0) {
        println("Number is even: $it")
    }
}
propertyDelegationExample.observer.listen(propertyDelegationExample::text) {
    if (it != null && it.contains("@")) {
        println("Text contains @: $it")
    }
}

propertyDelegationExample.number = 42 // Number is even: 42

propertyDelegationExample.text = null

propertyDelegationExample.text = "test@test" // Text contains @: test@test

// ...
```

### Conclusion
I know, this is just a simple example, but hopefully, it's enough to spark some creative thinking. There are often more solutions available than you might initially realize, and finding the best fit for your specific needs is the key to good code quality and personal growth as a developer.
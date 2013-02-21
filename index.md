---
layout: default
title: Homepage
---

In short hexagonal.js is JavaScript's implementation of Alistair Cockburn's [Hexagonal architecture](http://alistair.cockburn.us/Hexagonal+architecture), but with some unusual solutions and additional philosophy. Let's focus on philosophy first, because it'll let you judge if you're still interested in this idea.

# hexagonal.js philosophy

1. Business logic is software's heart and have to be exposed properly.
2. Business logic is pure: uses only objects that represents domain in domain-valid state.
3. Client-side app is core of whole project, should be implemented as first.
4. Server-side API development should be driven by client-side needs.
5. Client-side and server-side are separated.
6. Both layers implements MVC.

I want to focus on client-side layer, because it is most interesting part. All snippets in this post are copied from hexagonal.js' [hello-world](https://github.com/hexagonaljs/hello-world) project.

# Structure

Architecture is build with:

1. Business logic
    1. Use cases
    2. Models
2. Glue (ports)
3. Adapters
    1. GUI
    2. Server-side
    3. WebSockets, LocalStorage etc.

## Business logic

You're probably familiar with [use case](http://martinfowler.com/bliki/UseCases.html) term. If you have some experience with DCI (or figured it out different way) you probably know, that use cases can be represented as objects - and this is core idea.

{% highlight coffeescript %}
class UseCase
  constructor: ->

  start: =>
    @askForName()

  askForName: =>

  nameProvided: (name) =>
    @greetUser(name)

  greetUser: (name) =>

  restart: =>
    @askForName()
{% endhighlight %}

Our story is quite simple: we want to greet user that uses app - ask for his name and greet him using name. As you can see it uses only plain objects and don't care about booting, GUI or storage.

## Adapters

This sample app has only one adapter, the most basic - GUI. Let's have a look at code of GUI for just first step of UseCase - askForName. ```GUI#showAskForName``` shows simple form and binds to click event of its confirm button. It has no idea about domain objects and doesn't contain any logic.

{% highlight coffeescript %}
class Gui
  constructor: ->

  createElementFor: (templateId, data) =>
    source = $(templateId).html()
    template = Handlebars.compile(source)
    html = template(data)
    element = $(html)

  showAskForName: =>
    element = @createElementFor("#ask-for-name-template")
    $(".main").append(element)
    confirmNameButton = $("#confirm-name-button")
    confirmNameButton.click( => @confirmNameButtonClicked($("#name-input").val()))
    $("#name-input").focus()
{% endhighlight %}

## Glue

You probably wonder how GUI know what to present and how can it interact with our business logic. hexagonal.js uses Glue objects to glue those two layers:

{% highlight coffeescript %}
class Glue
  constructor: (@useCase, @gui, @storage)->
    After(@useCase, "askForName", => @gui.showAskForName())
    After(@useCase, "nameProvided", => @gui.hideAskForName())
    After(@useCase, "greetUser", (name) => @gui.showGreetMessage(name))
    After(@useCase, "restart", => @gui.hideGreetMessage())
    
    After(@gui, "restartClicked", => @useCase.restart())
    After(@gui, "confirmNameButtonClicked", (name) => @useCase.nameProvided(name))
{% endhighlight %}

Ok, so this part can be hard, because your don't know what ```After``` means. It's shortcut from [YouAreDaBomb](https://github.com/gameboxed/YouAreDaBomb) library, which can be described by following code:

{% highlight coffeescript %}
After = (object, methodName, advice) ->
  originalMethod = object[methodName]
  object[methodName] = (args...) ->
    result = originalMethod.apply(object, args...)
    advice.apply(object, args...)
    result
{% endhighlight %}

So basically - it adds to original function additional behaviour. There are also ```Before``` and ```Around``` functions that let you prepend or surround original function with additional behaviour.

## Booting

To make it all run we have to implement some booting code, that'll build all required objects: domain, glue, gui and other adapters and start use case. Here's an example from hello-world app.

{% highlight coffeescript %}
class App
  constructor: ->
    useCase      = new UseCase()
    gui          = new Gui()
    glue         = new Glue(useCase, gui)

    useCase.start()

new App()
{% endhighlight %}

<aside>Source: <a href="http://blog.arkency.com/2013/02/introducing-hexagonal-dot-js/">Introducing hexagonal.js</a></aside>

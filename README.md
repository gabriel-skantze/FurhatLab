# Furhat Lab

## Preparations before the lab

We strongly advise you to use a headset instead of your laptop's internal microphone and speakers. We also recommend that you install everything and make sure it runs before you come to the lab, so that you can use the time at the lab to learn as much as possible. 

### Installation and setup

You need to install two pieces of software before you can do the lab.

   * Download and install [IntelliJ IDEA](https://www.jetbrains.com/idea/download/). Its free core Java and Kotlin functionality is sufficient.
   * Download and install the Furhat SDK. **Note**: We are using the new 3.0.0 version of the Furhat SDK here, which might not have been publicly released at the time of the lab. Follow the instructions on Canvas for how to get this.
   <!--You need to sign up ("Get SDK") to be able to download and install the Furhat SDK. Follow [these instructions](https://docs.furhat.io/launcher/) for more details. -->

Full documentation for how to use the Kotlin Skill API 3.0 [can be found here](https://docs.furhat.io/3-0-beta-nghbql/kotlin-skill-api/intro). 

### Run the SDK Launcher 

Start the Furhat SDK Launcher and click "Launch Virtual Furhat". A virtual Furhat robot should appear in its own window. Click "Open web interface" to open the web interface (or visit "localhost:8080" in a web browser). You will need a password to log in, which is "admin" by default. From there, you can test the basic gestures, speaking and listening functionality in the Testing pane. This makes sure you can hear the robot and that it can hear you.  Keep in mind that the Furhat SDK only accepts microphones and speakers that were plugged in before you started the virtual Furhat, so if you plug in your headphones after the virtual Furhat head pops up, it won’t work until you restart the Furhat SDK.

When you have gotten this to work, the "dashboard" tab is where you want to be for most of the remainder of the lab. You can double-click in the area around the Furhat robot in the middle of the page to add virtual (simulated) users (since your simulated Furhat does not have a camera). These simulated users can be moved around by dragging them, and deleted by double-clicking them. There is an inner and outer interaction zone. When the user enters the inner zone, a `UserEnter` event is raised, and when the user leaves the outer zone, a `UserLeave` event is raised. This way, your skill will be notified when an interaction should start and end. 

### Create a new skill and open in IntelliJ

In the SDK Launcher, click "Create skill" and choose the Default template. Choose where to put your skill and give it a name (avoid spaces in the name). While you can choose any name you want, we will assume the name "FurhatLab" in the examples below. 

Once you are notified that the skill is created, open IntelliJ and choose `File > New > Project from existing sources`. Select the `build.gradle` file in the skill folder that you just created. IntelliJ will now set up the project for you.

Before running the skill, you first need to set the OpenAI credentials. This can either be done by setting the `OPENAI_API_KEY` environment variable on your computer, or by creating a `.env` file in the `src/main/resources` folder in your skill with the following contents:
```
OPENAI_API_KEY=key
```

You can find a key to use on the Canvas page. 

### Run the skill

You can now run the skill from IntelliJ. If you open the folder `src/main/kotlin/furhatos.app.furhatlab`, you should see a file named `main.kt` which you can open. This is the starting point of the skill. Next to the `main()` function, there should be a green arrow. Click on it to run the skill. Then go back to the dashboard and make sure there is a virtual user within the interaction distance to Furhat, otherwise insert one. Furhat should then start the interaction with you, driven by a basic LLM.  

See what happens when you talk to the robot, and what happens when the user leaves the interaction, either by double-clicking on it or by dragging it out of the outer zone. 
 
### Understanding the code

In the `main.kt` file, you will see that the interaction flow will start with the state `Init`.
You can follow references in IntelliJ by pressing the Control/Command key and clicking on them. Do this to open the `Init` state.

When a state is entered, the `onEntry` handler is triggered. 
In the `Init` state, basic parameters for the skill are configured before proceeding (`goto`) to the state `Idle`.

In `Idle`, you will see that it checks whether any user is present, which will make the flow go to `Interaction`. Otherwise, it will wait for the `onUserEnter` to be triggered to proceed to `Interaction`. 

In the `Interaction` state, the robot will attend the closest user, start listening, and say "Hello there, I am Furhat!". The robot will not actually listen while speaking, but the `furhat.startListening()` command tells it to listen while it is silent, until `furhat.stopListening()` is called (which happens when the user leaves, as you can see). 

At the end of the user's turn, `onUserTurnEnd` will be triggered, which calls the LLM `generator` to generate a response for the robot to say. In this file, you can also see how the `generator` is defined. 

You can read more about the SDK and how to program interactions in the [Furhat docs](https://docs.furhat.io/). We will only mention some of the functionality here, but you can find much more about how to implement your ideas in the docs.  

Remember: if you change the code, you have to restart the skill for the changes to take effect. 

## Task 1: Build a robot job interviewer

Create a new skill for a robot job interviewer and give it a suitable name, such as `JobInterviewer`.

### Set up the character and system prompt

Change the `generator` definition so that the robot instead acts as a job interviewer. You can explore different voices and faces in the dashboard, and then set them programmatically in the `Init` state:

```kotlin
furhat.setVoiceByName("...") 
furhat.setFace(mask="adult", character="...")
```

Try to define a sequence of questions you want the job interviewer to go through in the `systemPrompt` and interact with it to see how it handles your answers. Add any other instructions you think are needed for the character. 

### Make the robot interruptable

By default, you cannot interrupt the robot while it is speaking, but it is possible to change that. To do so, put this line after `furhat.startListening()`:

```kotlin
furhat.interruptable = true
```

Try to see if you can now interrupt the robot while it is speaking. 

**Note**: This only works if you are using a headset, or if your laptop has effective echo cancellation, otherwise the microphone will pick up what is being played on the speaker, and the robot will interrupt itself, or interruption might simply not work. If this happens, you should continue the lab without interruption enabled. 

### Study the effects of facial expressions

In the `Init` state, all automatic gestures (`blinking`, `turnTaking`, `microexpressions`, `neck`) are turned by by default through this command:

```kotlin
furhat.enableAutoGestures()
```

Try to see what happens if you turn them off, for example the automatic turn-taking gestures:

```kotlin
furhat.enableAutoGestures(turnTaking = false)
```

Discuss with your lab partner what difference it makes, and how these gestures can potentially help to facilitate turn-taking. 

### Make the opening more varied

Currently, the opening line is hard-coded. You can use the LLM to get more varied openings, while still steering it:

```kotlin
furhat.say(generator.generate("Something like: Hi there, welcome to this job interview!"))
```

The instruction sent to the `generate()` function will then be included in the system prompt. 

### Make it possible to end the conversation with a tool

The only way to end the conversation now is for the user to leave the interaction zone. We want to make it possible to say something like "goodbye". To do this, we can use tool calling. First, we define a tool for this:

```kotlin
val endConversation = tool(
    name="EndConversation",
    description="End the conversation if the user indicates this"
)
```

Then we include this tool in the state (for example, directly before `onUserTurnEnd`, but the order doesn't matter):

```kotlin
addTool(endConversation) {
    furhat.say(generator.generate("Something like: Goodbye, it was nice talking to you")).onEnd {
        furhat.users.disengage()
        goto(Idle)
    }
}
```

Note that `furhat.say()` is not blocking, so to make sure the robot disengages only after the robot has stopped speaking (and not while it is speaking), we put these actions in the `onEnd` handler. 

Note that when the state transitions to `Idle`, `onExit` will be triggered, which makes the robot stop listening. The `onEntry` trigger in the `Idle` state will make the robot stop attending to the user. Note, however, that the `Idle` state also checks if there are any users present (which there still are), in which case it would transition to `Interaction` again and continue the conversation. To stop this from happening, we have to tell the system that the user is no longer engaged, even though they are physically present. We do this through the `furhat.users.disengage()` function above. 

We then need to tell the generator that it can use this tool by changing the way we generate the response:

```kotlin
onUserTurnEnd {
    generator.generateWithTools(this) {
        furhat.say(it)
    }
}
```

Since the `endConversation` tool was registered with the current state, and `generateWithTools` is called with a reference (`this`) to the current state, it will check if any of the registered tools are relevant for what the user said. Otherwise, it will call the supplied handler, triggering `furhat.say(it)`, where `it` refers to the generated response. 

Now run the interaction and see if you can end the conversation by saying something like "Goodbye". To start the interaction again after this, the user would have to leave and come back.

### Test a different language

To test a different language, you can use a voice of that language, for example this way in the `Init` state:

```kotlin
furhat.setVoice(language = Language.SWEDISH)
```

You also need to specify in the `systemPrompt` of the `generator` which language it is supposed to use. If you have not hard-coded what the robot says (but use a generator), its utterances should be in your chosen language. The tool call for ending the conversation should also work. 

## Task 2: Build a weather guide robot

Create a new skill for a weather guide robot and give it a suitable name, such as `WeatherGuide`.

Since the LLM in itself does not have any access to external resources, we will use tool calling to allow it to answer questions about the current weather and forecasts. For this we will use the freely available [Open-Meteo](https://open-meteo.com/) API. 

### Set up the tools

We can use `functionTool` to define a tool that works like a function that returns a result. This is different from our previous `endConversation` tool, which was used to trigger a handler within the state. We define two function tools, one for checking the current weather (`askWeatherToday`) and one for checking the forecast (`askWeatherForecast`). As you can see, each function takes an argument, the `GeoLocation`, which contains the latitude and longitude of the location. The function handler within the tool definition then specifies what the function should return, which is the last line of the function (the result of calling `checkWeather`). 

```kotlin
data class GeoLocation(val latitude: Double, val longitude: Double)

fun checkWeather(location: GeoLocation, param: String): String {
    val result = URI.create("https://api.open-meteo.com/v1/forecast?latitude=${location.latitude}&longitude=${location.longitude}&$param").toURL().readText()
    println("Result: $result")
    return result
}

val askWeatherToday = functionTool<GeoLocation>("AskWeatherToday") {
    description = "Ask about today's weather for a given location"

    function { location ->
        println("Checking today's weather for $location")
        checkWeather(location, "current=temperature_2m,wind_speed_10m,rain,showers")
    }
}

val askWeatherForecast = functionTool<GeoLocation>("AskWeatherForecast") {
    description = "Ask about a forecast for a given location"

    function { location ->
        println("Checking the forecast for $location")
        checkWeather(location, "daily=temperature_2m_max,rain_sum")
    }
}
```

Now we add these function tools within the `Interaction` state:

```kotlin
addFunctionTools(askWeatherToday, askWeatherForecast)
```

A couple of things you should also check:

* Don't forget to update the `onUserTurnEnd` handler, as we did above, for the tools to be used. 
* Don't forget to also update the `systemPrompt` in the `generator`, as well as the greeting instructions, so that they are suitable for a weather guide. 
* You might want to add in the `endConversation` tool.
* You might want to set a suitable voice and face for the character. 

### Test the interaction

Try asking the robot different types of weather-related questions, including follow-ups that might be elliptical. Here is an example of a sequence of questions that could be used:

* Ask it about the current weather in Stockholm.
* Follow up with "what about tomorrow?"
* Follow up with "and what about Paris?"
* Ask "How is the weather in London compared to Madrid right now?" (which will trigger multiple tool calls in one turn)

You can also check the printouts that were added to the functions above. What is being sent to the model based on your questions, and how is the response from the API used to generate responses? Discuss this among yourselves and report to the TA. 

## Task 3: Any idea you might have!

Now (if time permits), you are free to implement any idea of your own choice. Some examples:

* Use tool calls to change the face and voice of the robot, or its LED lights, by telling it to do so
* Define multiple states where the system prompt varies between states
* Connect to an external API using tool calls for a robot talking about a specific domain:
    * [TVmaze](https://www.tvmaze.com/) for TV shows
    * [Open Library](https://openlibrary.org/developers) for books
    * [Frankfurter](https://frankfurter.dev/) for currencies 
    * [Nobel Prize](https://www.nobelprize.org/about/developer-zone-2) for facts about Nobel Prizes


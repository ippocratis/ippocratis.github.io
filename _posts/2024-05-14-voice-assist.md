---
title: Home Assistant as an android voice assistant
description: Home Assistant "assist" as an android assistant
author: ippo
image: assets/images/voice-assist.jpeg
categories:
    - alternative
    - assistant 
tags:
    - alternative
    - assistant 
---

The home assistant Companion app on android offers many different notification options.
Instead of sending an actual notification to the device, you can send a command as a message to trigger certain actions on your phone.
This service is called "notification commands"

[link](companion.home-assistant.io/docs/notifications/notification-commands)

What does that mean?
We can use the home assistant's voice assistant as an alternative open source assistant that will perform functions on the phone itself.

What we need?

Local TTS and STT engines on the HASS server.
If your installation is full HASS OS you can install whisper (Stt) and piper (tts) as an addon.
If your installation is a docker container then you need to install whisper and piper as separate docker containers.

You can spin up a home assistant docker install following my blogpost [here](https://ippocratis.github.io/https://ippocratis.github.io/assistant/)

Documentation on installing local assist pipelines [here](https://www.home-assistant.io/voice_control/voice_remote_local_assistant/)

We will run seperate Whisper (TTS) and Piper (SST) docker containers and use them in the Haas container.

Whisper:

```
version: '3'
services:
  wyoming-whisper:
    image: rhasspy/wyoming-whisper
    ports:
      - "10300:10300"
    volumes:
      - "./data:/data"
    command: ["--model", "tiny-int8", "--language", "en"]
```

Under HASS go to

Settings>devices>add integration>whisper>ok

Host is the IP of your raspberry pi, e.g. 192.168.1.1

The port is 10300 (can assign any available port on the host).

Then in Settings>devices>Wyoming protocol>entity>fast whisper>settings>enable

And finally

Settings>voice assistant>assist>enable whisper in speech to text

Piper:
```
version: '3'
services:
  wyoming-piper:
    image: rhasspy/wyoming-piper
    ports:
      - "10200:10200"
    volumes:
      - ./data:/data
    command: ["--voice", "en_US-lessac-medium"]
```

Under HASS

Settings>devices>add integration>piper>ok

Host is the IP of your raspberry pi, e.g. 192.168.1.1

The port is 10200 (can assign any available port on the host).

Then in Settings>devices>Wyoming protocol>entity>piper>settings>enable

And finally

Settings>voice assistant>assist>enable piper in text to speech

## notification commands

To execute an activity of an android application by specifying its URI, we will use the service notify.mobile_app with the message command_activity.

We will need the divice ID of the companion app:

Settings > companion app > home > device name

- Examples:

In the example below we will see how to use the voice assistant by giving it the command to start navigation to a specific location using here maps.

Script:

Under haas UI from settings> automation>scripts>add script>crate new>edit in yaml

```
alias: Navigate
mode: single
fields:
 location:
   description: The location to navigate to
   example: Nea Smyrni
sequence:
 - service: notify.mobile_app_phone
   data_template:
     message: command_activity
     data:
       intent_package_name: com.here.app.maps
       intent_action: android.intent.action.VIEW
       intent_uri: google.navigation:q={{ location }}?startActivity;
```

package name is the name of the package of the application we will use

The action is VIEW

In uri we use google intents since they work with the application "here maps"

Android intents docume ration is [here](https://developer.android.com/guide/components/intents-common)

automation

We will create a custom sentence which will run the above script.
In addition, we will add as a variable whatever it hears after the custom sentence.

We create it from the UI by going to settings>automation>automation>add automation>crate new>edit in yaml

```
alias: Navigate
description: ""
trigger:
 - platform: conversation
   command: Go to {location}
condition: []
action:
 - service: script.navigate
   data:
     location: "{{ trigger.slots.location }}"
mode: single
```

platform: conversation

Conversation is the voice assistant platform

command: go to {location}

In the Command field we write the phrases that if heard assist will trigger the action field

"notify.mobile_app_phone:" phone is my device ID, replace it with your phone id.

'location' is the variant I use for the voice match that the voice assistant makes after the command. The following automation fields must match the {{ location }} field of the script

"fields: location:"

"command: Go to {location}"

data: location:

"{{ trigger.slots.location }}"

Finally in

"service: script.navigate"

navigate must be identical to the alias of the script

## Another example

Search and play YouTube videos

In the below example when you say "YouTube" followed by the search query YouTube will start playing the matched query  

Script:

```
alias: youtube
mode: single
fields:
  song:
    description: The YouTube search query
    example: gojira
sequence:
  - service: notify.mobile_app_phone
    data_template:
      message: command_activity
      data:
        intent_package_name: app.revanced.android.youtube
        intent_action: android.intent.action.VIEW
        intent_uri: https://www.youtube.com/results?search_query={{ song }}
```

Automation:

```
alias: youtube
description: ""
trigger:
  - platform: conversation
    command: youtube {song}
condition: []
action:
  - service: script.youtube
    data:
      song: "{{ trigger.slots.song }}"
mode: single
```

## Endnote
In essence, if we know the uri of the application's activity we can execute it and use it in whatever automation we want


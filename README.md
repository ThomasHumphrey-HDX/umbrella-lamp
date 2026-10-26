# The Rain Check Lamp

A Home Assistant automation that turns a lamp by your front door into a rain reminder. When you're about to leave, the lamp pulses blue if rain is on the way, so you know to grab your umbrella.

## Introduction

Tired of always forgetting your umbrella? Same. I can't count how many times I walked out the door, got halfway down the street and then felt the first raindrops. So I created a quick and easy system that works with Home Assistant that checks the weather for you and reminds you to take an umbrella.

Here's how it works: there is a motion sensor next to my front door. When it notices that I'm getting ready to leave, Home Assistant checks the weather forecast for the next three hours. If it's going to rain, the orange donut lamp in my hallway slowly fades to blue and then back to orange again. Pale blue means a little bit of drizzle, deep blue means you definitely want that umbrella. If it's going to stay dry, the lamp just stays the way it is and doesn't bother you.

The lamp itself is an IKEA VARMBLIXT that I took apart and filled with a smart LED strip, so it still looks like a normal (very cute) lamp hanging on the wall. Nobody would ever guess that it's secretly checking the weather.

This guide is split into four parts: setting up your devices and Home Assistant, the scripting part, building the lamp, and finally testing everything out. You don't need to know how to code, but you should be okay with copy-pasting some YAML and holding a screwdriver.

## Contents

- [Items needed](#items-needed)
- [Part 1: Setting up your devices & Home Assistant](#part-1-setting-up-your-devices--home-assistant)
- [Part 2: The scripting part](#part-2-the-scripting-part)
- [Part 3: Assembling the motion sensor and donut lamp](#part-3-assembling-the-motion-sensor-and-donut-lamp)
- [Part 4: Testing it all out](#part-4-testing-it-all-out)
- [Troubleshooting](#troubleshooting)

## Items needed

- A home server / Raspberry Pi / Home Assistant Green running Home Assistant ([Raspberry Pi installation guide](https://www.home-assistant.io/installation/raspberrypi/))
- [Home Assistant Connect ZBT-2](https://www.home-assistant.io/connect/zbt-2/) for connecting via Thread
- Smart lights that connect via Zigbee or Thread (I am using Thread in this guide, but either option works)
    - I used these smart lights: [OSRAM Matter SMART+ Flex LED strip](https://shop.ledvance.com/products/smart-matter-flex-1m-rgb-tunable-white-extension). Heads up: the link is the 1m extension piece. You also need the starter kit that comes with the controller and the power supply, because the extension on its own won't connect to anything.
- A motion sensor, such as the [Aqara FP300](https://www.amazon.de/Aqara-Pr%C3%A4senzsensor-1-Bewegungsmelder-Temperatur-Feuchtigkeitserkennung/dp/B0FG2MD3YP)
- [IKEA VARMBLIXT LED lamp](https://www.ikea.com/de/en/p/varmblixt-led-table-wall-lamp-orange-glass-round-80499199/)
- A smartphone with the Home Assistant Companion app

Tools for building the lamp:

- Small Phillips screwdriver and a flathead screwdriver
- Wire cutters
- Pliers
- Strong double-sided tape
- Nails or screws (plus wall plugs if you have to drill) for mounting the lamp

## Part 1: Setting up your devices & Home Assistant

1. Set up Home Assistant using their guide [here](https://www.home-assistant.io/installation/). If you have a Home Assistant Green, it already comes with everything installed, so you just plug it in and open `homeassistant.local:8123` in your browser.
2. Follow the instructions on setting up the ZBT-2 [here](https://support.nabucasa.com/hc/en-us/articles/31592061576733-Installing-Home-Assistant-Connect-ZBT-2).
    1. When Home Assistant asks you which protocol you want to use, pick Thread. (If your lights and sensor are Zigbee, pick Zigbee instead.)
    2. Plug the ZBT-2 in with a USB extension cable instead of directly into your server. USB 3 ports and the server itself can mess with the radio signal, and a little bit of distance helps a lot.
3. Install the Home Assistant Companion app on your phone and connect to your home network.
    1. Log in to your Home Assistant from the app.
    2. Sync your Thread credentials to your phone, otherwise your phone can't add Thread devices. On iPhone you can find this under **Settings > Companion app > Troubleshooting > Sync Thread credentials**. On Android, Home Assistant will usually ask you about this the first time you add a Thread device.
4. Now that you have Thread enabled on your Home Assistant, we can begin connecting lights that work with Matter. If you purchased the LED strip that I recommended, then follow the instructions below:
    1. Make sure that your router allows both 2.4 GHz and 5 GHz dual bands and that this setting is activated.
    2. Turn on Bluetooth on your phone and stay close to the LED strip. Matter uses Bluetooth for the first connection.
    3. Reset your LED strip by plugging it in, waiting 10 seconds, and then unplugging it. Repeat this process 10 times.
    4. You are now ready to connect the LED strip to your Home Assistant. Open the Home Assistant Companion app and open your settings. Navigate to **Devices & services** and then tap on the **Devices** tab. In the bottom right hand corner is a floating button that says **+ Add device**. Tap on **Add Matter device**, then **No. It's new.** Then follow the instructions on screen to add your smart device using the QR code found on the LED strip.
    5. If Home Assistant asks you to install the Matter Server, say yes. This is what lets Home Assistant talk to Matter devices.
    6. Continue following the on-screen instructions and name your LED strip "Donut Lamp". Assign it to your hallway (or whatever room your front door is in).
    7. Note the entity ID. It should be something like `light.hallway_donut_lamp`. You can find it by going to **Settings > Devices & services > Entities** and searching for "donut".
    8. Quick check: tap on the lamp in the app and change the color. If the strip changes color, you're good to go.
5. Now that your LED strip is added, repeat the process for the motion sensor. You can follow the official guide from Aqara [here](https://store-support.aqara.com/de-de/blogs/guide/how-to-connect-aqara-zigbee-thread-dual-protocol-devices-to-home-assistant). Make sure to note the entity ID of this device, as well as the device ID.
    1. The device ID is a little hidden. Open the motion sensor in **Settings > Devices & services > Devices** and look at the URL in your browser. The long string of letters and numbers at the end (after `/config/devices/device/`) is your device ID.
6. Now that all the hardware is set up, it's time to begin stringing it all together using automations.

## Part 2: The scripting part

1. **Weather service:** First, we need a weather service that tells us when it's going to rain. There are paid options that give us updates by the minute, but I don't want to spend money on something I only need every hour. I recommend using the Open-Meteo integration for this. It's free and you don't need an account.
    1. Before you start, check that your home location is correct under **Settings > System > General**. Open-Meteo uses this location for the forecast.
    2. Browse to your Home Assistant instance.
    3. Go to **Settings > Devices & services**.
    4. In the bottom right corner, select the **Add Integration** button.
    5. From the list, select **Open-Meteo**.
    6. Follow the instructions on screen to complete the setup.
    7. Make sure the entity ID looks like `weather.home`. If it's called something else, like `weather.forecast_home`, that's fine, just remember to change it in the script (it shows up twice).
2. **Adding the automation:** Now we can add the YAML script to our automation.
    1. Before you add the script, copy [`RainWarning.yaml`](RainWarning.yaml) into a text editor and replace the placeholders with the IDs from your own Home Assistant (see the table below).
    2. Copy the updated script, then navigate to **Settings > Automations & scenes**.
    3. Click on **Create automation**, then **Create new automation**. Tap on the three dots in the top right corner to open the advanced settings.
    4. Click on **Edit in YAML**.
    5. Delete what's already in there, paste your script and then click on **Save** in the bottom right corner.
    6. Run your script once to see if it works. Troubleshoot by checking if all of the device and entity IDs are matching the ones you have in your Home Assistant setup. Keep in mind that the lamp only does something when rain is in the forecast, so if it's sunny outside and nothing happens, that's actually correct. In Part 4 I'll show you how to test it on a dry day.

| Placeholder in the script | Replace with |
|---|---|
| `exampledeviceid` | Your motion sensor's device ID (from Part 1, step 5) |
| `exampleentityid` | Your motion sensor's entity ID as it appears in the trigger (see the tip below) |
| `light.examplelight` | Your lamp's entity ID, e.g. `light.hallway_donut_lamp` (appears three times) |
| `weather.home` | Only if your weather entity has a different name (appears twice) |

Here is the script (also available as [`RainWarning.yaml`](RainWarning.yaml)):

```yaml
alias: Rain warning at front door
description: Pulses the hallway lamp blue once when motion is detected and rain is coming
triggers:
  - type: occupied
    device_id: exampledeviceid
    entity_id: exampleentityid
    domain: binary_sensor
    trigger: device
    for:
      hours: 0
      minutes: 0
      seconds: 1
actions:
  - action: weather.get_forecasts
    target:
      entity_id: weather.home
    data:
      type: hourly
    response_variable: result
  - variables:
      next3: "{{ result['weather.home'].forecast[:3] }}"
      rain_mm: >-
        {{ next3 | map(attribute='precipitation', default=0)
           | select('number') | sum | round(2) }}
      rain_hours: >-
        {{ next3 | map(attribute='condition')
           | select('in', ['rainy', 'pouring', 'lightning-rainy'])
           | list | count }}
  - variables:
      baseline_color: [255, 167, 88]
      rain_level: >-
        {% if rain_mm >= 1.0 or rain_hours >= 2 %}heavy{% elif rain_mm >= 0.1 or rain_hours >= 1 %}light{% else %}none{% endif %}
      pulse_color: >-
        {{ {'heavy': [0, 45, 200], 'light': [130, 195, 255]}.get(rain_level) }}
  - condition: template
    value_template: "{{ rain_level != 'none' }}"
  - action: light.turn_on
    target:
      entity_id: light.examplelight
    data:
      rgb_color: "{{ baseline_color }}"
      brightness_pct: 100
      transition: 2
  - delay: "00:00:02"
  - action: light.turn_on
    target:
      entity_id: light.examplelight
    data:
      rgb_color: "{{ pulse_color }}"
      brightness_pct: 100
      transition: 3
  - delay: "00:00:04"
  - action: light.turn_on
    target:
      entity_id: light.examplelight
    data:
      rgb_color: "{{ baseline_color }}"
      brightness_pct: 100
      transition: 3
  - delay: "00:00:04"
mode: single
max_exceeded: silent
```

> **Tip:** the trigger part at the top is the fiddliest bit to get right, because the `device_id` and `entity_id` there are long random strings. The easiest way around this is to build the trigger in the visual editor first (**When > Device > your motion sensor > "Started detecting presence"**), then switch to YAML and copy those lines into the script.

So what is the script actually doing? When the sensor detects you, it grabs the hourly forecast for the next three hours and adds up how much rain is expected (in mm) and how many of those hours are rainy. Then it sorts it into three levels:

| Level | When | What the lamp does |
|---|---|---|
| Heavy rain | 1 mm or more, or at least two rainy hours | Pulses deep blue |
| Light rain | At least 0.1 mm, or one rainy hour | Pulses pale blue |
| No rain | Anything less | Nothing, the script stops right there |

The orange baseline color is `[255, 167, 88]`. It's a warm orange that goes really nicely with the orange glass, but you can change it to whatever you like. Same goes for the two blues.

## Part 3: Assembling the motion sensor and donut lamp

1. **Mounting the motion sensor:** Mount the Aqara motion sensor using the sticky pad somewhere near your entryway, preferably someplace that you will definitely pass if you are getting ready to go out of the door.
    1. Try not to point it into a room you walk through all the time, like the living room or the kitchen. Otherwise the lamp will check the weather every time you go get a snack.
    2. Before you stick it down for good, hold it in place and watch the sensor in the Home Assistant app while you walk to the door. If it switches to "Detected" at the right moment, you found your spot.
    3. If it picks you up from too far away, you can lower the detection distance or sensitivity on the sensor's device page in Home Assistant.
2. **Replacing the LED strip inside your donut lamp:**
    1. Now that your smart LED strip is configured and connected to your Home Assistant, we can begin disassembling the donut lamp and replacing the LED strip inside. **Important: make sure the donut lamp is unplugged before you start!** Also, just so you know, taking the lamp apart will void the IKEA warranty.
    2. On the backside of the IKEA VARMBLIXT lamp, there is a white button that you can press down. This opens a latch that allows you to take the back panel off of the glass. Remove the back panel and put the glass somewhere soft so it doesn't get scratched.
    3. Now, on the back panel, there are screws hidden underneath the white pads placed around the back panel. Carefully lift the white pads to reveal the screws (keep the pads, you'll want to stick them back on later). Unscrew all three screws to dismount the small LED strip circle.
    4. Now that the LED strip can be removed, we need to open the box where the switch is. This can be done by using a flathead screwdriver and carefully lifting the plastic cover off of the switch box.
    5. Carefully cut the wires to the switch box and remove the switch entirely. The power cable can be pulled through the hole and also removed, as we will be replacing it with the smart LED. Don't throw the old parts away right away, just in case you ever want to turn it back into a normal lamp.
    6. Now we will begin mounting the new smart LED strip into the donut lamp. The goal is to have both the strip and the controller/power supply inside the lamp and only have the power cord feed through. You might need to take some pliers and remove a small portion of the plastic casing to feed the power cord through. Go slow here and break off small pieces at a time, the plastic cracks easily.
    7. Use double-sided sticky tape to mount the power supply onto the back panel.
    8. Before you stick the strip down, plug it in for a second and check that it still turns on and changes color from the app. It's a lot less annoying to find a problem now than after everything is closed up. Unplug it again before you continue.
    9. Now carefully feed the LED strip around the inner donut column, wrapping it around and making sure the tape is sticking down. We don't want this to move later when we are mounting it to a wall.
    10. Now that the power supply is mounted onto the back panel and the power cord is fed carefully through the plastic casing where the switch box was, we can begin reassembling the switch box with the screws that were hidden underneath the white pads. Stick the pads back on afterwards.
    11. Now that everything is secured, we can mount the back panel onto the orange glass donut again. Do this by using the latches along the side, making sure the donut lamp is nice and snug on the latches.
    12. Now, mount the donut lamp onto the wall in your entryway using nails or screws. Pick a spot close to a power outlet and where you can see the lamp while you're putting on your shoes or grabbing your keys. That's the moment you need to notice it.

## Part 4: Testing it all out

Everything is hanging on the wall, so now it's time to make sure it actually works. The tricky part is that the lamp only reacts when rain is coming, so if you're building this on a sunny weekend, we have to trick it a little bit.

1. **Testing the motion sensor:** Open the motion sensor in the Home Assistant app and walk to your front door like you're about to leave. The status should switch to "Detected". Now walk away and wait. It should go back to "Clear" after a little while.
    1. This part is important: the automation only starts when the sensor switches from "Clear" to "Detected". The FP300 is really sensitive, so if it can still see someone standing in the hallway, it won't go back to clear and won't trigger again. Always give it a minute between tests.
2. **Testing the lamp:** Open the Donut Lamp in the app and change the color to blue, then back to orange. It should fade nicely between the colors. If it jumps from one color to the other without fading, that's okay too, some LED strips just ignore the fade.
3. **Testing the weather:** Let's check that Home Assistant is actually getting a forecast.
    1. Go to **Developer Tools > Actions**.
    2. Search for **Weather: Get forecasts** and select it.
    3. Choose `weather.home` as the entity and **Hourly** as the forecast type.
    4. Click on **Perform action**. You should now see a list of hours with a `condition` (like sunny or rainy) and `precipitation` (how much rain in mm). If this list is empty or you get an error, go back and check your Open-Meteo setup.
4. **Faking a rainy day:** Now let's pretend it's raining so we can see the lamp in action.
    1. Open your automation and go to **Edit in YAML** again.
    2. Find the line that starts with `rain_level:` and replace it and the line below it with: `rain_level: heavy`
    3. Save, walk away from the sensor for a minute, then walk to the door. The lamp should fade to orange, then slowly to deep blue, hold there for a few seconds and fade back to orange.
    4. Now change it to `rain_level: light` and try again. This time it should be a pale blue.
    5. **Super important:** put the original `rain_level` lines back in when you're done! Otherwise your lamp will tell you to take an umbrella every single day.
5. **Checking the trace:** If something isn't working, Home Assistant keeps a little log of every time the automation ran. Open your automation and click on **Traces** in the top right corner. Click on the "Define variables" step and you can see exactly what the script calculated: `rain_mm`, `rain_hours` and `rain_level`. If the trace stops at the condition, it just means the script decided it's not going to rain.
6. **The real test:** Wait for a day when rain is in the forecast, grab your keys and head to the door. Check your weather app too and see if the lamp agrees with it.

## Troubleshooting

If things still aren't working, here are the problems I ran into:

- **The lamp doesn't change at all:** Double check the light's entity ID in the script. Also check that your lamp supports RGB colors by going to **Developer Tools > States**, searching for your lamp and looking at `supported_color_modes`. It should say something like `rgb`, `hs` or `xy`.
- **The automation never runs:** Make sure the automation is turned on, and check the trigger. Wrong device or entity IDs in the trigger are the most common reason. Rebuilding the trigger in the visual editor (see the tip in Part 2) usually fixes it.
- **The blue is too short and I keep missing it:** Change the first `delay: "00:00:04"` after the blue to something like `delay: "00:00:06"` so the blue stays on a little longer.
- **The lamp stays blue:** This can happen if the automation gets interrupted, for example if you restart Home Assistant mid-pulse. Just set it back to orange in the app. The next time the automation runs, it starts from orange again anyway.

## Conclusion

And that's it! You now have a cute orange donut lamp in your hallway that quietly checks the weather for you every time you're about to leave the house. No apps to open, no notifications to ignore, just a little blue glow that says "hey, take your umbrella".

The best part is that once it's set up, you don't have to think about it at all. On dry days it's just a lamp. On rainy days it's a lamp that saves you from showing up to work soaking wet.

If you want to take it further, you could add a different color for snow, send a notification to your phone at the same time, or use the same idea for other reminders (like taking out the trash on collection day). Have fun with it, and if you build one yourself, feel free to open an issue or share a photo!

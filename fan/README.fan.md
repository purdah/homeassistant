This is a how-to on setting up an RF controlled fan inside Home Assistant using the RM4 Pro device to send RF singals to the fan.

The basic process is:

1. Learn Commands from the RF remote into Home Assistant.
2. Create scripts to run those commands
3. Configure the fan template to build a fan entity that can be controlled inside HA.

## Learning Remote commands

Reference docs:

https://www.home-assistant.io/integrations/broadlink/

https://www.home-assistant.io/integrations/broadlink/#remote

Each button on the remote that you want to use within HA will need to be learnt and stored within the Broadlink storage file.

You will need to think ahead about names of the fan and the buttons.

Good names for a fan would be "Office Fan" or "Living Room Fan", and buttons could be "Low", "Medium", "High" or maybe speeds such as "Speed1", "Speed2", etc

When choosing names such as "Office Fan" these are the human-readable names, often you will need to use an underscore character "_" between the human-readable words
in order to create some device IDs used internally within HA.

For my example I have a remote with buttons labelled Off, 1, 2, 3, 4, 5 and 6.
As I dont use all 6 speeds I will settle on using 3 out of those speeds and calling speed 1 as Low, speed 2 as medium and speed 3 as high.
Feel free to choose whatever naming system you want for the fans.
You do not even need to have the same speeds recorded for each fan, so the office fan may only need the first 3 speeds 
but in a larger room such as the living room you may want to use speeds 2, 4 and 6 as low, medium and high.
The choice is yours but you need to record the values you want to use for each fan



The documentation is light on *how* to run a script that learns the button presses.

As of 12/2024:
From HA home page go to: 
1. Developer Tools -> Actions
2. In the action field enter: "Remote: Learn command"
3. In the command field enter the name of the button, eg "Speed1"
4. Tick 'Device' and enter the device_id of the fan you want to use eg office_fan
5. Select 'rf' as the command type
6. click 'Perform Action' button and be ready to press the button on the remote
7. wait for the red light on the RM4 to come on and hold the button until the light goes off (about 10 secs)
8. The light will come on again and repeat the same button for 10 secs until the light goes off

The button is now saved in the .storage file for broadlink codes.

You can test the button using the Remote: Send command action entering the same DeviceId and Command as used when learning the commands.

Some buttons have a long press feature, these can be identified by the fact that using the above process then running Send Command will not trigger the correct state in the fan.
Simply click the Alternative checkbox and repeat the process above except that the light on the RM4 will come on 4 times rather than 2, but just hold down the button as before until the light
goes out after about 10 seconds.

When you have recorded all the buttons you are interested in using you can proceed to the next step.

## Create scripts to Send the remote commands.

At the time of writing using the remote send command directly from the fan template did not work.
This was probably due to a syntax error that I could not work out, however, having the extra actions inside
the same fan template has some drawbacks that makes the template harder to read and maintain.

By using a single script to trigger the remote command has a few advantages in that variables can be used to keep the 
script quite generic and usable across multiple fans that use RF or IR remotes.

Start by going to Settings -> Automations & Scenes -> scripts

1. Click the blue Add Script button at the bottom of the page
2. Choose 'Create new script'
3. Click Add Action
4. Click Choose Device and select 'rm4pro'
5. Enter the text: Remote: Send Command
6. Click the device tickbox, and fill out the details with the deviceId
7. Enter any command in the command box, you will be replacing it in the next steps
8. Click the blue save script button
9. In the name field use an easy to remember script name that uses a generic name for the command, eg: fan_control (reccommended as that is what is used below)
10. Add an icon if you want and a description if you need a prompt for the purpose of the script
11. Click rename
12. Click the 3 vertical dots next to the sequence named "Remote 'Send command' on rm4pro" and select 'edit in yaml'

You should be presented with an editor with yaml code in it similar to below. 

First copy the original device_id value at the bottom of the existing script and then paste the script below into the editor.
Replace the text <<deviceId>> with the value you copied before you replaced the script with the new content.

```yaml
action: remote.send_command
data:
  num_repeats: 1
  delay_secs: 0.4
  hold_secs: 0
  device: "{{ fan_name }}"
  command: >
    {% if percentage <= 33 %}
      speed1
    {% elif percentage > 33 and percentage <= 66 %}
      speed2
    {% elif percentage > 66 %}
      speed3
    {% endif %}
target:
  device_id: <<deviceId>>
```

What you have done is create a generic script for your fan that you can pass in a fan_name such as 'office_fan' and a percentage
value such as '50' and this will send one of the commands to the fan.

You will make use of this single script in the next section.

## Configure the fan template

From now on you will need to be able to edit the configuration.yaml file in your home assistant config folder, if you are 
unsure on how to do that then it would be recommended to not proceed. There are instructions on how to edit your configuration on 
the home assistant site and familiarize yourself with how to edit your config as it is assumed you know how to edit your 
config from now on.

edit your configuration.yaml file.

Add the following to your config, read below for a full explanation:

```yaml
# If the speed_count = 3 then the values are: 0, 33, 66, 100
# Note that the graphics for speed 3 are different to speed 6 in that there are 4 clear sections to click on
# whereas 6 has a smooth slider.
input_boolean:
  my_fan_state:
    name: My Fan State

input_number:
  my_fan_percentage:
    name: Office Fan Percentage Value
    initial: 0
    min: 0
    max: 100
    step: 1

fan:
  - platform: template
    fans:
      my_fan:
        friendly_name: "My Fan"
        value_template: "{{ states('input_boolean.my_fan_state') }}"
        turn_on:
          - action: input_boolean.turn_on
            target:
              entity_id: input_boolean.my_fan_state
          - action: script.fan_control
            data:
              fan_name: "my_fan"
              percentage: 50
        turn_off:
          - action: input_boolean.turn_off
            target:
              entity_id: input_boolean.my_fan_state
          - action: script.fan_control
            data:
              fan_name: "my_fan"
              percentage: 0
        percentage_template: >
          {{ states('input_number.my_fan_percentage') if is_state('input_boolean.my_fan_state', 'on') else 0 }}
        speed_count: 3
        set_percentage:
          - action: input_boolean.turn_{{ 'on' if percentage > 0 else 'off' }}
            target:
              entity_id: input_boolean.my_fan_state
          - action: input_number.set_value
            target:
              entity_id: input_number.my_fan_percentage
            data:
              value: "{{ percentage }}"
          - action: script.fan_control
            data:
              fan_name: "my_fan"
              percentage: "{{ percentage }}"
```

If you chose a different name for the fan script created in the previous section then replace "fan_control" with the name of the script you used.
Replace all occurrences of "my_fan" with a name you want for your particular fan, eg office_fan. Replace the  friendly_name of the fan from "My Fan"
to something fiendly such as "Office Fan"

After editing the file, check and then reload the config 

Go to: Settings -> Devices & Services -> Entities

You should find a Fan entity with a name that you entered into the friendly_name field.

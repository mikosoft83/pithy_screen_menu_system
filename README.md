# Pithy Screen Menu System for controlling Home Assistant

**Created by: Milan Korenica**

This is an ESPHome YAML script that implements a menu system to show state of and control several entities from Home Assistant on a Pithy Screen device.

Pithy Screen is based on D1 mini, has a rotary encoder with push, a switch and SSD1306 screen over I2C. See more on [ioios website](https://ioios.io) and [their Github repo](https://github.com/ioios-io).

Rotary encoder is used to browse the menu, push on the rotary is used to either enter submenu or enter setting. Side switch is used to return from setting or return to previous menu level.

## Setup

There are two ways to setup the menu
1. Using config generator
2. Manual

### Using config generator

Config generator is in the [menu_generator](menu_generator) folder. The best way to use it is to clone/download the files to your computer and running from there.

The menu generator doesn't do any sanity checks. You are in charge of filling in all the data correctly. Every field of every item must be filled in, the only exception is the "Attribute" field in case of menu items that won't use it. If you manage to make a mistake you will only find out during compile or operation so be careful. Checks may be added in future versions.

To set up the menu structure use the Menu section. You can move the menu items around by dragging them by the large title. You can also drag items out of submenu to another menu level or submenu. __Remember there needs to be at least one item in each submenu!__ Failing that will cause crashes when running on the ESP.

For each item you can use the preconfigured action on encoder change (the dropdown labeled "What to do on change") or you can type in your own code. The fields will expand when you click "Advanced settings". If you want to use one of the preconfigured actions but you want to modify it, select the desired action in the dropdown and then select "Custom service call".

If you need help with the meaning of each setting hover your mouse over each label. You'll get a tooltip with some (hopefully) useful information. Also please refer to chapters 2.1 and 2.3 for definiton of menu functions and an explainer on the definiton of range for continuous value sensors.

Once you're happy with your menu click __Create config__. Two text fields will be populated, one with ESPHome YAML config and the other with Home Assistant template sensor config.

You can download the YAML file with the __Download config file__ button. Compile and upload the YAML file using ESPHome and your preferred method.

If any template sensors for HA have been generated, paste that config into the respective config file in Home Assistant. This will be dependent on the structure of your yaml files and sensor configuration. The default is configuration.yaml and ``sensor:`` section like this:

    sensor:
    ... your sensors..
      - platform: template
        sensors:
          pithy_....
            value_template: ...

### Manual configuration

Setup of menu structure and functions is done in several places:
1. Setup of menu structure and data
2. Setup of actual menu functions
3. Setup of template entities in Home Assistant
4. Import of entities from Home Assistant

Each part of configuration is marked by comments like these:

    #####
    #####  CONFIGURATION BLOCK HERE
    #####
    
    #####  END OF CONFIGURATION BLOCK 

First block marks substitutions, where you need to specify board, name and respective pins for each function and data for menu system.
Second block defines the menu structure and data and actions for each menu item
Third and fourth blocks define sensors and binary sensors imported from Home Assistant.
Let's look at each part in detail now.

### 1. Setup of menu structure and data

The structure of the menu is stored in one dimensional array. Since menu is a tree, we need to be able to define how menu items are interconnected. For example, if we have menu structure like this:

* Parent 1
  * Submenu 1-1
    *  Submenu item 1-1-1
    *  Submenu item 1-1-2
  * Submenu 1-2
    * Submenu item 1-2-1
    * Submenu item 1-2-2
* Parent 2
  * Submenu 2-1
    * Submenu item 2-1-1
    * Submenu item 2-1-2
  * Submenu 2-2
    * Submenu item 2-2-1
    * Submenu item 2-2-2

The structure would be stored in an array like this:

    Array index | Menu item
    0           | Parent 1
    1           | Parent 2
    2           | Submenu 1-1
    3           | Submenu 1-2
    4           | Submenu item 1-1-1
    5           | Submenu item 1-1-2
    6           | Submenu item 1-2-1
    7           | Submenu item 1-2-2
    8           | Submenu 2-1
    9           | Submenu 2-2
    10          | Submenu item 2-1-1
    11          | Submenu item 2-1-2
    12          | Submenu item 2-2-1
    13          | Submenu item 2-2-2

Array index always starts with 0.

But now we can see that if we go in order of the array we will actually not traverse the menu correctly. We need to define where each menu level starts and ends and how to go to submenu and return back. For this purpose we define a child for each submenu item and number of items in each submenu. So in our example, it would look like this:

    Array index | Menu item           | Child item | Number of items in submenu
    0           | Parent 1            | 2          | 2
    1           | Parent 2            | 8          | 2
    2           | Submenu 1-1         | 4          | 2
    3           | Submenu 1-2         | 6          | 2
    4           | Submenu item 1-1-1  | none       | none
    5           | Submenu item 1-1-2  | none       | none
    6           | Submenu item 1-2-1  | none       | none
    7           | Submenu item 1-2-2  | none       | none
    8           | Submenu 2-1         | 10         | 2
    9           | Submenu 2-2         | 12         | 2
    10          | Submenu item 2-1-1  | none       | none
    11          | Submenu item 2-1-2  | none       | none
    12          | Submenu item 2-2-1  | none       | none
    13          | Submenu item 2-2-2  | none       | none

And this is fundamentally how the menu is constructed.

The menu structure is stored in four global arrays:
* `menu_labels` - these are the texts that are displayed for each menu item
* `menu_functions` - these define what each menu item does (will be explained in next chapter)
* `menu_child` - for items, that are submenus, these point to the array index, where the first item of the submenu resides
* `menu_length` - for submenus, these tell how many items are there in the submenu, for settings, these define the range of values (explained in chapter 3)

And in two substitutions:
* `menuDepth` - this defines how many levels the menu has including the top level
* `menuSize` - this defines how many items there are (this is pretty much the last array index + 1)

So our example menu would look like this:

    substitutions:
      menuDepth: '3'
      menuSize: '14'

    globals:
       - id: menu_labels
         type: char * [${menuSize}]
         initial_value: 'menu_labels = { "Parent 1", "Parent 2", "Submenu 1-1", ... , "Submenu item 2-2-2" }'

And you can fill in the rest.

In this code the menu item with index 0 is always screensaver, so you will actually need to account for that. See the example sctructure in the second configuration block in the YAML file.

Also, the arrays in the example are formatted for readability like this:

         initial_value: '
           {
             "Saver", "Menu 1", "Menu 2",
               "1st submenu 1", "1st submenu 2",
                 "Continuous", "Binary", "Action button",
                 "Cont+confirm", "Display",
               "2nd submenu 1", "2nd submenu 2"
           }'

If you do this yourself, _don't forget comma_ after each element **but** the last one and don't forget that all array elements _need to be enclosed in curly braces_. 

### 2. Setup of menu functions

#### 2.1 Defining the functions

So now that we have the structure of our menu, we want to define what each menu item does. There are several functions available that we can store in `menu_functions`:

* 0: **Screensaver** (only valid for menu item 0)
* 1: **Display value** - only show a value of a sensor, do nothing
* 2: **Submenu** - if you press the encoder on this menu item, you will enter the submenu. Pressing the side button returns one level up.
* 3: **Continuous setting** - set sensor that has a continuous range of values (such as thermostat, volume, blinds position etc.). Pressing the encoder will enter set mode and every turn of encoder will immediately change the value of the sensor. Pressing the encoder or side button will exist the set mode. This is best used with devices that can handle rapid value changes well, e.g. volume settings on a media player, color of a bulb, etc. _Not recommended for example for relay controlled blinds since the relay will be working too hard_.
* 4: **Toggle** - change value of a binary sensor. Same as above but only has two states, ON or OFF.
* 5: **Action button** - if you press the encoder on this menu item an action will be called. Can be any homeassistant service, script or automation. It's also possible to pass data (see [ESPHome docs on how to call HA services](https://esphome.io/components/api.html#homeassistant-service-action) )
* 6: **Continuous setting with confirmation** - same as continuous setting (number 3) but the value won't be changed on every encoder turn. To set the value encoder needs to be pressed again. Pushing the side button will exit set mode without setting the value.

See the example in YAML file to see this in action.

#### 2.2 Configuring the function calls

Now that we have the functions stored we need to make sure ESPHome does something. For this there are three scripts defined that are called on appropriate places within the code. These three scripts are:
* `menu_values`
    Contains only single lambda action that contains a single `switch()` statement. For each menu item that displays or sets a value of a sensor ensures the correct value is displayed. This value is assigned to the global variable `id(menu_current_value)`. Each case is the array index of the menu item for which we want to display the value.
    Here are premade `case` blocks for each menu function. Replace the # symbol with the array index of your menu item and *** with the name of your internal ESPhome sensor (explained in chapter 4):
  * For sensors with continuous values, use this `case` code:

                       case #: id(menu_current_value) = id(***).state; break;

  * For binary sensors, the value needs to be converted to 0 or 1 like this:

                       case #: id(menu_current_value) = id(***).state ? 1 : 0; break;

  * You also need to use one of the above for menu function "1: Display" depending on whether you're trying to display a binary or continuous sensor.

  * For item with continuous value with confirmation we need to account for set mode, so use this code:

                       case #: id(menu_current_value) = id(menu_set_mode) ? #start + id(rotary_dial).state*#step : id(***).state; break;

  Note that there is a "#start" and "#step", these need to be replaced by actual values. These will be explained in chapter 2.3.

  **You need to put in as many of these `case` code blocks as many menu items displaying sensor values you have.** If you forget one or make a mistake you will see errorneous values displayed.
	
 * `menu_set_rotary`
    This is similar to the above script. Also contains only single lambda with `switch()` statement. This script sets the encoder to the value of the sensor when the encoder is pressed. As with previous script here are readymade `case` code blocks that you can use:
	 * Continuous value sensors (use this for continuous with confirmation as well):

                       case #: id(rotary_dial).set_value((id(***).state - #start)/#step); break;
	  again, note the #step and #start.

	 * Binary sensors:
  
                       case #: id(rotary_dial).set_value(id(***).state ? 1 : 0); break;

 * `menu_actions`
    This is where the real magic happens. In this script you configure the Home Assistant calls for each menu item. Note these are not lambda calls since ESPHome doesn't support HA calls from lambda. So each menu item has its own `if` action with one lambda condition where you put your array index.
    Again, here are premade `if` actions for you to use:
	 * Continuous sensor (also for continuous with confirmation):
  
                 - if:
                     condition:
                       lambda: 'return id(menu_current_node) == #;'
                     then:
                       homeassistant.service:
                         variables:
                           x: 'return #start + id(rotary_dial).state*#step;'
                         service: #service
                         data_template:
                           entity_id: #entity
                           value: '{{ x }}'

      Configuration of the `data_template` key will depend on which service you call. However to pass the value to the sensor always use `'{{ x }}'` (do not forget the single quotes). See YAML for examples with real HA services.
	  
	 * Binary sensor:
  
             - if:
                 condition:
                   lambda: 'return id(menu_current_node) == #;'
                 then:
                   if:
                     condition:
                       lambda: 'return id(rotary_dial).state;'
                     then:  
                       homeassistant.service:
                         service: switch.turn_on
                         data:
                           entity_id: #entity
                     else:
                       homeassistant.service:
                         service: switch.turn_off
                         data:
                           entity_id: #entity

      Here's an example with switch. If you need to triger something else just replace the service calls in the appropriate places (`then` block for "ON" state, `else` block for "OFF" state).
	    **Note**: It's not a good idea to use `switch.toggle` or similar services that toggle a state in this block. The behavior will seem erratic. Use "5: Action button" for that.
	
	* Action button:
	  We didn't define anything for action button in previous scripts because the Action button menu function doesn't call those. But it will call this script so it needs to be here:
    
             - if:
                 condition:
                   lambda: 'return id(menu_current_node) == #;'
                 then:
                   homeassistant.service:
                     service: script.example_script
                     data:
                       example: data

      Just a simple HA service call. 
	  
#### 2.3 Mapping the continuous sensor values

The internal rotary encoder range always starts at 0 but some sensors and service calls will require values from range that start on non-zero value. For example, a thermostat setting may be in a range of 16 to 25 degree Celsius. Furthermore, the encoder always increments by 1 but the value step may be different, in our thermostat example, the step may be 0.5 degree.

So to properly map the encoder to the sensor value we need to calculate the range for each sensor/service call first:
`range = (stop - start) / step`
  where start and stop are the first and last value in the sensor range and step is the minimum step (difference between two closest values)
  In our temperature example the range would be `(25 - 16)/0.5 = 18`

This `range` value is the one you put into `menu_length` array for each menu item that uses continuous sensor (so both continuous and continuous with confirmation). If you forget to put in the correct value the settings won't cover the whole range of the sensor.

The start and step values then need to be used in the scripts above in place of `#start` and `#step` placeholders.

Please keep in mind this calculation is valid for sensors/service calls with linear range. If the sensor/service uses a different maping (logarithmic, exponential or anything else) you'll need to figure out the correct mapping and code yourself.

### 3. Setup of template entities in Home Assistant

Since ESPHome doesn't support importing entity attributes from Home Assistant entities you need to configure template entities in HA for each attribute that you want to display or control using this menu system. Note that if the value that you want to show/control is in the state of the entity, you don't need to configure a template entity for that, you can import it directly to ESPHome (see next chapter on how to do that).

So in your Home Assistant _configuration.yaml_ (or relevant yaml file if you use split config) put this code for each entity attribute you want to show/control:

    sensor:
      - platform: template
        sensors:
          sensor_1:
            value_template: "{{ state_attr('some.entity', 'attribute') }}"
          sensor_2:
            value_template: "{{ state_attr('other.entity', 'different_attribute') }}"
		
This will create two new sensors named `sensor.sensor_1` and `sensor.sensor_2`. These are now ready to be imported to ESPHome.

### 4. Import of entities from Home Assistant

To use a value of HA entity in ESPHome you need to import it to an internal ESPHome sensor. Here's an example:

    sensor:
      - platform: homeassistant
        name: "Example sensor"
        entity_id: sensor.example_sensor
        id: ex_cont
        internal: true
        on_value:
          then:
            # Logic to correctly update menu values
            - script.execute: menu_values
            - if:
                condition:
                  lambda: 'return id(menu_set_mode);'
                then:
                  - script.execute: menu_set_rotary
            # End of menu values logic

`entity_id` contains the name of the HA sensor you want to import. `id` contains a name that you can pick and it will be internal to ESPHome. You need to use this sensor `id` in the three scripts configured in chapter 2.2 in place of the `***` placeholder.

Note that the sensor has `internal: true` configured, so that it doesn't show in Home Assistant. Also the `on_value:` block needs to contain the two `script.execute` and `if` calls to properly update values in the menu system when the state of the sensor changes. If you accidentaly delete the `on_value:` block the menu system may display or set wrong values when used.

The sensors can be imported in the third (for continuous sensors) and fourth (for binary sensors) configuration block (see the introductory chapter about the configuration blocks).

## Final remarks

It's probably a good idea to first prepare the whole menu on a piece of paper together with array indices, menu functions, ranges, associated sensors etc. Once you start putting it into code it's quite difficult to make changes. If you need to insert a menu item somewhere in the middle of the array all the indices that follow will shift and you need to rewrite all the indices in the three srevice calls from chapter 2.2 accordingly.

That is it, if you reached this far you should have successfully configured the menu system for use with your chosen sensors. Have fun!

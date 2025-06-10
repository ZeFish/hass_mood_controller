
# Home Assistant Mood Controller

Mood controller is a modular Home Assistant scripting system that allows centralized control of lighting across rooms using concepts like moods, presets, and transitions. It supports asynchronous operation, mood-based scenes, and fine-grained room-level logic.

## The Problem: A Missing Rhythm

Home Assistant is incredibly powerful for connecting and automating devices. However, it lacks a built-in "rhythm" or stateful scene management system. The challenge arises when you want to temporarily override a room's lighting (for instance, to watch a movie) and then have it revert not to its _previous_ state, but to the _current_ home-wide ambient state, which may have transitioned (e.g., from "Evening" to "Night") in the interim.

This system solves that by establishing a clear hierarchy of moods and presets, ensuring that your home's lighting and atmosphere are always in sync with your life's rhythm.

## The Core: Moods and Presets
At its heart, Mood controller embodies a _state-centric hierarchical automation_ philosophy, where moods cascade from the house level down to individual rooms, yet each space retains the autonomy to maintain its own state when needed. The system uses minimal helpers that serve dual purposes as both configuration points and memory, ensuring your home can gracefully recover from any disruption.

### Moods

These are high-level, home-wide scenes that define the general ambiance. They are the primary states of the home. You can create as many for any needs that you might have. My personal implementation uses five principal moods tied to the time of day.

- **Morning**: Calm and easy light to start the day
- **Day**: Working lighting for daytime activities
- **Evening**: Warm, comfortable lighting for early evening and cooking
- **Unwind**: Softer lighting for relaxation and bedtime reading
- **Night**: Dim, gentle lighting for nighttime

### Presets

These are variations or modifications of a Mood, often used for manual control within a room without changing the underlying Mood. This allows for manual alternative within the current mood without breaking the overall rhythm. My personal standard presets are:

- **default**: The intended lighting scene for the mood.
- **bright**: A brighter version of the current mood's scene.
- **off**: Turns the lights off in the specified area.

This two-tiered system allows for both powerful, home-wide automation and intuitive, manual control via physical switches or dashboards.

## Usage Examples

Here are some examples of how to call the `mood_set` script in your automations or dashboard actions.

```yaml
# Change the mood of one or multiple areas
- service: script.mood_set
  data:
    target_areas:
      - living_room
      - kitchen
    mood: unwind
```

```yaml
# Sync all rooms to the current home mood.
- service: script.mood_set
  data:
    mood: "{{ states('input_text.home_mood') }}"
    transition_time: 30
```

```yaml
# Sync a specific room to the home mood.
# Useful for after an override,
# like when a movie finishes.
- service: script.mood_set
  data:
    target_areas:
      - living_room
    mood: "{{ states('input_text.home_mood') }}"
    transition_time: 5
```

```yaml
# Refresh all rooms to their current state.
# This reverts any manual light changes
# that were made outside of the mood system.
- service: script.mood_set
```

```yaml
# Refresh a single room.
- service: script.mood_set
  data:
    target_areas:
      - kitchen
```

```yaml
# Set specific rooms to a new mood and preset.
- service: script.mood_set
  data:
    target_areas:
      - bedroom
      - children
    mood: night
    preset: default
```

```yaml
# Toggle a preset for a physical switch.
# This call sets the kitchen to bright.
# If it's already bright, it toggles
# to the default preset.
- service: script.mood_set
  data:
    target_areas:
      - kitchen
    toggle: true
    preset: bright
```

# How It Works: The Setup (4 steps)

## 1. The helpers : `input_text.home_mood`

The system relies on a few key components: Home Assistant *helpers* for state storage, a control script, and individual scripts for each mood.

State is stored using `input_text` helpers. *This makes the current mood and preset of any area instantly accessible to other automations.*

|                              |                 |           |                                                                                               |
| ---------------------------- | --------------- | --------- | --------------------------------------------------------------------------------------------- |
| **Helper**                   | **Type**        | **Scope** | **Description**                                                                               |
| `input_text.home_mood`       | `input_text`    | Global    | Stores the current active Mood for the entire home.                                           |
| `input_text.area_id_mood`    | `input_text`    | Per Area  | (Optional) Stores the current Mood for a specific area (e.g., `input_text.kitchen_mood`).     |
| `input_text.area_id_preset`  | `input_text`    | Per Area  | (Optional) Stores the current Preset for a specific area (e.g., `input_text.kitchen_preset`). |
| `input_boolean.area_id_lock` | `input_boolean` | Per Area  | (Optional) A lock to prevent an area from being changed by home-wide Mood calls.              |


> [!TIP]
> You can still control areas that do not have dedicated `input_text` helpers.
> If you call `mood_set` with no `target_areas`, it will target all areas in your Home Assistant instance. For areas without helpers, it will apply the requested mood using the `default` preset.
> This is a great way to have secondary areas (like hallways or closets) follow the main home rhythm without needing their own complex state management.

## 2. The Controller : `script.mood_set`
This is the central nervous system of the entire setup. It is responsible for interpreting requests, determining which areas to change, and dispatching commands to the appropriate mood scripts.

```yaml
- target_areas: The area(s) to apply the mood to. If left empty, applies to all unlocked areas.
- mood: the desired lighting mood (e.g., "morning", "off", "cozy").
- preset: optional override for brightness/configuration level (e.g., "bright", "dim").
- toggle: If true, toggles between the specified preset and 'default' when an area is already in the target preset state.
- transition_time: number of seconds for light transition (default: 2).
```

> [!NOTE]
> **The "Smoking Gun" : Dynamic Script Execution**
> The core of this system's flexibility comes from using Jinja2 templates to dynamically call scripts. A single action can trigger any mood script based on a variable, allowing for infinite scalability.
> ```yaml
> - variables:
>     mood_script: "script.mood_{{ states('input_text.home_mood') }}"
> - service: "{{ mood_script }}"
>   data:
>     target_areas: "{{ areas }}"
>     preset: "{{ preset }}"
>     transition_time: "{{ transition_time }}"
>   alias: Call the mood script
>   continue_on_error: true
> ```
> This makes the system infinitely extensible. To add a new mood, you simply create a new `script.mood_[newmood]` and the system will handle it automatically.

## 3. The Automation : `automation.home_mood_change`
When the `input_text.home_mood` helper changes, it calls `script.mood_set` to propagate that change to all unlocked rooms. The beauty of it is that if you use any "Do not disturb" or "Away" mode, you can disable the automation so that it does not trigger. Altho, you can keep updating the `input_text.house_mood` to keep the moods flowing. When you turn on the automation, it will automatically propagate the mood to all unlocked rooms and thus, keep your home just like you want it.

```yaml
alias: Home mood change
description: It call script.mood_set when input_text.home_mood change
triggers:
  - entity_id:
      - input_text.home_mood
    to: null
    trigger: state
  - trigger: state
    entity_id:
      - automation.home_mood_change
    from: "off"
    to: "on"
conditions: []
actions:
  - action: script.mood_set
    metadata: {}
    data:
      mood: "{{ states('input_text.home_mood') }}"
      transition_time: 30
mode: restart
```

> [!IMPORTANT]
> Make sure that this automation is the one that mood_set disable and re-enable when the script need to update `input_text.house_mood` it is important or you might get an infinite loop.

## 4. The Mood : `script.mood_{mood_name}`

### The Individual Mood Script Architecture

Each mood requires its own script (e.g., `script.mood_morning`). This script defines the specific scenes for each preset within that mood.

The script is structured into three main sections:

1. **Preset-Agnostic Actions**: A parallel block where you can define actions that should run for a room whenever this mood is set, regardless of the preset. This is ideal for setting things like speaker volume.
2. **Preset Logic**: A `choose` block that routes to the requested preset (`default`, `bright`, `off`, etc.) and applies the specific light settings for that preset in each targeted room.
3. **Home Wide Actions**: A final section that runs only if the mood was applied to the entire home. This is useful for setting global variables, like a home-wide music playlist.

```yaml
alias: Morning
description: ""
icon: mdi:weather-sunset-up
sequence:
  - alias: Preset Agnostic modifications
    parallel:
      - alias: Kitchen
        if:
          - alias: If area is requested
            condition: template
            value_template: "{{ 'kitchen' in target_areas }}"
        then:
          {# actions heres like speaker change in the kitchen #}
      - alias: Living Room
        if:
          - alias: If area is requested
            condition: template
            value_template: "{{ 'living_room' in target_areas }}"
        then:
          {# actions heres like speaker change in the living room #}
  - alias: Preset Logic
    choose:
      - conditions:
        alias: Default
          - condition: template
            value_template: "{{ preset == 'default' }}"
        sequence:
          - alias: Run thru the rooms
            parallel:
              - alias: Kitchen
                if:
                  - alias: If area is requested
                    condition: template
                    value_template: "{{ 'kitchen' in target_areas }}"
                then:
                  {# any change you want in that mood/preset/areas combination #}
              - alias: Living Room
                if:
                  - alias: If area is requested
                    condition: template
                    value_template: "{{ 'living_room' in target_areas }}"
                then:
                  {# any change you want in that mood/preset/areas combination #}

      - conditions:
        alias: Bright
          - condition: template
            value_template: "{{ preset == 'bright' }}"
        sequence:
          - repeat:
              sequence:
                - target:
                    area_id: "{{ repeat.item }}"
                  data:
                    brightness_pct: 55
                    color_temp_kelvin: 2700
                    transition: "{{ transition_time }}"
                  action: light.turn_on
              for_each: "{{ target_areas }}"

      - conditions:
        alias: "Off"
          - condition: template
            value_template: "{{ preset == 'off' }}"
        sequence:
          - repeat:
              sequence:
                - target:
                    area_id: "{{ repeat.item }}"
                  data:
                    transition: "{{ transition_time }}"
                  action: light.turn_off
              for_each: "{{ target_areas }}"
    default:
      - action: persistent_notification.create
        metadata: {}
        data:
          message: |Preset not found
            Mood : Morning
            Preset : {{ preset }}
          title: Mood Set
          notification_id: mood_script_preset_not_found_morning_{{ preset }}

  - alias: If for the whole home
    condition: template
    value_template: "{{ is_homewide | default(true) }}"
  - action: input_text.set_value
    metadata: {}
    data:
      value: "{{ states('input_text.playlist_morning') }}"
    target:
      entity_id: input_text.playlist_home
    alias: Define home playlist
mode: parallel
max: 10
```


## Advanced Features

### Area Locking

The system includes a locking mechanism to prevent mood changes in sensitive areas. If an `input_boolean.[area_id]_lock` is `on`, any `mood_set` call that targets a group of areas will automatically exclude the locked area.

> [!TIP]
> A call made _specifically_ to that single locked area will still go through. This is the desired behavior for in-room controls, allowing someone in a locked room to still adjust their own lights.

### Performance Optimization

To ensure fast execution, `script.mood_set` intelligently groups areas by their required mood and preset combination. If 4 rooms need `Morning/default` and 1 room needs `Morning/off`, the script doesn't make 5 individual calls. It makes two optimized calls:

```yaml
- service: script.mood_morning
  data:
    target_areas:
      - room1
      - room2
      - room3
      - room4
    preset: default
    
- service: script.mood_morning
  data:
    target_areas:
      - room5
    preset: off
```

**This significantly reduces overhead and execution time.**

### Event Hook: mood_setted

At the end of its execution, `script.mood_set` fires a custom event named `mood_setted`. This event includes all relevant parameters—such as the targeted areas, mood, preset, and whether the change is home-wide—allowing other automations to react dynamically.

```yaml
- event: mood_setted
  event_data:
    target_areas: "{{ areas_resolved_unlocked }}"
    mood: "{{ mood_resovled }}"
    preset: "{{ preset_resolved }}"
    is_homewide: "{{ is_homewide }}"
```

> [!TIP]
> I have a script that on birthdays or special occasions found in a calendar, it will call a script bound to that occasions and thus overwrite some lights after the moods have been setted.

## Automation Ideas
The true power of this system is unlocked through automation. Because the state of every room is always known and stored in helpers, you can create highly specific and intelligent automations.

#### Motion-Based Night Light

This automation turns on a dim light in the kitchen upon motion, but _only_ if the kitchen is in the `Night` mood and `default` preset. When motion clears, it gracefully returns the room to the standard `Night` scene.


```yaml
alias: Kitchen - Night - Motion
trigger:
  - platform: state
    entity_id: binary_sensor.kitchen_motion_occupancy
    from: "off"
    to: "on"
    id: "Detected"
  - platform: state
    entity_id: binary_sensor.kitchen_motion_occupancy
    from: "on"
    to: "off"
    for: "00:02:00"
    id: "Cleared"
condition:
  - condition: state
    entity_id: input_text.kitchen_mood
    state: "Night"

action:
  - choose:
      - conditions:
          - condition: trigger
            id: "Detected"
          - condition: state
            entity_id: input_text.kitchen_preset
            state: "default"
        sequence:
          - service: script.mood_set
            data:
              target_areas:
                - kitchen
              preset: motion
      - conditions:
          - condition: trigger
            id: "Cleared"
          - condition: state
            entity_id: input_text.kitchen_preset
            state: "motion"
        sequence:
          - service: script.mood_set
            data:
              target_areas:
                - kitchen
              preset: default
mode: restart
```
#### Illuminance based mood change
The home gets to mood “Day” when the outdoor illuminance get over 4200
```yaml
alias: Day
description: ""
triggers:
  - trigger: numeric_state
    entity_id:
      - sensor.outdoor_illuminance
    above: 4200
    enabled: true
    for:
      hours: 0
      minutes: 5
      seconds: 0
conditions:
  - condition: time
    after: sensor.home_sun_solar_noon
    before: input_datetime.unwind
  - condition: or
    conditions:
      - condition: state
        entity_id: input_text.home_mood
        state: Evening
      - condition: state
        entity_id: input_text.home_mood
        state: Morning
actions:
  - action: input_text.set_value
    metadata: {}
    data:
      value: Day
    target:
      entity_id: input_text.home_mood
mode: restart
```
#### Save energy
After 15 minutes of no motion in bright preset, return to default preset
```yaml
alias: Kitchen - Auto - Bright -> Neutral
description: ""
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.kitchen_motion_occupancy
    to: "off"
    for:
      hours: 0
      minutes: 15
      seconds: 0
    from: "on"
conditions:
  - condition: state
    entity_id: input_text.kitchen_preset
    state: bright
actions:
  - condition: state
    entity_id: binary_sensor.kitchen_motion_occupancy
    state: "off"
    for:
      hours: 0
      minutes: 15
      seconds: 0
  - action: script.mood_set
    metadata: {}
    data:
      transition_time: 10
      target_areas:
        - kitchen
      preset: default
mode: single
```
#### Movie time
This one will lock the living room when the projector is turned on. When a media plays, it will set the living room into movie mode but after a 30 seconds pause, the living room will sync back with the home. When the projector got turned off, the living room get unlocked.
```yaml
alias: Movie
description: ""
triggers:
  - trigger: state
    entity_id:
      - media_player.projector
    to: playing
    id: Playing
  - trigger: state
    entity_id:
      - media_player.projector
    to: idle
    id: Stop
    for:
      hours: 0
      minutes: 0
      seconds: 30
  - trigger: state
    entity_id:
      - media_player.projector
    id: Pause
    for:
      hours: 0
      minutes: 0
      seconds: 30
    to: paused
    from: playing
  - trigger: state
    entity_id:
      - binary_sensor.movie_mode
    to: "on"
    id: Projector On
  - trigger: state
    entity_id:
      - binary_sensor.movie_mode
    to: "off"
    id: Projector Off
conditions: []
actions:
  - alias: If projector Off
    if:
      - condition: trigger
        id:
          - Projector Off
    then:
      - alias: Turn on back outdoors lights
        if:
          - condition: state
            state: "on"
            entity_id: light.outdoor_back
        then:
          - action: light.turn_on
            metadata: {}
            data: {}
            target:
              entity_id:
                - light.outdoor_front
      - action: input_boolean.turn_off
        metadata: {}
        data: {}
        target:
          entity_id: input_boolean.living_room_lock
  - choose:
      - conditions:
          - condition: trigger
            id:
              - Playing
          - condition: not
            conditions:
              - condition: state
                entity_id: input_text.living_room_mood
                state: Movie
        sequence:
          - action: script.music_stop
            metadata: {}
            data: {}
            enabled: false
          - action: script.mood_set
            metadata: {}
            data:
              target_areas:
                - living_room
              mood: Movie
              transition_time: 30
          - action: media_player.volume_set
            metadata: {}
            data:
              volume_level: 0
            target:
              entity_id: media_player.living_room
      - conditions:
          - condition: trigger
            id:
              - Projector Off
              - Stop
              - Pause
        sequence:
          - condition: state
            entity_id: input_text.living_room_mood
            state: Movie
          - action: script.mood_set
            metadata: {}
            data:
              target_areas:
                - living_room
              transition_time: 20
              mood: "{{ states('input_text.home_mood') }}"
      - conditions:
          - condition: trigger
            id:
              - Projector On
        sequence:
          - action: input_boolean.turn_on
            metadata: {}
            data: {}
            target:
              entity_id: input_boolean.living_room_lock
          - action: light.turn_off
            metadata: {}
            data: {}
            target:
              entity_id: light.salt_lamp
mode: parallel
max: 10
```
#### Bedroom Lock
Automatic bedroom lock based on the mood of the room
```yaml
alias: Children - Lock
description: Lock or unlock bedroom to protect them of lightning up during the night
triggers:
  - trigger: state
    entity_id:
      - input_boolean.children_lock
    to: "off"
    id: Unprotected
    enabled: true
  - trigger: state
    entity_id:
      - input_text.children_mood
    id: Unprotect
conditions: []
actions:
  - choose:
      - conditions:
          - condition: trigger
            id:
              - Unprotected
        sequence:
          - if:
              - condition: or
                conditions:
                  - condition: state
                    entity_id: input_text.children_mood
                    state: Night
                  - condition: state
                    entity_id: input_text.children_mood
                    state: Unwind
              - condition: template
                value_template: >-
                  {{ states('input_text.children_mood') !=
                  states('input_text.home_mood') }}
            then:
              - action: script.mood_set
                metadata: {}
                data:
                  target_areas:
                    - children
                  transition_time: 2
                  preset: default
                  mood: "{{ states('input_text.home_mood') }}"
                enabled: true
            alias: Sync with house if current mood is night or unwind
        alias: When unlocked
      - conditions:
          - condition: or
            conditions:
              - condition: trigger
                id:
                  - Protect
              - condition: state
                entity_id: input_text.children_mood
                state: Night
              - condition: state
                entity_id: input_text.children_mood
                state: Unwind
        sequence:
          - action: input_boolean.turn_on
            metadata: {}
            data: {}
            target:
              entity_id:
                - input_boolean.children_lock
        alias: Lock it!
      - conditions:
          - condition: trigger
            id:
              - Unprotect
          - condition: state
            entity_id: input_boolean.children_lock
            state: "on"
        sequence:
          - action: input_boolean.turn_off
            metadata: {}
            data: {}
            target:
              entity_id:
                - input_boolean.children_lock
        alias: Unprotect
mode: parallel
max: 10
```

#### Morning routine
In the morning, each room will get to morning individually. When the main rooms are all in the morning state, then the whole home get into morning mood.
```yaml
alias: Morning
description: Trigger morning mood and play music once per day
triggers:
  - entity_id:
      - binary_sensor.kitchen_motion_occupancy
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 45
    id: Motion Kitchen
    trigger: state
    from: "off"
  - entity_id:
      - binary_sensor.living_room_motion_occupancy
    to: "on"
    id: Motion Living Room
    trigger: state
    from: "off"
    for:
      hours: 0
      minutes: 0
      seconds: 45
  - entity_id:
      - binary_sensor.children_motion_occupancy
    to: "on"
    id: Motion Children
    trigger: state
    from: "off"
    for:
      hours: 0
      minutes: 0
      seconds: 45
  - at: input_datetime.morning
    id: MorningTime
    trigger: time
conditions:
  - condition: state
    entity_id: input_text.home_mood
    state: Night
  - condition: time
    after: input_datetime.early_morning
    before: sensor.home_sun_solar_noon
actions:
  - alias: Change house mood at scheduled morning time
    if:
      - condition: trigger
        id: MorningTime
    then:
      - data:
          option: Morning
        target:
          entity_id: input_text.home_mood
        action: input_select.select_option
      - stop: >Don’t need to compute the rest of the logic, the house will change to
          morning
  - alias: Motion Kitchen
    if:
      - condition: trigger
        id:
          - Motion Kitchen
      - condition: state
        entity_id: input_text.kitchen_mood
        state: Night
      - condition: state
        entity_id: input_boolean.do_not_disturb
        state: "off"
    then:
      - alias: Play music once per day
        if:
          - condition: state
            entity_id: binary_sensor.music_is_playing
            state: "off"
          - condition: state
            entity_id: input_boolean.away
            state: "off"
          - condition: state
            entity_id: input_boolean.do_not_disturb
            state: "off"
          - condition: state
            entity_id: person.francis
            state: home
        then:
          - action: script.music_play
            metadata: {}
            data:
              resume_last_playlist: false
              playlist: "{{ states('input_text.playlist_morning') }}"
      - data:
          transition_time: 30
          mood: Morning
          preset: default
          target_areas:
            - kitchen
        action: script.mood_set
  - alias: Motion Living room
    if:
      - condition: trigger
        id:
          - Motion Living Room
      - condition: state
        entity_id: input_text.living_room_mood
        state: Night
      - condition: state
        entity_id: input_boolean.do_not_disturb
        state: "off"
    then:
      - data:
          transition_time: 30
          mood: Morning
          preset: default
          target_areas:
            - living_room
        action: script.mood_set
  - alias: Motion Children
    if:
      - condition: trigger
        id:
          - Motion Children
      - condition: state
        entity_id: input_boolean.do_not_disturb
        state: "off"
      - condition: state
        entity_id: input_text.children_mood
        state: Grow Clock
      - condition: or
        conditions:
          - condition: state
            entity_id: input_text.children_mood
            state: Night
            enabled: true
        enabled: false
    then:
      - data:
          transition_time: 30
          mood: Morning
          preset: default
          target_areas:
            - children
        action: script.mood_set
  - alias: Set house mood to Morning when all rooms have updated
    if:
      - condition: and
        conditions:
          - condition: state
            entity_id: input_text.kitchen_mood
            state: Morning
          - condition: state
            entity_id: input_text.living_room_mood
            state: Morning
          - condition: state
            entity_id: input_text.children_mood
            state: Morning
    then:
      - action: input_text.set_value
        metadata: {}
        data:
          value: Morning
        target:
          entity_id: input_text.home_mood
mode: restart
```


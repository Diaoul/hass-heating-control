# 🔥 Heating Control Blueprint

**Version 1.0**

A smart, reliable Home Assistant blueprint for managing your heating based on presence, schedules, and real-world conditions.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FDiaoul%2Fhass-heating-control%2Fblob%2Fmain%2Fheating_control.yaml)

## ✨ Features

- 🌡️ **Temperature Control** - Set comfort and eco temperatures that match your lifestyle
- ⚡ **Boost Overlay** - Apply offset to comfort temperature during boost periods (default +1°C)
- 🚶 **Away Overlay** - Apply offset to comfort temperature when away during scheduled periods (default -1°C)
- 👥 **Smart Presence Detection** - Automatically adjust heating when you leave or return home
- 📅 **Schedule Integration** - Define comfort periods that fit your daily routine
- 🪟 **Window Detection** - Turn off heating when windows are open to save energy
- ❄️ **Frost Protection** - Keep your home safe during vacations with automatic low-temperature mode
- 🎭 **Guest Mode** - Acts like an additional person entity for presence detection
- 🎯 **Preset Support** - Optionally use your thermostat's built-in preset modes
- ⏱️ **Configurable Durations** - Fine-tune timing for all state changes

## 📦 Installation

Click the button above to import the blueprint directly into your Home Assistant.

## 🚀 Quick Start

1. **Create an automation** from the blueprint
2. **Configure your climate entities** (thermostats) - required
3. **Set your temperatures** - comfort and eco base temperatures, plus optional boost/away offsets
4. **Choose presence detection** - person entities and/or binary sensor
5. **(Optional)** Configure schedule, windows, and frost protection

The blueprint organizes all settings into collapsible sections for easy configuration.

## 🧠 How It Works

The blueprint uses a **priority-based system**:

1. **Heating off override** → Heating turns **OFF** (highest priority)
2. **Window/door open** → Heating **OFF**
3. **Frost protection** → Uses frost protection temperature
4. **Schedule + presence** → Uses comfort/boost/away temperature based on schedule and presence
5. **No schedule** → Uses comfort/boost (present) or eco (away) temperature

**Temperature Overlays:**

The blueprint uses a base comfort temperature with two optional overlays:
- **Boost Overlay** (⚡): Applies boost offset to comfort when boost schedule/sensor is active
  - Only applies during comfort periods (when you're home during scheduled times)
  - Example: 21°C comfort + 2°C boost offset = 23°C
- **Away Overlay** (🚶): Applies away offset to comfort when away during scheduled comfort periods
  - Configurable offset (default -1°C) or eco temperature fallback if offset is 0
  - Only applies when schedule is active but you're not present
  - Example: 21°C comfort + (-1°C) away offset = 20°C

**Presence Detection:**
- Combines person entities, presence sensors, and guest mode using configurable logic
- When both presence sensor and persons are configured, **both** must indicate presence (AND logic)
- Guest mode acts like an additional person entity

**Frost Protection:**
- Activates manually via override, or automatically after everyone is away for the configured duration
- Uses optional datetime helper for persistence across HA restarts
- Automatically deactivates when anyone returns home

## ⏱️ Optional: Frost Protection Helper

For reliable frost protection across HA restarts, create a datetime helper:

**1. Create Helper**

Go to **Settings** → **Devices & Services** → [**Helpers**](https://my.home-assistant.io/redirect/helpers/)

Create a **Date and/or time** helper:
- **Has date:** ✅ Enabled
- **Has time:** ✅ Enabled
- Example: `input_datetime.frost_protection_schedule`

**2. Configure in Blueprint**

Select your datetime helper in the **Frost Protection DateTime Helper (Optional)** field.

**Why?** The helper persists the scheduled activation time across restarts, preventing the duration timer from resetting.

## 🎛️ Configuration Tips

- **Durations:** Set person away duration longer (10-30 min) to avoid triggering on brief exits
- **Guest Mode:** Perfect for visitors - acts like someone being home
- **Window Detection:** Heating turns off immediately when windows open

## 🤝 Support

If you encounter issues:
- Test conditions in **Developer Tools** → **Template**
- Review automation traces in **Settings** → **Automations & Scenes** → _your automation_ → **Traces**
- Open an issue on [GitHub](https://github.com/Diaoul/hass-heating-control/issues)

---

## 📚 Advanced Documentation

<details>
<summary><b>🔍 Detailed Decision Logic (Click to expand)</b></summary>

### Temperature & HVAC Mode Priority

The blueprint evaluates conditions in priority order (highest to lowest):

| Priority | Condition | Temperature | HVAC Mode |
|----------|-----------|-------------|-----------|
| 1️⃣ **Highest** | Heating Off Override ON | - | **Off** |
| 2️⃣ | Window/Door Open | - | **Off** |
| 3️⃣ | Frost Protection Override ON | Frost | Heat |
| 4️⃣ | Frost Protection Scheduled (datetime reached) | Frost | Heat |
| 5️⃣ | Schedule Defined + Active + Present | Comfort + boost offset* | Heat |
| 6️⃣ | Schedule Defined + Active + Away | Comfort + away offset** | Heat |
| 7️⃣ | Schedule Defined + Inactive | Eco | Heat |
| 8️⃣ | No Schedule + Present | Comfort + boost offset* | Heat |
| 9️⃣ **Lowest** | No Schedule + Away | Eco | Heat |

*Boost offset: default +1°C, only applied when boost schedule/sensor is active
**Away offset: default -1°C, or eco temperature if offset is 0

### Presence Detection Priority

How presence is determined based on configuration:

| Priority | Configuration | Condition | Result |
|----------|---------------|-----------|--------|
| 1️⃣ **Highest** | Sensor + (Persons/Guest) configured | Sensor ON **AND** (Person home **OR** Guest ON) | **Present** |
| 2️⃣ | Sensor + (Persons/Guest) configured | Sensor OFF **OR** (No persons **AND** Guest OFF) | **Away** |
| 3️⃣ | Only Presence Sensor configured | Sensor ON | **Present** |
| 4️⃣ | Only Presence Sensor configured | Sensor OFF | **Away** |
| 5️⃣ | Only Person(s)/Guest configured | At least one person home **OR** Guest ON | **Present** |
| 6️⃣ | Only Person(s)/Guest configured | All persons away **AND** Guest OFF | **Away** |
| 7️⃣ **Lowest** | Nothing configured | - | **Present** (default) |

**Key Points:**
- **Guest Mode** is treated like a person entity (not an override). When Guest Mode is ON, it's as if a person is home.
- When **both** Presence Sensor and (Person entities/Guest Mode) are configured, **both must indicate presence** (AND logic). This ensures heating only runs when you're actually in the room, not just at home.
- Example: Guest ON + Sensor OFF = Away (both required when both configured)

### Detailed Behaviors

**Schedule Logic:**
- When schedule is **defined and active** → Defines comfort periods where boost/away overlays can apply
  - Present during scheduled period: Comfort + boost offset (if boost active)
  - Away during scheduled period: Comfort + away offset (or eco if offset = 0)
- When schedule is **defined but inactive** → Uses eco temperature (night/off-hours)
- When schedule is **not defined at all** → Relies purely on presence (present = comfort + boost offset, away = eco)

**Frost Protection:**
- **Manual activation:** Toggle the Frost Protection Override input boolean anytime
- **Automatic activation:** When everyone is away (persons AND guest mode) for the configured frost protection duration
  - Considers: Person entities + Guest Mode
  - **Does NOT consider:** Presence sensor (room motion doesn't affect frost protection)
  - Uses datetime helper (optional but recommended) for persistence across HA restarts
  - Automatically deactivates when anyone returns home OR guest mode turns on
- **Remote pre-heating tip:** When away and frost protection is active, turn the override toggle OFF remotely (via HA app) to cancel frost protection and pre-heat your home before returning. The system includes a safeguard: if you're still away, frost protection will automatically reschedule for the configured duration (preventing energy waste if plans change).
- **Rationale:** Frost protection mode = house empty (not just "no motion in room")

**Window Detection:**
- Any window/door sensor showing "open" → Immediately turns heating **OFF**
- Resumes normal operation when all windows/doors close

**Overrides:**
- **Window/Door Open** → Highest priority! Immediately forces HVAC **OFF** (overrides everything including frost protection)
- **Heating Off Override** → Forces HVAC mode to **OFF** (second highest priority, overrides frost protection)
- **Frost Protection Override** → Forces frost protection mode (highest priority for temperature when heating is allowed)
- All overrides persist until conditions change or you manually turn them off

</details>

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

Made with ❤️ for the Home Assistant community

---
name: smart-home
description: FRITZ!Box 7690 smart home setup covering Zigbee device integration, DECT ULE/HAN FUN devices (plugs, thermostats, buttons, sensors), FRITZ!Smart Gateway, smart home routines/automations, and third-party device compatibility. Use when pairing smart home devices, creating automations, setting schedules, or troubleshooting smart home connectivity.
---
# FRITZ!Box 7690 Smart Home

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

The FRITZ!Box 7690 integrates a smart home hub supporting two wireless protocols:
- **Zigbee** – built-in Zigbee radio for direct pairing with Zigbee-compatible devices (Philips Hue, IKEA TRÅDFRI, LEDVANCE, OSRAM, Müller Licht, and others)
- **DECT ULE / HAN FUN** – for AVM FRITZ!DECT devices (plugs, radiator controls, buttons, sensors)

Smart home devices are managed centrally in the FRITZ!Box web interface under **Smart Home**.

---

## Zigbee Device Pairing

### Requirements
- The FRITZ!Box 7690 must have an active internet connection for the initial Zigbee setup.
- The Zigbee device must be reset/in pairing mode before registration.

### Steps to pair a Zigbee device
1. In the FRITZ!Box web interface, go to **Smart Home → Devices and Groups**.
2. Click **Register Device** → **Zigbee**.
3. Put the Zigbee device into pairing mode (refer to the device manual – usually a button press or power cycle).
4. The FRITZ!Box discovers the device; confirm the pairing.
5. Give the device a name and assign it to a room (optional).

### Compatible Zigbee devices
The FRITZ!Box 7690 is certified for a growing list of Zigbee devices. Check the official AVM compatibility list for supported devices including:
- Smart plugs and dimmers
- LED bulbs and strips
- Blind and shutter actuators
- Sensors (motion, contact, temperature)

### Zigbee channels
The FRITZ!Box automatically selects the Zigbee channel to minimize interference with Wi-Fi. You can view and change the channel at **Smart Home → Zigbee Settings**.

---

## DECT ULE / HAN FUN Devices

These are AVM FRITZ!DECT branded devices registered via the DECT base station.

### FRITZ!DECT device types

| Device type | Examples |
|-------------|---------|
| Smart plug | FRITZ!DECT 200, FRITZ!DECT 210 |
| Radiator thermostat | FRITZ!DECT 301, FRITZ!DECT 302 |
| Repeater | FRITZ!DECT 100 |
| Push button | FRITZ!DECT 440 |
| Floor sensor | FRITZ!DECT 500 |

### Registering a FRITZ!DECT device
1. Go to **Smart Home → Devices and Groups → Register Device → FRITZ!DECT**.
2. Press the **Connect** button on the FRITZ!DECT device until the status light flashes.
3. The device registers and appears in the device list.
4. Assign a name and room.

---

## FRITZ!Smart Gateway

For households with many Zigbee devices or larger coverage areas, use the **FRITZ!Smart Gateway** as an additional Zigbee coordinator:
- Register the FRITZ!Smart Gateway as a FRITZ!DECT device (it connects via DECT ULE).
- All Zigbee devices behind the Gateway are managed centrally through the FRITZ!Box web interface.
- Devices registered to a FRITZ!Smart Gateway appear in the same Smart Home device list.

Register the FRITZ!Smart Gateway:
1. **Smart Home → Devices and Groups → Register Device → FRITZ!Smart Gateway**.
2. Press the Connect button on the Gateway.
3. After registration, pair Zigbee devices through the Gateway by selecting it as the target coordinator during pairing.

---

## Smart Home Devices Overview

After pairing, devices appear at **Smart Home → Devices and Groups**:
- Toggle devices on/off
- Set target temperature (thermostats)
- View power consumption statistics (smart plugs)
- Group devices (e.g., "Living Room Lights")

### Rooms and groups
- Create rooms (e.g., "Kitchen", "Bedroom") at **Smart Home → Rooms**.
- Assign devices to rooms for easier management.
- Create device groups to control multiple devices together.

---

## Smart Home Automations (Routines)

Create routines to automate smart home actions:
1. Go to **Smart Home → Routines → Create Routine**.
2. Choose a **trigger**:
   - Time of day / day of week
   - Sunrise / Sunset (offset supported)
   - Device state change (e.g., push button pressed, sensor triggered)
   - Telephone event (e.g., incoming call from a specific number)
   - Temperature threshold
3. Choose an **action**:
   - Switch device on/off
   - Set thermostat temperature
   - Send push notification (via MyFRITZ! app)
4. Optionally add **conditions** (e.g., only run on weekdays).
5. Save and activate the routine.

### Example routines
- Turn on the porch light at sunset
- Set heating to comfort temperature on weekday mornings
- Send a push notification when the doorbell sensor is triggered
- Turn off all plugs when the last phone leaves the home network (geofencing via MyFRITZ!)

---

## Heating Schedules (Thermostats)

For FRITZ!DECT radiator valves:
1. Select the thermostat in **Smart Home → Devices**.
2. Under **Heating Schedule**, define temperature time windows for each day.
3. Use **Eco temperature** for energy saving when away/asleep and **Comfort temperature** for active hours.
4. Holiday schedules let you pre-set temperatures for vacation periods.

---

## Smart Home via FRITZ!App Smart Home

The **FRITZ!App Smart Home** (iOS/Android) provides remote control of smart home devices when you are away from home:
- Toggle devices, adjust temperatures, view power usage.
- Receive push notifications from routines.
- Requires **MyFRITZ!** remote access to be enabled.

---

## Third-Party Devices (from Other Manufacturers)

The FRITZ!Box 7690 supports third-party Zigbee and HAN FUN devices. Compatibility details:
- Check the **AVM Zigbee compatibility list** at the AVM website.
- Devices must support the standard Zigbee Home Automation (ZHA) or Zigbee 3.0 profile.
- Some advanced functions (like energy monitoring) may only be available for AVM FRITZ!DECT devices.

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| Zigbee device not found during pairing | Reset the device to factory; bring it close to the FRITZ!Box; retry |
| DECT ULE device won't register | Ensure FRITZ!Box is not in DECT Eco mode; press Connect button correctly |
| Device shown as offline | Check battery level; verify range; check for Zigbee channel interference |
| Routine not triggering | Check that routine is active; verify conditions are met; confirm device state |
| FRITZ!Smart Gateway not recognized | Update FRITZ!OS; check DECT range |

---

## References

- [Using a smart home device from another manufacturer](https://en.avm.de/service/knowledge-base/dok/FRITZ-Box-7690/3435_Using-a-smart-home-device-from-another-manufacturer/)
- [Registering a smart home device with the FRITZ!Smart Gateway](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-7690/3728_Registering-a-smart-home-device-with-the-FRITZ-Smart-Gateway)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)

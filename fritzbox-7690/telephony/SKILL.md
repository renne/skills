---
name: telephony
description: FRITZ!Box 7690 telephony setup covering DECT cordless phone registration, VoIP/SIP internet telephony, analog phone ports, answering machines, call lists, fax-to-mail, call blocking, and FRITZ!App Fon. Use when configuring phones, telephone numbers, voicemail, fax, or call management on the FRITZ!Box.
---
# FRITZ!Box 7690 Telephony

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

The FRITZ!Box 7690 is a full-featured telephone system supporting:
- **DECT cordless phones** – integrated base station for up to 6 handsets
- **Analog phones / fax** – via the **FON 1** and **FON 2** RJ11 ports
- **IP phones** – SIP clients connected via LAN or Wi-Fi
- **Internet telephony (VoIP)** – register SIP accounts from any provider
- **Up to 5 answering machines** – integrated in the FRITZ!Box
- **FRITZ!App Fon** – use a smartphone as a DECT handset via Wi-Fi

Access telephony settings at **Telephony** in the web interface (http://fritz.box).

---

## Telephone Numbers (SIP / VoIP Accounts)

Before configuring phones, add your telephone numbers:

1. Go to **Telephony → Telephone Numbers → Add Telephone Number**.
2. Select your VoIP provider from the list, or choose **Other VoIP Provider**.
3. Enter the SIP credentials:
   - Username / phone number
   - Password
   - SIP registrar / proxy server (from your provider)
4. Click **Next** and complete the wizard.
5. Assign incoming/outgoing numbers to telephony devices in **Telephony → Telephony Devices**.

---

## Registering DECT Cordless Phones

The FRITZ!Box 7690 has an integrated DECT base station (DECT-GAP/CAT-iq compatible).

### Registration steps
1. Go to **Telephony → Telephony Devices → Configure New Device → Cordless (DECT) Telephone**.
2. The FRITZ!Box enters registration mode (DECT LED flashes).
3. On your cordless phone, activate registration mode (refer to the phone's manual – usually a long press on the Find/Page button or via the menu).
4. Enter the FRITZ!Box DECT PIN when prompted on the handset (default PIN: **0000**).
5. The handset registers and appears in **Telephony → Telephony Devices**.

### Supported phones
- All **FRITZ!Fon** models (recommended; extra features like phonebook sync, HD audio, display menus)
- Any **DECT-GAP** compatible handset (Gigaset, Panasonic, etc.)
- Up to **6 DECT handsets** can be registered simultaneously

### Troubleshooting DECT registration
- Temporarily disable **DECT Eco Mode** (saves energy but can interfere with registration): **Home Network → DECT → DECT Eco**.
- Ensure the phone is in range of the FRITZ!Box.
- Reset the DECT base: In the FRITZ!Box web interface, go to **Home Network → DECT → Reset DECT Base**.
- Verify the phone supports DECT-GAP (most modern cordless phones do).

---

## Analog Phones

Connect analog phones or fax machines to the **FON 1** / **FON 2** ports using an RJ11 cable.

1. Go to **Telephony → Telephony Devices → Configure New Device → Fixed-line Telephone (FON port)**.
2. Select the appropriate FON port (FON 1 or FON 2).
3. Assign incoming and outgoing telephone numbers.

---

## IP Phones (SIP via LAN/Wi-Fi)

Register a SIP softphone or IP desk phone with the FRITZ!Box acting as a SIP registrar:

1. Go to **Telephony → Telephony Devices → Configure New Device → IP Telephone (LAN/Wi-Fi)**.
2. The FRITZ!Box generates a username and password for the SIP client.
3. Configure the IP phone/softphone with:
   - **SIP registrar**: `fritz.box` (or 192.168.178.1)
   - **Username**: as shown in the FRITZ!Box
   - **Password**: as shown in the FRITZ!Box
4. Assign telephone numbers to this device.

---

## Answering Machines

The FRITZ!Box supports up to **5 integrated answering machines**.

### Setup
1. Go to **Telephony → Telephony Devices → Configure New Device → Answering Machine**.
2. Name the answering machine (e.g., "Family", "Business").
3. Assign which telephone numbers it should answer.
4. Configure:
   - **Recording time** (maximum length of messages)
   - **Greeting message** – use the default or upload a custom WAV/MP3 file
   - **Schedule** – active only during certain hours
   - **Email notification** – send new voicemails as email attachments

### Accessing messages
- Via any registered phone: Dial the internal number of the answering machine (e.g., `**600`)
- Via the FRITZ!Box web interface: **Telephony → Calls** → filter for answering machine messages
- Via FRITZ!App Fon (push notifications for new messages)
- Via FRITZ!NAS / Samba share: messages stored as WAV files

---

## Call Lists

View all incoming, outgoing, and missed calls:
- **Telephony → Calls**
- Filter by call type (missed, incoming, outgoing) and date
- Export call list as CSV

---

## Call Blocking

Block unwanted callers:
1. Go to **Telephony → Call Blocking**.
2. Add numbers to block:
   - Specific numbers
   - Anonymous calls (no caller ID)
   - International calls
3. Blocked calls are silently rejected or redirected to the answering machine.

---

## Call Diversion

Redirect incoming calls:
1. **Telephony → Telephony Numbers → select a number → Diversion**.
2. Configure diversion rules: always, when busy, or when unanswered.
3. Divert to another number or internal extension.

---

## Fax via FRITZ!Box

- Connect a fax machine to a FON port or use the integrated **FRITZ!Box fax** software.
- Enable **Fax-to-Email**: received faxes are forwarded as PDF attachments to a configured email address.
- Configure at **Telephony → Telephony Devices → Configure New Device → Fax Device**.

---

## FRITZ!App Fon

Use a smartphone as a DECT handset when on the home Wi-Fi:
- Install **FRITZ!App Fon** (iOS/Android) on the smartphone.
- The app registers the smartphone as a DECT device when connected to the FRITZ!Box Wi-Fi.
- Supports HD audio, phonebook access, and answering machine access.

---

## Internal Calls and Intercom

- Each telephony device gets an internal number (e.g., DECT handset 1 = **611**, answering machine = **600**).
- Call internal numbers directly from any registered phone.
- View internal number assignments at **Telephony → Telephony Devices**.

---

## References

- [Setting up a telephone in the FRITZ!Box](https://en.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/3569_Setting-up-a-telephone-in-the-FRITZ-Box/)
- [Setting up and using the FRITZ!Box answering machine](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/6_Setting-up-and-using-the-FRITZ-Box-answering-machine/)
- [Cannot register a cordless phone – FRITZ!Box 7690](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/65_Cannot-register-a-cordless-phone/)
- [Using an IP telephone in the FRITZ! home network](https://en.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/268_Using-an-telephone-or-internet-telephony-software-in-the-FRITZ-home-network/)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)

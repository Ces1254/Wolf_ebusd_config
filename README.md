# Wolf ebusd messages configuration

eBus configuration files for **Wolf heating systems** – BM-2 / FHA / MM-2

### Overview

This repository contains the `.csv` configuration files I currently use with `ebusd` to monitor and control a **Wolf heating system** based on an FHA heat pump.

The `master` folder includes a calculation sheet listing all parameters (260+) that I have decoded by analyzing the eBus traffic of my system.

The parameter list is based on information shared by other users — in particular the excellent work by [zivillian](https://github.com/zivillian/ism7mqtt/blob/8265271df0c7fddf044f61151740b627771c934c/PROTOCOL.md) — and my own additional reverse engineering effort to decode messages for newer Wolf models, particularly the FHA heat pump family.

---

## System Configuration

My installation consists of:

* **FHA split heat pump** (master 3 - slave 8)
* **BM-2 room controller** (master 30 - slaves 35 & 85)
* **MM-2 mixer module** (master 70 - slaves 51 & 75)
* **Wolfnet interface** (master ff)

The system includes:

* Floor heating
* 200 L domestic hot water (DHW) tank   
The association of the slaves 51 and 75 to the master 70 and the MM-2 mixer unit is my deduction, but still hypothetical as I am missing a firm confirmation: any insight to confirm or reject is highly appreciated.  
---

## eBus Messaging Details

In this topology, both the **BM-2** and the **Wolfnet** interface periodically poll other devices to read parameter values.

### BM-2 Behavior

* Polls single parameters per read request
* Sends setpoints to the FHA and MM-2
* Provides internal room temperature measurements

### Wolfnet Behavior

* Continuously reads parameters even when no user is connected
* Expands the set of requested parameters when a user is connected, depending on the page displayed in the web app
* Sends write commands when a user changes a setpoint

Unlike the BM-2, Wolfnet typically reads **five unrelated parameters in a single message**.
The parameter combination varies depending on:

* System state
* System topology
* Active user interaction

As a result, multiple message descriptors may exist for messages that include overlapping parameter sets.

---

## Bus Load Considerations

The described topology generates significant eBus traffic:

* Average bus occupancy exceeds **40% of bandwidth**
* Occasional saturation at **~218 symbols/sec**

For this reason, my monitoring approach prioritizes **passive listening** rather than active polling.

A dedicated `listener.csv` file contains all observed message patterns.

Note:

* The list of Wolfnet read combinations is not exhaustive (message like `ff0850220b540200...`)
* Most parameters of interest are read via the BM-2
* One important exception is **room temperature**, which originates internally in the BM-2 and must either:

  * Be passively decoded from Wolfnet polling, or
  * Be actively read from the BM-2

---

## Reading from the BM-2 Controller

The BM-2 appears on the eBus as:

* Master at logical address `30`
* Multiple logical slave addresses

In my system:

* Slave `35` → Internal controller parameters
* Slave `85` → Parameters affecting FHA operation

It is unclear whether other heating generators controlled by a BM-2 would appear under different slave addresses.

The configuration files explicitly reference slave `85` where required.

---

## Writing to the BM-2 Controller

Although many parameters can technically be modified, only a limited number of write descriptors (`w`) are included in the configuration.

This is intentional:

* Occasional setpoint changes can be handled via the Wolf web app
* The primary goal of this project is monitoring

Exposed write parameters allow:

* Enabling/disabling DHW
* Enabling/disabling floor heating
* Setting DHW temperature
* Setting room temperature setpoints

### Write Command Structure

Write commands can generally be derived from the corresponding read command by:

1. Replacing read code `0x5022` with `0x5023`
2. Adding a fixed `ULG` suffix to value `0x000001d5`(encoded as the literal `write` in the message description)

### Known ebusd Limitation

`ebusd` currently expects every master/slave transaction to include a slave response.

Wolf write commands **do not generate a response**, resulting in:

```
ERR: read timeout
```

This error should be ignored.

See:
[https://github.com/john30/ebusd/issues/1696](https://github.com/john30/ebusd/issues/1696)

Despite the timeout, the write command is successfully received and executed by the BM-2.

---

## Language Support

The configuration files use English parameter names.

Where possible, terminology matches the wording used in Wolf’s English web interface.
However:

* Some translations may not be ideal
* Minor inconsistencies may exist

Contributions and improvements are very welcome.

---

## Home Assistant Integration

I plan to publish a custom **Home Assistant (HA)** integration that:

* Imports entities directly from `ebusd`, without the need for an `MQTT` service
* Simplifies monitoring and automation
* Provides structured integration into HA

Planned release: **March 2026**

---

## Disclaimer

These files are the result of personal reverse engineering work performed on my own system.

I am not affiliated with Wolf or any other company.

**Use this information at your own risk.** I cannot accept responsibility for any direct or indirect damage resulting from its use.

Contributions, corrections, and shared findings from other users are highly appreciated.


---
date: '2025-12-06'
title: 'Fixing Omarchy Audio'
tags: ["omarchy","arch","audio","linux"]
showToc: true
---

Recently I have been having problems on my Arch Linux omarchy setup. The system does not recognize my audio jack therefore I cannot use my wired headphones.
I have been trying for quite some time to fix it luckily I managed to fix it and this is how.

## Finding out audio codec

To find what device does the audio encoding and decoding you can use this command.

```bash
cat /proc/asound/card0/codec* | grep -i Codec
```

This command shows you what codec your computer uses. Depending on the device you can search for known issues and fixes for the problems.
My computer uses `Conexant CX118800`, I found this codec is known to have a lot of issues when using it for Linux.

## Installing the tools

The tools we will need are `hdajackretask` for the pin remapping, `alsamixer` and `alsactl` for monitoring and `wpctl` for output management.

```bash
sudo pacman -S alsa-tools alsa-utils
```

## Using the tools

```bash
alsactl monitor
```
This command monitors all changes to the audio hardware, it shows volume changes, muting, connecting/disconnecting of audio devices and more. We can see if our system can recognize if headphones are connected.

```bash
wpctl status
```
This command displays the current state of objects in our PipeWire multimedia framework.

## Mapping the pins for audio devices

To fix our firmware not recognizing the audio jack we will use `hdajackretask` tool to override pins on our codec.

```bash
sudo hdajackretask
```

After opening the `hdajackretask` tool we can select a codec which is `Conexant CX118800` for our hardware. Here we have multiple pins recognized and we will setup them up how we need. We will check the option for Advanced override for more options.

1. Pin ID 0x16, this pin description says: Black Headphone, Left side. This indicates us what it is and where is it located on the computer.
   - We will check the Override checkbox,
   - `Connectivity`, select the Jack option,
   - `Device`, select the Headphone option,
   - `Jack`, select the 3.5 mm option,
   - `Jack Selection`, select Present to activate it.

2. Pin ID 0x17, this pin description says: Internal Speaker, Rear side. 
   - We will check the Override checkbox,
   - `Connectivity`, select the Internal option,
   - `Device`, select the Speaker option,
   - `Jack Selection`, select Not present.

3. Pin ID 0x19, this pin description says: Black Mic, Left side. If there is only one audio jack this means we setup the jack for both headphone and microphone detection.
   - We will check the Override checkbox,
   - `Connectivity`, select the Jack option,
   - `Device`, select the Microphone option,
   - `Jack`, select the 3.5 mm option,
   - `Jack Selection`, select Present to activate it.

4. Pin ID 0x1a, this pin description says: Internal Mic, Top.
   - We will check the Override checkbox,
   - `Connectivity`, select the Intenal option,
   - `Device`, select the Microphone option,
   - `Jack Selection`, select Not present.

After setting up all the pins we can click Apply now and test and verify it everything works correctly, we want to also check the output from the monitoring tools.

When everything works correctly we can install boot override to make it permanent. Now we can reboot and it should work permanently.
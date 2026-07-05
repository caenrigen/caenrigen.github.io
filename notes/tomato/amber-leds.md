# FreshTomato - Taming the Amber LEDs

The amber LEDs on my NETGEAR R700 would be a warm subtle illumination for the stairs in
my house when the sleepy me descends for a nocturnal toilet visit.

So let's fire up the tomato in FreshTomato in yet another home automation rabbit hole.

After squeezing the tomatoes, add the secret sauce to one of the FreshTomato's scripts
via its Web GUI. `Administration → Scripts → WAN Up (main)` might be a good idea
since some leds can reset back to white after a temporary WAN connection loss.

```sh
sleep 20

# ######################################################################################
# Manual LEDs control
# ######################################################################################

# Intended effect: all white LEDs off, amber (redish/orangish) LEDs always on, and
# no blinking (neither white nor amber, though either could be achieved in principle).

# Stop LED daemons/driver from controlling the LEDs
killall blink 2>/dev/null
killall blink_5g 2>/dev/null
wl -i eth1 leddc 1 2>/dev/null
wl -i eth2 leddc 1 2>/dev/null

gpio enable 2 # Power white LED off
gpio disable 3 # Power amber LED on
gpio enable 9 # WAN white LED off
gpio disable 8 # WAN amber LED on

# Did not test the WiFi LEDs in detail, my WiFi is turned off on my R7000
gpio enable 12 # 5 GHz LED off
gpio enable 13 # 2.4 GHz LED off

# These reset when connecting USB devices
gpio enable 17 # USB LED off
gpio enable 18 # USB LED off

gpio disable 14 # WPS button LED off
gpio disable 15 # WiFi button LED off

# WAN and LAN LEDs mode reprogram

# LED enable and mode, set to normal/"auto"
et robowr 0x00 0x16 0x01bf 2
et robowr 0x00 0x18 0x01ff 2
et robowr 0x00 0x1a 0x01ff 2

# --------------------------------------------------------------------------------------

# Registers 0x10 (LAN) and 0x12 (WAN) choose the LED status function, according to GPT,
# which seems to have figured it out from https://www.farnell.com/datasheets/2830672.pdf
# bit 15  PHYLED3
# bit 14  BroadSync HD Link
# bit 13  1G/ACT
# bit 12  10/100M/ACT
# bit 11  100M/ACT
# bit 10  10M/ACT
# bit 9   SPD1G
# bit 8   SPD100M
# bit 7   SPD10M
# bit 6   DPX/COL
# bit 5   LNK/ACT
# bit 4   COL
# bit 3   ACT
# bit 2   DPX
# bit 1   LNK
# bit 0   PHYLED4

# White slot = COL, normally false (COL stands for collision)
# Amber slot = PHYLED4, apparently this function is always true on my R7000
et robowr 0x00 0x10 0x0011 2
et robowr 0x00 0x12 0x0011 2

# Alternative:
# White slot = SPD10M, false unless a 10 Mbps device connected
# Amber slot = PHYLED4
# et robowr 0x00 0x10 0x0081 2
# et robowr 0x00 0x12 0x0081 2
```

If you are using any USB devices, add to your `USB and NAS → USB Support → Hotplug script`:

```sh
# sleep 3 # some sleep might be required
gpio enable 17 # USB LED off
gpio enable 18 # USB LED off
```
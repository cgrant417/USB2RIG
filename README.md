# USB2RIG
USB2RIG Documentation

This repository contains information on wiring the USB2RIG. The USB2RIG is an inexpensive CM108B audio interface with a serial port included to provide an interface between a computer and a ham radio for use with Packet, FT8, VARA, PSK31, etc. The USB2RIG is available on Aliexpress and Amazon but lacks documentation. Hopefully this helps. 

Please note I have no affiliation with the company that produces it.

The USB2RIG works well for voice and slow digital modes such as FT8. In bench tests against a Digirig, though, its audio stream had timing faults: about 1 frame in 6 was lost when sending 1200 baud packet, and fast modes like VARA ran at a fraction of their normal speed. See [Audio timing tests: USB2RIG vs Digirig](timing-tests.md) for the measurements and some Linux setup tips.

73,
Cam VE3CA


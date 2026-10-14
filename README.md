# OMTech Pro laser: Wi-Fi setup

Put an OMTech Pro 2440, Pro 3655 or similar Pro-series laser on your existing Wi-Fi. LightBurn can then reach it from your normal network, with no USB cable and no hopping onto the laser's hotspot.

The laser has a small Padavan-based Wi-Fi router built in. The manual barely mentions it. This repo shows how to switch that router to access-point mode, bridge it to your home Wi-Fi, and point the Ruida controller and LightBurn at it.

- **[GUIDE.md](GUIDE.md)**: a friendly step-by-step for non-technical owners, about an hour.
- **[TECHNICAL-NOTES.md](TECHNICAL-NOTES.md)**: architecture, port map, MAC translation, why each step matters, quirks, and errors in the manual.
- **[router-settings-reference.md](router-settings-reference.md)**: a sanitized settings capture from a working unit.

> ⚠️ Never cable the laser's external Ethernet jack into your home network, before or after setup. See the guide for why.

This is a community write-up, not official OMTech support. There's no warranty, and you use it at your own risk. Hardware can differ between units.

Licensed [CC BY 4.0](LICENSE).

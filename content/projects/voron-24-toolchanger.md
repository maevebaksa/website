---
title: "Voron 2.4 Toolchanger"
description: "Four-tool Voron platform integrating tool identity, sensing, mapping, failover, and distributed toolhead electronics."
image: "images/voron-24-toolchanger.webp"
tags: ["portfolio"]
---
<div class="mb-project-hero">
  <div class="image"><img src="../../images/voron-24-toolchanger.webp" alt="Black-and-red Voron 2.4 with four docked toolheads"></div>
  <div class="mb-project-facts">
    <div><span>TYPE</span><strong>COREXY / SYSTEMS</strong></div>
    <div><span>PERIOD</span><strong>2026</strong></div>
    <div><span>DOMAIN</span><strong>Mechanical systems / additive manufacturing</strong></div>
    <div><span>STATUS</span><strong>Development platform / documented work</strong></div>
  </div>
</div>

<p class="mb-page-lede">Four-tool Voron platform integrating tool identity, sensing, mapping, failover, and distributed toolhead electronics.</p>

## System integration

The machine is a four-tool Voron 2.4 built around physical/logical tool separation, repeatable toolchanging, per-tool filament state, sensing, calibration, and host-side workflow integration.

The project became as much a systems problem as a mechanical one: tool identity has to remain coherent across G-code, Klipper, Moonraker, the toolchanger stack, filament metadata, user interfaces, and the actual hardware.

[Read the full technical note](../../notes/the-voron-toolchanger-is-a-systems-project/)
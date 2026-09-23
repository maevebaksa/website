---
title: "Polar3D Klipper Conversion"
description: "A legacy polar printer converted to Klipper and extended to handle center-crossing Cartesian toolpaths."
image: ""
tags: ["portfolio"]
---
<div class="mb-project-hero">
  <div class="image"><span class="mb-ascii">[ + ]</span></div>
  <div class="mb-project-facts">
    <div><span>TYPE</span><strong>POLAR / KINEMATICS</strong></div>
    <div><span>PERIOD</span><strong>2026</strong></div>
    <div><span>DOMAIN</span><strong>Mechanical systems / additive manufacturing</strong></div>
    <div><span>STATUS</span><strong>Development platform / documented work</strong></div>
  </div>
</div>

<p class="mb-page-lede">A legacy polar printer converted to Klipper and extended to handle center-crossing Cartesian toolpaths.</p>

## Kinematics work

The conversion extends Klipper's polar kinematics so ordinary Cartesian G-code can cross the XY origin while respecting the physical nonnegative-radius mechanism.

[Read the full technical note](../../notes/teaching-a-polar-3d-printer-to-cross-the-origin/)
---
title: "Polar3D — Klipper Conversion"
weight: 8
description: "A Polar3D printer converted to Klipper and extended to support ordinary Cartesian toolpaths across the polar origin."
type: "POLAR / KINEMATICS"
year: "2026"
image: "images/polar3d.jpg"
alt: "Polar3D printer with circular bed and vertical tower"
domain: "Mechanical systems / additive manufacturing"
status: "Klipper conversion / kinematics development"
tags: ["portfolio"]
---
## Kinematics work

The machine uses a rotating circular bed and radial arm rather than conventional Cartesian XY motion. I converted it to Klipper and extended the polar kinematics so ordinary Cartesian G-code can cross the XY origin while respecting the mechanism's nonnegative radial coordinate.

[Read the kinematics note](../../notes/teaching-a-polar-3d-printer-to-cross-the-origin/)

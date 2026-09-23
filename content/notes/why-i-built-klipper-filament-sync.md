---
title: "Klipper Filament Metadata Synchronization"
date: 2026-09-22
description: "Technical notes on synchronizing filament material, color, temperature, and tool identity across Klipper, Moonraker, Mainsail, KlipperScreen, RatOS, and multi-tool printer workflows."
tags: ["klipper", "software", "additive-manufacturing"]
categories: ["software"]
aliases:
  - "/2026/09/22/why-i-built-klipper-filament-sync/"
---
<p>Klipper provides detailed machine state for motion, heaters, endstops, fans, and other hardware, but filament identity is not inherently represented with the same consistency. A printer can know that a particular extruder is active and that its heater is at 215 °C without having a durable, shared representation of the fact that the tool is physically loaded with a particular material and color, or that a preferred nozzle and bed temperature should be associated with that material. This project adds that missing metadata layer and keeps it synchronized across the interfaces used to operate the printer.</p>

<h2>System scope</h2>

<p>Klipper Filament Sync stores material, color, nozzle temperature, and bed temperature for each physical tool. The information is exposed through Moonraker and used by Mainsail and, when installed, KlipperScreen. On multi-tool printers it also coexists with logical-to-physical tool mapping, where the tool requested by G-code is not necessarily the same numbered physical tool mounted on the machine.</p>

<p>The same package is intended to operate on both standard Klipper installations and RatOS. Those environments share Klipper and Moonraker concepts but differ in the macros and helper behavior surrounding filament loading, unloading, sensors, and printer configuration. The integration therefore cannot treat every printer as a blank system or replace the existing filament workflow with one universal implementation.</p>

<p>The project is distributed as a unified package with host-side services, Klipper configuration, a Mainsail integration, an optional KlipperScreen panel, verification tooling, and update-manager configuration. Current releases build the Mainsail frontend away from the printer and publish tested archives. The printer downloads the built result instead of compiling TypeScript, Vue components, or frontend dependencies locally.</p>

<h2>Representing filament state</h2>

<p>The first implementation question was where filament information should live. A browser-only color selector is easy to build, but it creates state that disappears outside that browser. A Klipper macro variable is more persistent, but it is not automatically available to every host-side interface. A separate database introduces another state store that must be reconciled with the printer.</p>

<p>The current design uses Moonraker lane data as the shared host-side representation. Commands such as <code>SET_TOOL_FILAMENT</code> update the saved metadata for a physical tool, while the Mainsail and KlipperScreen interfaces read from the same state. The practical benefit is that a material change made from one interface can be visible from another without each interface maintaining an independent filament profile.</p>

<p>A bridge can also update lane data from slicer-side information. This matters because the slicer has information about the planned job while the printer has information about what is physically loaded. Those are related but not identical concepts. The integration does not assume that a slicer-assigned logical tool number uniquely identifies a physical extruder.</p>

<h2>Physical tools and logical tools</h2>

<p>Tool mapping became one of the more important constraints. In a conventional printer, T0 usually means both “the first logical extruder in the G-code” and “the first physical extruder on the machine.” A toolchanger can separate those identities. A single-tool print sliced for T0 can be intentionally assigned to physical tool 3, or a print can move a logical tool assignment to another physical tool after filament runs out.</p>

<p>This means filament metadata has to be associated with the physical tool while print intent can remain associated with the logical tool. If material metadata follows the logical number instead, remapping a tool can make the user interface report a material that is no longer physically present. If metadata follows only the physical number but the interface ignores mapping, it can be unclear which material a current G-code command will actually use.</p>

<p>The integration therefore retains the existing tool-mapping workflow on multi-tool machines and avoids displaying it on single-tool systems. The single-tool case was an important design boundary. A feature that is required on a four-tool machine can become unnecessary complexity on an ordinary printer. The interface detects the configured physical tool count and changes the available controls accordingly.</p>

<h2>Filament sensors and temperature behavior</h2>

<p>Filament metadata also intersects with load and runout behavior. On printers using Orbiter-style filament sensing, the sensor reports physical events such as runout, insertion, or a button press. The metadata system has to coexist with those callbacks rather than simply treating them as UI events. On RatOS systems, native sensor callbacks and private motion routines are retained. Public load and unload entry points can use the saved filament temperature when an explicit temperature is not supplied.</p>

<p>This introduced a less obvious distinction between “temperature as metadata” and “temperature as a command.” A saved nozzle temperature describes the preferred processing condition for a material. It should not automatically cause a cold printer to heat simply because a user changed a filament profile. Heating remains an explicit machine action. The saved value becomes an input to load, unload, or print-start behavior when those routines request it.</p>

<h2>Unexpected integration challenges</h2>

<p>The largest difficulties were not in storing a material name or drawing a color swatch. They appeared at the interfaces between systems.</p>

<h3>Multiple valid sources of state</h3>

<p>The slicer, printer configuration, Moonraker, user interface, and physical tool can all contain information that appears to describe the same filament. They do not necessarily update at the same time. A slicer may describe what should be loaded, while a sensor only confirms that some filament is present. A mapping table may redirect a logical tool after the G-code has already been generated. The system therefore has to preserve the distinction between planned state and physical state instead of collapsing them into one field.</p>

<h3>Different host environments</h3>

<p>RatOS and standard Klipper installations can expose similar user-facing operations through different macro structures. Replacing those implementations with a common macro would make the plugin easier to describe but could remove behavior supplied by the existing environment. The installer instead detects the environment and selects the corresponding integration. This makes installation more complicated but reduces the number of printer-specific behaviors the plugin has to reimplement.</p>

<h3>Frontend distribution</h3>

<p>Mainsail customization initially creates a maintenance problem because a frontend patch has to follow upstream Mainsail releases. Building the frontend directly on every printer also adds Node.js, npm, Vite, TypeScript, source caches, and compilation time to machines whose primary job is running a printer. The current release process moves that work to GitHub Actions. A new frontend archive is published only after the patch applies and upstream linting, unit tests, TypeScript checks, production compilation, installer tests, archive validation, and checksums complete successfully.</p>

<h3>Updates on an active machine</h3>

<p>An ordinary desktop application can update whenever the user closes and reopens it. A printer host can be in the middle of a long print, heating a tool, or maintaining state used by a toolchanger. The update mechanism therefore has operational constraints. Automatic checks use startup delay and jitter and install at most one interface update in a run. The updater requires Klipper to be ready, the print state to be idle, complete, or cancelled, the idle-timeout state to be Idle, heater targets to be zero, and Moonraker's updater to be idle. Dirty, modified, unknown, failed, anomalous, or downgrade candidates are skipped rather than forced.</p>

<p>This was an important finding from the project: software update behavior is part of machine reliability. The update can be technically correct and still be inappropriate if it restarts a control interface while the machine is in an active process.</p>

<h2>Preserving existing configuration</h2>

<p>The installer is designed around an existing configured printer rather than an empty reference installation. It preserves <code>config.json</code>, data below Klipper's <code>SAVE_CONFIG</code> boundary, RatOS-generated files, PID values, and the configured physical-tool count. It also avoids turning the Klipper checkout into a container for unrelated frontend source and caches.</p>

<p>This approach came from a recurring systems issue: printer configuration accumulates calibration and machine-specific behavior over time. Treating the configuration as disposable makes installation scripts simpler but makes the printer harder to maintain. The plugin instead attempts to isolate the changes that belong to the filament integration and leave unrelated state alone.</p>

<h2>Validation</h2>

<p>The release process validates both the integration code and the upstream frontend it modifies. Release 2.0.2 passed the project's installer and updater test suite as well as the upstream Mainsail unit tests used by that release, together with linting, TypeScript checks, a production build, archive-integrity checks, and checksum validation. The purpose of those checks is primarily compatibility: a frontend release should not be published merely because a patch command returned successfully.</p>

<p>Validation is still different from proving every printer configuration. The plugin can verify installation behavior and known interface paths, but a toolchanger may have custom macros, sensor wiring, or local modifications that are outside the test matrix. For that reason the installer includes a verification script and avoids silently rewriting unrecognized configuration.</p>

<h2>Current findings</h2>

<ul>
<li>Filament metadata is most useful when it is attached to physical tools and exposed through a shared host-side state rather than stored only in one user interface.</li>
<li>Logical tool mapping requires filament identity and G-code tool identity to remain separate concepts.</li>
<li>Single-tool and multi-tool printers should not be forced through the same interface simply for implementation uniformity.</li>
<li>Saved processing temperatures are useful as inputs to load and unload routines, but they should remain distinct from commands that energize heaters.</li>
<li>Supporting an existing printer distribution is often better handled by preserving its native behavior and integrating around it rather than replacing it.</li>
<li>Frontend build and update strategy affects machine reliability even though it is not part of the motion-control firmware.</li>
</ul>

<h2>Limitations and current status</h2>

<p>The system depends on the surrounding Klipper, Moonraker, Mainsail, and optional KlipperScreen interfaces remaining compatible with the integration points used by the plugin. A future upstream frontend release can require a new patch, which is why incompatible Mainsail releases are not automatically published. The integration also cannot infer material identity from a basic presence switch; physical metadata still has to originate from a user, slicer workflow, or another source capable of identifying the filament.</p>

<p>Tool mapping is intentionally not equivalent to automatic material management. The software can record that physical tool 3 contains PLA and map logical T0 to that tool, but the decision to perform a remap during a print has to satisfy the mechanical and process constraints of the machine. Material, color, temperature, nozzle geometry, and tool availability can all matter.</p>

<p>The project remains a host-side integration rather than a replacement for Klipper's core machine state. That boundary is deliberate. Klipper continues to control motion and heaters; Moonraker and the interfaces coordinate metadata and user workflow around that machine state.</p>

<p><a href="https://github.com/maevebaksa/Klipper-Filament-Sync">Source code, releases, and installation documentation</a></p>
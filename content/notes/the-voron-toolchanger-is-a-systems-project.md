---
title: "Voron 2.4 Toolchanger Integration"
date: 2026-09-22
description: "Technical notes on integrating a four-tool Voron 2.4, including physical and logical tool identity, homing and probing, filament sensing, per-tool behavior, failover, and migration between toolchanger software stacks."
tags: ["toolchanging", "klipper", "systems"]
categories: ["systems"]
aliases:
  - "/2026/09/22/the-voron-toolchanger-is-a-systems-project/"
---
<p>This project is a four-tool Voron 2.4 configured as a tool-changing printer. The mechanical tool docking is only one part of the system. Reliable operation also depends on homing, probing, toolboard electronics, filament sensing, logical-to-physical tool mapping, per-tool extrusion settings, user-interface state, heater behavior, fan monitoring, and print-start logic remaining consistent with one another.</p>

<p>The machine has also been used as a development platform for several related software projects, including filament metadata synchronization and mapped-tool workflows. As a result, the configuration is less useful as a list of macros than as an example of what changes when a printer stops having a one-to-one relationship between “the extruder,” “the tool requested by the slicer,” and “the physical tool currently mounted.”</p>

<h2>Physical architecture</h2>

<p>The printer uses four physical tools, identified separately from the logical T numbers that appear in sliced G-code. Each tool has its own toolhead electronics and can carry tool-specific calibration and extrusion behavior. The machine uses Tap-style probing on the tools, while physical tool 0 serves as the reference tool for operations that need a known mechanical datum.</p>

<p>That distinction is important because a logical T0 request does not necessarily mean physical tool 0. A single-tool job sliced entirely for T0 can be intentionally assigned to another physical tool, for example physical tool 3. The G-code remains unchanged while the mapping layer decides which physical assembly represents the requested logical tool.</p>

<h2>Homing, gantry alignment, and the reference tool</h2>

<p>Homing and probing need a more explicit definition on a toolchanger than on a fixed-tool printer. The machine has to know which physical tool defines the Z relationship used by the rest of the calibration sequence. The current workflow uses physical tool 0 as the reference, independent of any logical mapping that may be active for the print.</p>

<p>The <code>G32</code> sequence homes the printer, selects the physical reference tool, synchronizes motors when required, rehomes XY after that synchronization, performs quad gantry leveling, and then establishes Z again. This sequence separates mechanical calibration operations from slicer-facing tool identity. The printer should not perform a reference probe with a different physical tool merely because logical T0 has been remapped for the upcoming job.</p>

<p>That separation became one of the central architecture rules for the machine: operations tied to physical geometry use physical tool identity, while operations tied to G-code intent use logical tool identity and the mapping table.</p>

<h2>Print-start sequencing</h2>

<p>The print-start path validates that the logical tools required by the job have physical assignments. It then prepares the printer, heats the bed, performs the probing and adaptive-mesh sequence with the physical reference tool, and manages tool temperatures before selecting the mapped logical starting tool.</p>

<p>This sequencing avoids a subtle problem: a tool used for probing can retain a heater target that is no longer appropriate once the printer moves on to a different physical tool. The reference tool is therefore explicitly shut down after its probing role when it is not the mapped printing tool. Mapped tools can be preheated according to the job so that thermal preparation follows logical print requirements without confusing the physical reference sequence.</p>

<h2>Logical-to-physical tool mapping</h2>

<p>Tool mapping adds flexibility but also introduces a second layer of state that every related subsystem has to respect. The slicer emits T0, T1, and other logical tool commands. The printer maintains a mapping from those numbers to the physical tools that are currently loaded and available. A physical tool can therefore be selected for maintenance or reference operations without changing the logical mapping used by the print.</p>

<p>This enables workflows such as using physical tool 3 for a single-material job still sliced as T0. It also makes automatic replacement possible in principle: when a physical tool becomes unavailable, the logical tool can be reassigned to another physical tool that satisfies the material requirements without modifying the G-code file.</p>

<p>The mapping layer also made it clear that material metadata has to follow the physical tool. If the software reports that logical T0 contains red PLA while T0 has just been remapped from physical tool 0 to physical tool 3, the information can become misleading. The filament-sync work therefore treats physical-tool material and logical-tool assignment as related but separate states.</p>

<h2>Unexpected state-management challenges</h2>

<p>The most difficult issues in a toolchanger tend to occur at the boundaries between otherwise-working subsystems.</p>

<h3>A successful mechanical change is not a complete tool change</h3>

<p>The carriage can physically release one tool and pick up another while the rest of the printer still has stale assumptions. Heater targets, pressure advance, fan state, filament metadata, sensor callbacks, and logical mapping all need to reflect the newly active physical tool. The machine therefore treats tool selection as a state transition rather than only a docking motion.</p>

<h3>Reference geometry and logical mapping can conflict</h3>

<p>Logical mapping is intentionally flexible, while probing geometry should be stable. If a mapped logical tool were allowed to redefine which physical tool performs every reference operation, calibration could change simply because a material assignment changed. Explicit physical reference operations prevent that coupling.</p>

<h3>Homing behavior depends on the complete motion system</h3>

<p>The printer uses sensorless homing on its XY motion system. During migration between toolchanger software stacks, homing had to be restored as a complete behavior rather than copied as a single sensitivity value. Current, rebound distance, coupled-motor behavior, and the order in which the motors are synchronized all affect whether a homing event is repeatable.</p>

<p>This is a general toolchanger issue because the additional carriage mass, docks, and service loops change the physical conditions under which the gantry reaches an endstop or detects a stall. A homing setting that is valid for one motion state cannot automatically be assumed to describe every reconfigured toolchanger.</p>

<h3>Per-tool extrusion settings must follow the physical extruder</h3>

<p>Pressure advance and related extrusion parameters belong to the physical filament path. They cannot safely be treated as a single global value when tools have different extruders, hotends, nozzles, or materials. The tool-change workflow therefore has to preserve per-tool pressure-advance behavior even when logical tools are remapped.</p>

<p>This created compatibility work around existing Mainsail pressure-advance tooling. A wrapper is used so that normal <code>SET_PRESSURE_ADVANCE</code> behavior continues to cooperate with the installed interface rather than bypassing it. The broader finding is that a replacement macro can be syntactically correct while still breaking surrounding tooling that expects to observe or intercept a standard command.</p>

<h2>Filament sensing and failover</h2>

<p>Each tool can have physical filament sensing, including runout and button or clog-related inputs. These sensors report conditions on a physical tool. The response, however, can involve a logical tool that is currently mapped to that hardware.</p>

<p>Automatic failover therefore has two separate questions: which physical tool has become unavailable, and which replacement physical tool can represent the affected logical tool. Replacement selection can consider saved material and color metadata instead of simply choosing the next tool number. A manual replacement path is also useful because material compatibility cannot always be determined from a small set of metadata fields.</p>

<p>The single-tool-print case exposed an especially useful edge condition. A job can contain only logical T0 while T0 is mapped to physical tool 3. If physical tool 3 runs out, the failover mechanism must remap logical T0 to a replacement physical tool. Changing to logical T1 would be incorrect because the G-code was never sliced to use T1.</p>

<h2>Toolboard and sensor integration</h2>

<p>Tool-changing printers tend to move more electronics onto the toolheads. That reduces carriage wiring but increases the number of distributed controllers, sensor pins, bus connections, and firmware identities that must be tracked. The current machine has gone through more than one toolboard and hub arrangement during development.</p>

<p>One result is that pin names should be treated as properties of a specific physical board revision and wiring path rather than universal facts about a sensor model. Pull-up behavior, active-low switches, event delays, and button inputs all have to be validated in the context of the actual toolboard. The same Orbiter-style sensor can therefore require different configuration when moved to a different controller even though the sensor itself has not changed.</p>

<h2>Thermal and fan safety</h2>

<p>A toolchanger has more opportunities for a tool to be hot while it is not mounted on the carriage. It also has multiple heatsink fans and more thermal state than a single-tool machine. The configuration therefore includes fan-stall monitoring and explicit handling of heater targets during tool changes and print preparation.</p>

<p>This was another case where adding tools changes the meaning of a familiar subsystem. A failed heatsink fan on an inactive cold tool is different from the same failure on a hot tool that has just been parked. Safety logic has to consider tool temperature and operational state rather than treating every fan tachometer transition identically.</p>

<h2>Migration between toolchanger software stacks</h2>

<p>The machine was migrated from an earlier DraftShift-based configuration to Klipper Toolchanger Easy. The goal was not to rebuild the printer from a blank configuration but to preserve verified machine behavior while changing the software responsible for tool-changing coordination.</p>

<p>That required restoring sensorless homing, tool mapping, filament-management callbacks, print-start and print-end behavior, LEDs, fan-stall handling, and other local integrations. Compatibility problems were often caused by differences in internal assumptions rather than by the mechanical toolchanger itself. For example, older Moonraker behavior could affect a single-tool mapping path even though the same mapping logic worked on a multi-tool print.</p>

<p>The migration reinforced a design preference used elsewhere on the printer: rely on native Klipper behavior where possible, then small transparent macros, then Moonraker integration, and only use larger third-party Python extensions where the required feature genuinely cannot be expressed at a more stable layer. Extensions that depend on internal Klipper APIs can be effective, but they create additional compatibility work when those interfaces change.</p>

<h2>Slicer and user-interface synchronization</h2>

<p>Tool mapping and material assignment are easier to operate when the slicer, Mainsail, KlipperScreen, and printer do not each maintain unrelated copies of the same information. A host-side bridge can synchronize slicer filament data into Moonraker lane data. The Mainsail and KlipperScreen interfaces can then display the physical filament information used by the mapping and failover logic.</p>

<p>This does not make the slicer authoritative for physical state. It provides planned material information that can be compared with or applied to the machine. The actual mapping remains a printer-side decision because a physical tool can be reloaded or reassigned after the G-code has been generated.</p>

<h2>Current findings</h2>

<ul>
<li>Physical tool identity and logical G-code tool identity need to remain separate throughout homing, probing, mapping, material tracking, and failover.</li>
<li>A fixed physical reference tool makes geometric calibration independent of logical material assignments.</li>
<li>Tool changes should be modeled as state transitions that include heaters, fans, sensors, extrusion settings, and interface state, not only docking motion.</li>
<li>Per-tool extrusion settings need to follow the physical filament path even when the logical tool assignment changes.</li>
<li>Filament-runout replacement should remap the affected logical tool rather than invent a new logical tool that is not present in the sliced job.</li>
<li>Distributed toolhead electronics make board revision, pin behavior, and bus identity part of the machine configuration.</li>
<li>Migration is more reliable when known-working machine behavior is preserved and each integration is reintroduced separately.</li>
</ul>

<h2>Limitations and ongoing work</h2>

<p>The toolchanger remains a mechanically and electronically complex machine, and not every failure can be resolved through mapping or host software. Dock alignment, connector reliability, filament path behavior, thermal expansion, and tool-specific calibration remain physical constraints. Automatic failover also depends on having a genuinely compatible replacement tool; metadata matching cannot establish nozzle condition or every material property.</p>

<p>The configuration continues to change as the printer hardware and software stack change. For that reason, the project is better represented as an integration architecture and a set of tested workflows than as one frozen printer.cfg intended for direct copying to another machine.</p>
---
title: "KlipperScreen Gamepad Jogging and Motion Interlocks"
date: 2026-09-22
description: "Technical notes on adding physical gamepad jogging to a KlipperScreen control station, including printer-state interlocks, network latency, stale input, remote access, and validation limits."
tags: ["klipper", "controls", "interface"]
categories: ["controls"]
aliases:
  - "/2026/09/22/building-a-better-control-surface-for-klipper/"
---
<p>This project adds physical jogging controls to a KlipperScreen-based printer interface without replacing KlipperScreen's normal touchscreen controls. The system runs on a Raspberry Pi 400 and combines a multi-printer dashboard, local Moonraker discovery, saved printer connections, KlipperScreen, and a configurable USB or Bluetooth HID gamepad.</p>

<p>The primary engineering problem is not reading joystick axes. Linux and SDL already provide that information. The difficult part is deciding when an input is allowed to become machine motion, how that decision changes when the selected printer or network connection changes, and what can and cannot be guaranteed after a motion request has already left the control station.</p>

<h2>System architecture</h2>

<p>The control station does not run Klipper or Moonraker locally. It connects to one or more printers that already expose Moonraker over the network. A private pinned KlipperScreen checkout provides the standard printer-control panels. A separate launcher and adapter layer adds the dashboard, connection management, gamepad handling, and printer switching without directly modifying the upstream KlipperScreen source tree.</p>

<p>Printer profiles are stored locally in an owner-readable configuration file. A profile can contain a local Moonraker URL, authentication information when required, and optional remote-access information. The generated KlipperScreen configuration is derived from these profiles so that the user does not have to maintain a second hand-edited list of printer connections.</p>

<p>Local discovery uses Moonraker service advertisement when available. A fallback network scan can probe common Moonraker and reverse-proxy ports on local IPv4 ranges. Manual URLs remain available because multicast discovery does not cross every network, printers can run on nonstandard ports, and some reverse proxies expose Moonraker under a path rather than directly on port 7125.</p>

<h2>Gamepad input model</h2>

<p>The gamepad is not treated as a free-running motion source. The operator first maps a hold-to-jog enable button, chooses which analog axes correspond to X, Y, and Z, and can assign additional digital buttons to jogging directions or selected KlipperScreen actions. Axis inversion and deadzone are configurable because SDL axis numbering and sign conventions are not uniform across controllers.</p>

<p>Jog distance comes from the currently selected move distance in KlipperScreen. XY and Z speed limits also come from the corresponding KlipperScreen Move panel. This avoids creating a second independent set of motion distances and feed limits in the gamepad subsystem.</p>

<p>For local connections, continuous held input is represented as a sequence of bounded moves. Speed starts at roughly 35 percent of the configured cap and ramps toward the selected limit over approximately 1.5 seconds while the same direction remains active. Changing direction, returning the stick to center, or releasing the enable control resets that ramp.</p>

<h2>Motion-state interlocks</h2>

<p>Every move request is preceded by a fresh state check. Motion is blocked when the selected printer is printing, paused, unhomed, disconnected, or otherwise not ready. The requested axis and magnitude are checked for validity before G-code is generated. Only one motion request is allowed to remain in flight, and <code>M400</code> is used so the command path has an explicit synchronization point.</p>

<p>The motion command is wrapped with Klipper's G-code state save and restore behavior. This matters because an interactive jog control should not unexpectedly leave the printer in a different coordinate mode or feed state from the one established by the existing UI or macros.</p>

<p>Jogging is also coupled to interface state. Switching printers, leaving the Move page, opening a dialog, losing application focus, disconnecting the controller, or losing the printer connection disarms motion. A new centered and released input sequence is required before the system will accept another jog. The intent is to prevent an input that was valid in one UI context from being carried into another context after the operator's attention has moved elsewhere.</p>

<h2>Unexpected challenges from asynchronous state</h2>

<p>The largest unforeseen constraints came from the fact that gamepad input, user-interface state, network communication, and printer motion occur on different timelines.</p>

<h3>A button release cannot recall a network request</h3>

<p>A physical joystick feels continuous, so it is natural to expect the machine to stop at the exact instant the stick returns to center. That expectation is not valid for a networked G-code interface. Once a bounded move has been transmitted to Moonraker and accepted by Klipper, releasing the gamepad only prevents the next move from being sent. It does not retract the request already queued on the printer.</p>

<p>This is why the implementation does not describe the gamepad as a hardware dead-man control. The distinction is important. The system can bound requests, avoid queuing multiple outstanding moves, and disarm future motion, but the end-to-end path still includes operating-system scheduling, network latency, Moonraker, Klipper's command queue, and the physical machine.</p>

<h3>Stale callbacks after printer switching</h3>

<p>A multi-printer interface adds another asynchronous case. A status request can be sent to printer A and still be in flight when the user selects printer B. If the response from A is accepted without checking its origin, a late callback can update the current interface or authorize a move against the wrong selected-printer context.</p>

<p>The adapter therefore rejects callbacks associated with a previously selected printer. Automated tests also cover printer changes that occur while a motion-state query is pending. This race condition is not visible in a single-printer implementation and only appears once connection switching becomes part of the interface.</p>

<h3>Input can become stale before a state query returns</h3>

<p>The gamepad state that caused a printer-status query may no longer be true when the query returns. A user can release the enable button, center the stick, switch printers, or move to a different page during that interval. The motion path therefore has to re-check whether the initiating input and UI context are still current before sending a command. Tests explicitly cover release during a status query and other changes that occur while the request is pending.</p>

<h2>Local and remote behavior</h2>

<p>The control station can use an OctoEverywhere App Connection as a fallback when the normal LAN Moonraker endpoint cannot be reached. The local connection remains primary. When an initialized session loses its active transport, the software re-evaluates local reachability and rebuilds the connection on the available endpoint rather than assuming the original path remains valid.</p>

<p>Remote jogging intentionally behaves differently from local jogging. Instead of trying to reproduce a continuous held-stick stream over an Internet path, the remote mode sends a discrete selected-size step and requires the control to return to center or release before another remote step is accepted. This reduces the number of pending requests and makes each remote action easier to reason about under variable network delay.</p>

<p>Remote access also introduced authentication behavior beyond the main Moonraker WebSocket and HTTP control path. Media such as webcam streams may use different requests that still need the App Connection authorization header. A connection that is sufficient for status and G-code control is therefore not automatically proof that every auxiliary media path is correctly authenticated.</p>

<h2>Controller variability</h2>

<p>Another practical constraint is that a “gamepad” is not one standardized logical layout at the SDL layer. Axis numbers can differ between devices. A right stick may appear on different axis indices, triggers can be represented as axes, signs can be inverted, and a D-pad may be exposed as a hat rather than a group of digital buttons.</p>

<p>The interface therefore includes a learning workflow rather than assuming a fixed controller map. Analog axes can be inspected live, individual directions can be inverted, and an axis can be disabled when movement is intended to come from mapped digital buttons. Hat-only D-pads remain a documented limitation because not every such direction is represented as a normal SDL button in the current input layer.</p>

<h2>Display and service integration</h2>

<p>The Raspberry Pi 400 is also responsible for the touchscreen user interface. That means Linux display configuration, the display server, touch input, power management, and the KlipperScreen process are part of the complete control system even though none of them move a printer directly.</p>

<p>The installer deliberately avoids overwriting display overlays or Wi-Fi configuration. It also detects conflicting display managers or KlipperScreen services instead of silently stopping them. This creates a stricter installation requirement, but it prevents the application from assuming ownership of a display stack that may already be used by another interface.</p>

<p>This became relevant during hardware bring-up because a control application can be functioning correctly while the display path itself is not. Treating the UI, X/display service, HDMI output, touch interface, and application process as separate layers makes it possible to isolate a black-screen condition from a printer-connection or gamepad problem.</p>

<h2>High-consequence commands</h2>

<p>The controller supports shortcuts beyond jogging, including printer navigation, pause and resume, cancel, homing, heater shutdown, emergency stop, and named custom macros. These actions do not all receive the same interaction model.</p>

<p>Resume, cancel, home, heater shutdown, and custom macro execution require touchscreen or keyboard confirmation that names the selected printer. Emergency stop remains immediate. This split is intentional: some commands benefit from a confirmation step because the consequence of acting on the wrong selected printer is greater than the cost of an extra interaction. Emergency stop has the opposite requirement.</p>

<h2>Validation</h2>

<p>Automated tests cover the non-GTK motion, connection, privacy, gamepad, and OctoEverywhere helper code. Covered cases include blocked motion for printing, paused, unhomed, non-ready, or missing printer state; axis bounds; non-finite input; Move-panel step propagation; speed ramping; parser-state restoration; held-button startup; deflected-stick startup; release during status queries; printer switching during a query; loss of focus; stale input; timeout without automatic retry; and remote center-to-repeat behavior.</p>

<p>Connection tests cover URL normalization, reverse-proxy paths, IPv6 handling, API-key headers, local Moonraker probing, owner-only profile permissions, and rejection of login redirects. OctoEverywhere parsing, credential redaction, and LAN-first endpoint selection are also tested in software.</p>

<p>The actual GTK dashboard and configuration panels have been rendered against the pinned KlipperScreen checkout, but the project still distinguishes that from hardware acceptance. On-device tests are required with the intended Pi 400, touchscreen, physical controller, real printers, and an authenticated remote connection before the system can be treated as fully validated hardware.</p>

<h2>Current findings</h2>

<ul>
<li>Physical jogging can reuse KlipperScreen's existing movement distance and speed settings rather than maintaining a competing motion configuration.</li>
<li>Network control cannot provide the same stop semantics as a hardwired dead-man circuit because accepted moves may complete after input is released.</li>
<li>Multi-printer interfaces must protect against stale asynchronous responses after the selected printer changes.</li>
<li>Input validity has to be checked both before and after network state queries because the user's control state can change while a request is in flight.</li>
<li>Remote motion is more predictable when represented as discrete acknowledged steps instead of attempting to emulate continuous local input.</li>
<li>Controller mapping needs to be discoverable at runtime because SDL layouts vary across hardware.</li>
</ul>

<h2>Limitations and current status</h2>

<p>The current version does not provide a hardware safety channel and should not be treated as one. It does not enable jogging while a print is paused, does not expose extrusion axes through the stick, does not perform stick-driven homing, and does not automatically change tools. A queued move can finish after a network interruption or release. Simultaneous movement commands from another user interface are outside the local controller's ability to prevent.</p>

<p>OctoEverywhere App Connection setup also depends on the external authorization flow and an application ID assigned for production use. The remote path is therefore intentionally optional; local Moonraker access remains the primary operating mode.</p>

<p><a href="https://github.com/maevebaksa/Klipper-Dashboard-Jogger">Source code and validation notes</a></p>
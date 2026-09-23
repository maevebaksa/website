---
title: "Multi-Printer Farm Management and Job Dispatch"
date: 2026-09-22
description: "Technical notes on a self-hosted mixed-printer farm manager, including protocol normalization, job scheduling, operator confirmation, authentication, file transfer, and deployment constraints."
tags: ["automation", "software", "additive-manufacturing"]
categories: ["automation"]
aliases:
  - "/2026/09/22/from-one-printer-to-a-print-farm-automation-as-a-design-problem/"
---
<p>Managing one networked 3D printer is mostly a printer-control problem. Managing a mixed fleet adds a second problem: the printers do not expose the same concepts through the same protocols. A farm manager therefore has to represent common operational state without erasing differences that matter for file transfer, authentication, job start behavior, material selection, or error handling.</p>

<p>The system documented here is a self-hosted web application for centralized status monitoring and job dispatch across several printer ecosystems. The current codebase includes support for PrusaLink, Klipper/Moonraker, OctoPrint, Bambu Lab, and Elegoo Centauri systems. It runs as a local service with a React frontend, an Express backend, and SQLite persistence.</p>

<h2>System goals</h2>

<p>The basic workflow is to define projects and parts, associate printable G-code with those parts, monitor the state of available printers, and dispatch work to compatible idle machines. A fleet view shows printer status, progress, and remaining time. A TV-style dashboard provides a simplified summary for a shared display.</p>

<p>The application also keeps job history, supports adding many printers through CSV import, and provides backup and restore of farm configuration and history. The deployment is intended for a trusted local network or access through a private VPN rather than direct exposure to the public Internet.</p>

<h2>A common model over different protocols</h2>

<p>Each printer family has its own control protocol. PrusaLink uses a REST interface. Klipper printers are generally managed through Moonraker. OctoPrint provides its own REST API. Bambu systems use MQTT for state and control together with file-transfer mechanisms. Elegoo Centauri models use different network interfaces depending on generation, including SDCP WebSocket communication and MQTT-based control.</p>

<p>The backend therefore uses protocol-specific drivers behind a shared farm model. The scheduler should not have to know the syntax of a Bambu MQTT message or a Moonraker upload request. It needs a narrower set of operations such as “is this printer idle?”, “can it accept this job?”, “upload this file,” and “start the prepared job.”</p>

<p>That abstraction has limits. A multi-brand manager should not claim that all machines are interchangeable merely because they can all print G-code. Some printers expose material slots, some have remote cameras, some provide richer progress data, and authentication requirements differ. The common model therefore has to represent the intersection of useful farm operations while still allowing driver-specific behavior where the hardware requires it.</p>

<h2>Unexpected challenge: printer state is not standardized</h2>

<p>The first significant integration problem is state normalization. Different APIs use different labels and transitions for conditions such as idle, ready, printing, paused, complete, cancelled, offline, and error. Some systems report a print as active during heating. Others distinguish file preparation from printing. Network loss can look different from a printer-side fault.</p>

<p>A scheduler cannot safely dispatch work based only on the literal status string returned by a driver. Each driver therefore has to translate platform-specific state into farm-level semantics and retain enough detail for the UI to explain what the printer is actually doing.</p>

<p>This also affects polling. The current fleet view refreshes printer information on an interval rather than maintaining a permanent connection to every possible protocol in exactly the same way. Polling is straightforward to recover after a temporary network failure, but it means state is always a recent sample rather than a continuously synchronized truth. Scheduling decisions therefore need to be made conservatively around transitions.</p>

<h2>Job dispatch and operator confirmation</h2>

<p>Automatic dispatch can remove the repetitive work of manually choosing an idle printer and transferring a file. It should not automatically assume that a printer that just finished one job is physically ready for the next one.</p>

<p>The current workflow includes an operator-confirmation step after a completed print. A printer does not immediately receive another scheduled job until someone confirms the result and the machine is ready to continue. This is intentionally less autonomous than a system that continuously fills every idle machine.</p>

<p>The confirmation boundary exists because several important conditions are not reliably visible through a printer API: whether the previous part was removed, whether the bed needs cleaning, whether a failed part is still attached to the nozzle, whether a spool has enough material, or whether the machine needs maintenance. Automation is used for the repeatable digital handoff, while physical readiness remains an explicit state transition.</p>

<h2>Scheduler concurrency</h2>

<p>A scheduler also has to avoid assigning the same job or printer twice when several state updates happen close together. This is less visible in a small fleet but becomes more important as multiple printers become idle during the same polling interval.</p>

<p>SQLite is used as the central persistent store, which keeps project, printer, and job state in one local database. The scheduling logic can make assignment decisions against that stored state rather than relying only on what the browser last displayed. This is important because the user interface is a view of the farm, not the authority that determines which job is currently assigned.</p>

<h2>Files are part of the state machine</h2>

<p>Starting a job on a network printer is usually at least two operations: transfer a file and then instruct the machine to print it. The details differ by platform. A failed transfer, a successful transfer followed by a failed start command, or a printer disconnect between those operations all create different recovery conditions.</p>

<p>The farm manager therefore cannot treat “send job” as one indivisible button press. Driver code has to report enough information for the backend to determine whether the file exists on the printer, whether a start was acknowledged, and whether the job should return to the queue or require operator review.</p>

<p>Direct-to-printer workflows add another edge case. A file can be sent from an external slicer without passing through the farm scheduler. For supported printer detail views, the application can record or catalog such a print so that job history is not limited to work originally dispatched from the farm interface.</p>

<h2>Material and accessory differences</h2>

<p>Some printer ecosystems expose additional configuration such as Bambu AMS slot selection. Others rely on the printer's currently loaded material or on host-side macros. A generic job model therefore needs to avoid assuming that every machine can accept the same material-selection instruction.</p>

<p>This is an example of a broader constraint in multi-brand software: common operations should be normalized, but features that are genuinely platform-specific should remain platform-specific. A compatibility layer is useful only while it preserves the information needed to operate each machine correctly.</p>

<h2>Authentication and local deployment</h2>

<p>The application supports accounts, sessions, administrative and operator roles, API keys for scripts or slicers, and optional OpenID Connect single sign-on. This was necessary because a farm-control interface is not simply a dashboard. It can upload files, start printers, alter the job queue, and store credentials for the printer protocols it manages.</p>

<p>At the same time, the current security model is designed around a trusted local network. Printer credentials and G-code are stored within the same application environment that an authorized farm account can reach. The service should therefore sit behind a router firewall or private VPN rather than being exposed directly to the Internet simply because it has a login page.</p>

<p>This distinction became an important deployment finding. Authentication controls who can use the application; it does not automatically make every internal printer protocol safe for public exposure. Several printer APIs were designed for LAN use and may rely on long-lived local credentials.</p>

<h2>API keys and automation clients</h2>

<p>Human login sessions are not convenient for slicers, scripts, or unattended integrations. The farm manager therefore supports long-lived API keys that can be created, copied once, and later revoked. Keys are handled separately from user passwords so an automation client does not need to store a person's interactive login credentials.</p>

<p>This is another example of a feature that appears peripheral until the system is used in practice. A print-farm manager sits between automated clients and physical machines. If every external integration shares the administrator password, there is no clean way to revoke one client without changing access for every other client.</p>

<h2>Deployment and persistence</h2>

<p>The application can run directly under Node.js or in Docker. A production container stores the database and uploaded G-code in persistent volumes so the runtime container can be replaced during an update without losing farm state. Multi-architecture images are used so the same deployment model can run on common x86-64 or ARM64 hardware.</p>

<p>Containerization solves packaging problems but not operational ones. The application still needs a stable local address, persistent storage, backups, and access to the printers' network segments. When deployed through Portainer or another container manager, the surrounding host remains part of the farm infrastructure.</p>

<p>The built-in backup and restore path exports farm configuration and job history so the database is not the only practical recovery mechanism. This matters because a farm manager accumulates operational state over time; reconstructing dozens of printer addresses, credentials, models, and job records manually would be a significant recovery task.</p>

<h2>Adding a large fleet</h2>

<p>Manual entry becomes inefficient when many printers are added at once. CSV import provides a structured path for names, network addresses, platform type, model, grouping, and platform-specific credentials such as API keys, access codes, or serial numbers.</p>

<p>The import process also illustrates why validation matters. A printer name can suggest a model, but inferred metadata should not silently become authoritative when the name is ambiguous. Unrecognized models require explicit selection rather than being forced into the closest known type.</p>

<h2>Unexpected protocol differences</h2>

<p>Several challenges only become visible after more than one vendor is connected. File upload can use HTTP multipart requests, FTPS, or vendor-specific methods. Live state can arrive through REST polling, WebSockets, or MQTT. Some systems expose a serial number as part of authentication while others use an API key. A printer can be reachable for status but fail during file transfer because those operations use different network services.</p>

<p>These differences make connectivity a multi-stage property. “Printer online” is not sufficient to prove that the farm can upload and start a job. Driver diagnostics need to distinguish status reachability, authentication, file transfer, and command execution.</p>

<h2>Relationship to earlier printer automation</h2>

<p>The farm-management work follows earlier printer-automation projects such as automatic bed clearing and belt-printer workflows. Those systems focused on making one machine capable of continuing production with less intervention. The fleet manager addresses a different level of the problem: deciding which machine should receive work and maintaining a consistent operational record across several printer types.</p>

<p>The earlier projects also show why complete autonomy is not always the correct objective. A belt printer can physically move a completed part away, while a fixed-bed printer may still require removal and inspection. Farm software therefore benefits from representing physical readiness explicitly instead of assuming that every “complete” state means the same thing mechanically.</p>

<h2>Current findings</h2>

<ul>
<li>A mixed fleet needs a common operational state model, but platform-specific capabilities should remain visible rather than being flattened away.</li>
<li>Printer status is a sampled and translated value; scheduling logic should be conservative around transitions and temporary network failures.</li>
<li>File transfer and print start are separate state transitions and can fail independently.</li>
<li>Operator confirmation after a completed job is a useful boundary between digital automation and physical conditions the printer cannot reliably sense.</li>
<li>Authentication for a control application does not make LAN-oriented printer protocols appropriate for direct Internet exposure.</li>
<li>Long-lived API keys provide a better automation boundary than sharing interactive user credentials with slicers and scripts.</li>
<li>Persistent database state, G-code storage, and backups are part of the production system, not deployment details that can be added later.</li>
</ul>

<h2>Limitations and current status</h2>

<p>The system depends on vendor and open-source printer APIs that can change independently. A driver can be correct for one firmware generation and require updates after a vendor changes authentication, MQTT fields, file-transfer behavior, or state labels. Multi-brand support therefore creates ongoing compatibility work.</p>

<p>The application also does not remove the need for physical process controls. It cannot determine every spool condition, inspect every finished part, clear every bed, or make an unsafe printer mechanically safe. Its role is coordination: maintaining state, moving files, dispatching jobs under defined conditions, and presenting the fleet through one operational interface.</p>

<p><a href="https://github.com/maevebaksa/print-farm-manager">Repository and installation documentation</a></p>
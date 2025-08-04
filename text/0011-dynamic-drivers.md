- Feature Name: Dynamic Driver Loading and Implicit Ordering
- Start Date: 2025-07-07
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary
Discussion on how the service monitor could dynamically load drivers.

For clarification:
- In many places "user service monitor" is shortened to "service monitor". 
- The core service monitor is specified if it is referenced. This is still an unresolved question.
- The core and user/optional monitor both can refer to a system services monitor.
- A "per-user" may also be referenced (not the same as just "user service monitor") and refers to a daemon run with a non-root uid.
- TODO: Decide between system/per-user and core/optional architecture, or some other alternative, and clean up this wording. Leaning towards the former. 

Additional details on the bigger idea this is planned to play into [in this repo on GitLab](https://gitlab.redox-os.org/CharlliePhillips/service-monitor-rsoc2025_planning) and the other markdown files in this repo.

# Motivation
[motivation]: #motivation

Some drivers do not need to run until a particular device is attached. 
Since the service monitor can control and monitor services, programs needing a particular service could get to one that isn't running through the service monitor. 
- This RFC is intended to document discussion on functionality that will be implemented after RSoC2025.

# Detailed design
[design]: #detailed-design

__This Section Documents Potential Methods for Dynamic Driver Loading__

## Scheme Pre-Registration
- From Bjorn3:
    - Redox OS may be able to get away with fully implicit ordering. Basically, when loading the service files it would register all schemes that are exposed according to the service files and then pass the scheme fd when starting the executable that is responsible for this scheme. If another process needs this scheme, it will see the scheme, but block until the responsible service is started.  
  
- So when the service monitor is started, it reserves the names for each scheme in its registry and reserves them.
- When any program on the system attempts to open that scheme, the service monitor starts that service.
- The opened service gets its real scheme and completes the original open request.
- Further calls on that service's scheme will be completed by that running service. 
- This means handling of user permissions would be the responsibility of the individual service.
  
- Right now mounting rootfs involves iterating through the root scheme to find all available disks, and those disks won't be listed there if their drivers have not already been started. This necessitates some explicit ordering of services.

## Notification Interface
- Handlers for additional `SYS_CALL`s could be added to the service monitor to notify it of hardware and system changes. 
- This would likely involve communication with `pcid` and `acpid` among other services.
- TODO - Link RFC on ACPI, PCI/core system behavior.

## Hardware Detection Daemon
- Another idea is to create a second daemon for monitoring hardware changes.
- This may combine methods described here and issue start requests to the service monitor.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

This opens a door for a user to give arbitrary input to the service monitor, which runs as root. This risk can be mitigated though.

# Alternatives
[alternatives]: #alternatives

- Adding some other kind of specific hooks/event handlers into the service monitor to determine when certain services should start.

# Unresolved questions
[unresolved]: #unresolved-questions

- triggering service shutdown as well? The API may partially accommodate this for applications that do the same, but what about other system events?
- Are there ways to work around some scenarios where explicit dependencies would be required? There will probably not be a way to avoid explicit ordering entirely.
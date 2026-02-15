- Feature Name: Service and Driver Recovery
- Start Date: 2025-07-07
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary
Discussion on how the service-monitor recovers faulting services.

For clarification:
- In many places "user service monitor" is shortened to "service monitor". 
- The core service monitor is specified if it is referenced. This is still an unresolved question.
- The core, and user/optional monitor both can refer to a system services monitor.
- A "per-user" may also be referenced (not the same as just "user service monitor") and refers to a daemon run with a non-root uid.
- TODO: Decide between system/per-user and core/optional architecture, or some other alternative, and clean up this wording. Leaning towards the former. 

Additional details on the bigger idea this is planned to play into [in this repo on GitLab](https://gitlab.redox-os.org/CharlliePhillips/service-monitor-rsoc2025_planning) and the other markdown files in this repo.

# Motivation
[motivation]: #motivation

One of the benefits of a micro-kernel architecture is the increased isolation of processes. 
This creates the potential to recover from errors in core drivers and services that would be unrecoverable on other systems.
- This RFC is intended to document discussion on functionality that will be implemented after RSoC2025.

# Detailed design
[design]: #detailed-design

- See [the optional services rfc](./0010-optional-services.md?ref_type=heads#failure-detection) for one existing recovery method.

- When a service exits with code 0 then it's been stopped properly. How could other exit codes be used to determine how a service should be recovered?

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

# Unresolved questions
[unresolved]: #unresolved-questions

- NEED Other service recovery ideas

If a core daemon fails, then what should the user service monitor do?
- Maybe if the core service monitor can successfully recover that daemon, the user monitor is restarted?
- Maybe the service-monitor could attempt to gracefully shutdown running services (and potentially other applications later i.e. save your work before a crash)?
- The core monitor could send a message to the user monitor telling it what failed, and those service names could be included in the depends list? This could allow "less destructive" recovery preserving as much of the pre-failure environment as possible.
- Combining the last two points, although maybe most of this is stretch goals or future work?
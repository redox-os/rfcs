- Feature Name: Optional/User Service Lifecycle
- Start Date: 2025-07-07
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary
The service monitor aims to fill existing gaps in the way that Redox handles services, daemons, and drivers. This RFC discusses the general lifecycle and handling for these services. The service-monitor is highly configurable and intended to be used with a variety of system configurations. Some core system services and drivers necessitate further discussion but for most drivers and daemons the following completely outlines their interaction with the service monitor.  

This RFC covers the behavior of the services monitor and its constituent services 
specifically:
1. [Service Startup](./0010-optional-services.md?ref_type=heads#service-startup)
2. [Service Shutdown](./0010-optional-services.md?ref_type=heads#service-shutdown)
3. [Failure Detection and Recovery](./0010-optional-services.md?ref_type=heads#failure-detection-restart-and-recovery)

Additional details on the bigger idea this is planned to play into [in this repo on GitLab](https://gitlab.redox-os.org/CharlliePhillips/service-monitor-rsoc2025_planning) and the other markdown files in this repo.
# Motivation
[motivation]: #motivation

Service monitoring & management systems are extremely important for workstations, servers, appliances, and many other computing environments. Having none or one monolithic service monitor for all system services and drivers could pose issues for the security and stability of Redox.

Use Cases:
- Lingering:
    - A user can configure a service to keep running after they logout. One use case of this would for a user to keep their web server running while away from the system.
- Parallel System Initialization:
    - The service monitor can build a dependency graph and start independent services in separate threads. This could significantly improve Redox's time to boot.
- Service & Driver Recovery:
    - While the service monitor is continuously updating service statistics it can detect when a service is misbehaving and restart or potentially recover that service to minimize the impact on the rest of the system.
 Dynamic Driver Loading:
    - Hot-Swap devices don't need their drivers running if they are not connected. Other system components can communicate to the service monitor that a newly attached device needs a driver loaded.
    - It is not always desired to have drivers running for attached/integrated devices either. I.e. unloading WiFi/Bluetooth drivers when turning off wireless communications (airplane mode).
- Easier Driver & Service Management:
    - Organizing per-service configuration into individual files instead of combining them into init scripts will make it easier to implement partial service overriding. Overriding is helpful for making modifications to a service's configuration persist through updates.
    - The service monitor provides standardized methods for interacting with services. This can be utilized by TUI or GUI utilities for system management.
- Debugging & Benchmarking:
    - The service monitor can expose information indicating how much a service is used by the system.
- Graceful System Shutdown:
    - Since the service monitor continues to track services after initialing them, it can also properly prepare them for shutdown.

# Detailed design
[design]: #detailed-design
- The `usermon` library contains a few static functions and enum(s) for interacting with the service monitor defined in this section.
This library may be added to and documented elsewhere as it is implemented.

- The `service-stats` library is also referenced several times in this section and is being worked on [here](./source/service-stats/src/lib.rs). Some functions defined here are not implemented yet. 
- An example of the prototype stats library integrated into the storage `driver-block` [can be found here](https://gitlab.redox-os.org/CharlliePhillips/driver-block/-/blob/master/storage/driver-block/src/lib.rs?ref_type=heads), and with randd [here](./source/randdstat/src/main.rs).

## Service Startup
User services may be started at any time once the core services monitor indicates its ready status to the user services monitor. This means that all user services assume core daemons are functioning properly. During development it will be started with `init.rc` - see unresolved questions
- The `/usr/lib/init.d` and `/etc/init.d` directories contain `*.toml` files and subdirectories with `*.toml` files that are all parsed into a list of services hereby referred to as the "registry".
- When the user service monitor starts, it reads each `*.toml` file in these directories to get the information it needs to start and manage each of it's services.
- These files should each contain one service.
- The `[service]` table contains the following fields: 
    - `name`: String - Used to identify the service
    - `command`: String - as you would type it in the terminal, including arguments
    - `scheme` String - name/path: see unresolved questions
    - `restart_behavior`: String - [see the corresponding subsection](./0010-optional-services.md?ref_type=heads#restart-behavior)
    - `init_after`: String - One or no name of a service that is the one run before this on system startup. This is used to order services during system startup. The first services to be started should name `service-monitor` here.
     - `always_after`: String - Same as `init after` but the named service is always started right before this table's. If a name is specified, then `init_after` is ignored.
    - `depends`: [String] - A list of other user services required for this one to start and run.
    If one of those services is stopped, then this one will be as well.
    The services listed here are not affected by actions on this one. 
    This is a one way relationship like the `partOf` option for systemd units.
- The toml may also contain the following field:
    - `drivers` (optional): The presence of the `drivers` string indicates that the named file or directory should be passed to pcid-spawner to start it. The named file(s) should contain one or more `[[drivers]]` entries:  
    ```toml
    [[drivers]]
    name = String # This is the name of the device/driver
    # The following 6 fields are used to identify the device and some may be omitted depending on the driver.
    class = u8 # This is the type of driver. 1 for storage, 2 for networking, 3 for graphics, etc.
    subclass =  u8 # For storage would be 0 for virtio, 1 for IDE, 8 for NVME, etc.
    interface = u8 # E.g. for xhcid: class = 0x0C (serial), subclass = 3 (USB), and interface = 0x30 (XHCI)
    vendor = u32 # Vendor ID (HW vendor specific)
    device = u16 # Device ID (HW specific)
    ids = {u16 = [u16, ], } # Map of Vendor IDs to Device IDs for a driver that supports multiple devices from the same vendor
    ##
    command = String # Command to start driver daemon
    use_channel = bool # For DMA devices?
    ```
    - These files already exist in `/etc/pcid.d` and `/etc/pcid/initfs.toml`.
    - The pcid-spawner code will be wrapped into the service monitor as it currently exists as a single function in a single file.

- After system startup, the service monitor can start services through an API request:
    1. Call `SYS_CALL` on the service monitor's scheme with the metadata array produced by `usermon::startup()`
    2. The service name is parsed from the payload buffer.
    3. the service monitor then uses the information from the registry to startup the service if it was not already running.
    4. If a service cannot start because it requires others to be running then it will return an error listing the names of the services that need to be running. 
    This list is prepended with the string "depends:", separated by spaces, and overwritten into the payload. 
    5. If a service starts successfully, then the service name(s) started are overwritten into the payload and separated by spaces.

- The service monitor uses the information from the registry to build a`std::process::Command`:
    - This starts the daemon, and once this spawn returns Ok and the child has exited, the service has been demonized and its scheme can be opened, and that fd will be used for recording stats. Eventually Unix Domain Sockets will be used to help secure the socket between services and the service monitor. Once the service monitor has successfully opened a scheme, that service's running state is recorded.
    - If the service is a driver then it's `command` field should be populated with `pcid-spawner <driver(s)>.toml` for now. The drivers in that file will be searched for a matching name to the `[[service]]` entry. 
    - When the `pcid-spawner` is integrated into the service monitor the `command` field in the `[[drivers]]` will be used, and the one in the `[[service]]` will be ignored. 
    - If a registered service is started without the service monitor (i.e. running the `command` manually), then it cannot be started by the service monitor, and it will be treated as a normal process.
    - In the future, services should be able to start more dynamically. For example, when a program attempts to access that service's scheme but it is stopped. This is discussed [in the RFC here](./0011-dynamic-drivers.md).

### Service Monitor Startup
When a service monitor is started it is passed a configuration file in TOML format. The name of this file is used as the name of the service-monitor instance, and its scheme name. Beyond this a service-monitor instance behaves like a standard service so that, for example, one service-monitor could start from the ramfs at boot to then start one from the rootfs, which then starts a user-specific instance upon login. The ramfs monitor would collect data on its services including the rootfs monitor, and the same for the rootfs monitor and a per-user monitor.

The service-monitor configuration file contains the following fields:

## Service Shutdown
1. Service shutdown is triggered by the user service monitor sending the termination signal (`SIGTERM`) to that process.
3. The service that is about to shut down has its last set of statistics recorded.
2. Any running services in the registry that list this one in their  `depends` list get their stats collected, and sent `SIGTERM` first.
4. The service developer should write a handler for `SIGTERM` that gracefully shuts it down. 
6. If the service is still running after 60? seconds it is killed.
7. Once the service monitor has confirmed that the process has stopped by receiving exit code 0, the service's state is recorded as stopped.

### Requesting Shutdown
- To request the user service monitor shut down a service, `SYS_CALL` should be called on its  scheme with the metadata array produced by `usermon::shutdown()`.
- The service name is parsed from the payload buffer. - see unresolved questions
- The service monitor then shuts down the specified service using the information from the registry.
- The return value will be a result containing the number of bytes overwritten to the payload buffer. Those bytes contain a list of names for the services stopped, separated by spaces.

### System Shutdown
When the service monitor receives the system shutdown notification (through acpid, hwd, core monitor?) it will loop through its  running services and shut down each one before notifying the system (acpid...?) that it can proceed with the rest of shutdown. 
The order in which services are shutdown is determined by their `depends` field, so a service cannot be stopped until all the ones that depend on it have shut down.
TODO - Link RFC on ACPI, PCI/core system behavior.

## Monitoring, Failure Detection, Restart, and Recovery
### Monitoring and Data Collection
- The service monitor performs a "heartbeat" check on an interval determined in the configuration for the service monitor. On each heartbeat the service monitor gathers statistics on each daemon. This is done using a `StatMsg` struct serialized to RON, and includes the following fields:


- These stats are stored, and updated by the service monitor for access by other applications.
- A service-monitor instance can also provide it's statistics the same way as the services that it monitors. This is vital for using hierarchical service-monitor instances and is how applications calling a service-monitor instance determine which services it is responsible for.

### Restarting a service
- A service can be restarted which follows the shutdown and then startup procedure on a running service.
- This can be requested through the service monitor's API by calling `SYS_CALL` on its scheme with the metadata produced by `usermon::restart()` and the service's name on the payload buffer.

### Restart Behavior
Each service has a default restart behavior specified in the registry. This can be temporarily overwritten with an optional argument (as in option type?) to `req_shutdown()` or the service monitor API. Each option is an enum that can be converted from and into a string used for the toml. The options for restart behavior are:
- `Restart::Timer(millis: usize)`: The service monitor will always restart this service at least `millis` ms after it has been found stopped. For half a second the restart behavior string would be `timer500`.
- `Restart::Recovery(millis: usize)`: This service will only be restarted if a failure is detected, and it will wait `millis` milliseconds before recovering it (e.g. `recovery500` in the toml). If this is set to 0 it may be possible for a call to the initially failing service to succeed?
- `Restart::Never`: This service won't be restarted automatically once it has stopped. Other applications may call the service monitor's API to restart it. (`never` in the toml)
- To modify a service's restart behavior call `SYS_CALL` on the service monitor's scheme with the metadata array produced by `usermon::behavior(Restart::*)` and the service's name on the payload buffer. - see unresolved questions

### Failure Detection
- When the service monitor preforms a heartbeat check and detects a service that stopped due to some failure its state is recorded as faulted.  
- If the service monitor did not shut down a service, and the restart behavior allows, it will be automatically restarted.  
One other way service faults can be detected and potentially recovered is:
- Each call the service monitor makes to its constituent services is made with a timeout of 50?ms.
    - If the call takes longer than this, and its restart behavior allows, the service will be restarted.
    - If the restart behavior prohibits then the service will just be sent `SIGTERM` (then likely killed).

__Further recovery methods will be outlined in [this RFC](./0012-service-recovery.md).__

## Security Considerations
The service monitor will be configurable so that it may be used as a system services monitor, a per-user services monitor, or for any other use case where a service management system may be needed. The plan for Redox's security system is to move away from more traditional user-based security and implement a capability based permission system. When starting a service, that service will send a statistics, and potentially a control capability. These capabilities should function in some ways like a file descriptor and will be sent using Unix domain sockets. With a statistics capability a service-monitor can only read statistics from a service, and with the control capability a service-monitor can stop a running service and change it's restart behavior.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?
- Maybe the user service monitor should not block system shutdown?

# Alternatives
[alternatives]: #alternatives

- Monolithic service monitor: security issues as mentioned in the motivations section.  
- No monitoring: Monitoring is simpler than control but comes with additional overhead.
- No monitoring or control: former (current) implementation.
- More complex service options like systemd's unit system. 
- shell commands instead of rc script? something bespoke or any others?
- maybe use RON instead of toml to accommodate enums in the registry? The driver files could then be referenced from separate .ron service files.
- The existing init system uses .rc scripts to do things like clean up files and set environment variables. With the system described here, this functionality would be replaced with additional service files.

# Unresolved questions
[unresolved]: #unresolved-questions

- Core/Optional service monitor vs System/per-user service monitor?
- Before implementing the core service monitor and integrating it with low level system services like acpid and hwd, what is the best plan to demonstrate stats collection on all services? Maybe disallowing control over the "core" daemons and open them on startup assuming the preceding system startup has succeeded?
- What about daemons that the service-monitor may depend on but should still monitor? Specifically the process manager, but there may be more.
- Handling async schemes - I do not have much experience with this, but I know it may have implications on the way shutdown and probably other things work.
- how best to uniquely ID services? Name in toml/registry? Scheme name?
- Should restart behavior on reboot persist in the registry after reboot or be temporary? Maybe this could be a boolean parameter?
- Adding an additional registry folder for non-init/per-user daemons?
- is having only one or no name required before attempting to start a service adequate for eliminating dependency loops and accommodating cross dependent daemons?
- is the depends array enough to properly shut services down in order? - No, need to differentiate between initfs & rootfs. 
- How to handle when the payload buffer is too large? Maybe error with the required length of the buffer and preform a separate read for the data? Maybe impose limits on service name length and number of services one can depend on.
- Could the system service monitor binary be used for per-user monitors? Maybe this could be done with launch arguments (I guess this doesn't necessitate the same binary though)? Maybe a per-user service monitor would function too differently?
- Detailed discussion for the per-user service monitor may be more appropriate in the dynamic driver loading RFC? This would be good to keep in mind for development though to maybe start the default user's monitor at boot for security proof of concept.
- Can the service monitor be set to automatically shut down a service once all open fds to that service have been closed through the stats crate? Or would this functionality be too service specific?
- What else is needed to start up a service?
- Should shutdown timeout be a service toml field?
- Get service names from the `service.toml` file for that service instead of an explicit name field?
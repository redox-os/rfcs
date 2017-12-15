- Feature Name: netcfg
- Start Date: 2017-12-16
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

_netcfg_ is a method for configuring and managing network interfaces. It's goals
are to provide an intuitive back-end for doing so. After the back-end is created,
a configuration daemon should be created that provides a user interface, reads
the configuration supplied from the `file` scheme, and uss the utilities detailed
here to push the configuration.

# Motivation
[motivation]: #motivation

The immediate motivations for changing the method of configuring network
interfaces are due to the following weaknesses in the current implementation.

 - Inability to configure multiple interfaces.
 - Handling of interfaces configured to use DHCP is currently non-functional.
 - Inability to change interface IP addresses or the gateway IP.
 - Inability to inspect or manipulate the neighbor cache.

Once IPv6 is supported, the situation is expected to become more complex.

# Initial Requirements
[initial-requirements]: #initial-requirements

The initial requirements of _netcfg_ are the following:

 - Adding, removing, and listing
   - IPv4 addresses
   - The IPv4 gateway
   - ARP cache entries
 - Setting the link up or down
 - Changing the link mac address
 - Ability to listen for a change in one of the above

# Desired architecture
[desired-architecture]: #desired-architecture

```
                  __________      ________________
 ----------      |  Config  |    |    Network     |
| Smolnetd | <-> | Protocol | <- |     Config     |
 ----------       ----------     | Application(s) |
                      ^          |    (dhcpd)     |
                      |           ----------------
                 _______________
                |   Desktop     |
                |     GUI       |
                |  Displaying   |
                |   Interface   |
                | Configuration |
                 ---------------
```

# Detailed design
[design]: #detailed-design

## Design Summary
[design-summary]: #design-summary

Create a new scheme at `netcfg:` that contains endpoints for each of
the options to configure for each interface.

## Structure
[structure]: #structure

The design proposed here is quite simple. Create `netcfg:` which contains
a path for each network interface with various
[interface specific](#interface-specific-endpoints) options and a path
for [`route`]s.

Each of the interface specific paths contains a endpoint for [`addr`] and
[`neigh`] that have a few specified operations that can be used, including
`add`, `rm`, and `list`. Each interface also contains endpoints for the
[`mac`] and [`flags`] of the interface. The [`mac`] endpoint simply must
contain the current hardware address of the interface. The [`flags`]
endpoint contains `list` which returns the flags bitmask and an enpoint
for each flag that may be `0` or `1`.

The [`route`] endpoint is global (not interface specific) and also uses
the `add`, `rm`, and `list` operations.

```
netcfg:
├── route
│   ├── add (WO)
│   ├── rm (WO)
│   └── list (RO)
├── <interface name>
│   ├── mac (RW)
│   ├── flags
│   │   ├── up (RW)
│   │   └── list (RO)
│   ├── addr
│   │   ├── add (WO)
│   │   ├── rm (WO)
│   │   └── list (RO)
│   └── neigh
│       ├── add (WO)
│       ├── rm (WO)
│       └── list (RO)
└── <interface name>
...
```

### Interface specific endpoints
[interface-specific]: #interface-specific-endpoints

#### `mac`
[`mac`]: #mac

A simple `Read-write` endpoint containing the hardware address of the interface.

***Note:*** [smoltcp] currently only supports Ethernet devices.

#### `flags`
[`flags`]: #flags

| Endpoint | Effect                       | Permissions  |
|----------|------------------------------|--------------|
| `up`     | Set to `1` if up `0` if down | `Read-write` |
| `list`   | Current flags                | `Read-only`  |


#### `addr`
[`addr`]: #addr

| Endpoint | Effect                                             |
|----------|----------------------------------------------------|
| `add`    | Adds `<address>/<prefix>` to the interface         |
| `rm`     | Removes `<address>/<prefix>` to the interface      |
| `list`   | Line feed separated list of `<address>/<prefix>`   |

#### `neigh`
[`neigh`]: #neigh

| Endpoint | Effect                                                    |
|----------|-----------------------------------------------------------|
| `add`    | Adds `<address> lladdr <mac>` to the Neighbor cache       |
| `rm`     | Removes `<address> lladdr <mac>` from the Neighbor cache  |
| `list`   | Line feed separated list of `<address> lladdr <mac>`      |

***Note:*** The initial implementation will be IPv4 only. The
neighbor cache will only include the ARP cache.

### Global endpoints
[global-endpoints]: #global-endpoints

#### `route`
[`route`]: #route

| Endpoint | Effect                                                    |
|----------|-----------------------------------------------------------|
| `add`    | Adds `<dest subnet> via <address>` to routes              |
| `rm`     | Removes `<dest subnet> via <address>` from routes         |
| `list`   | Line feed separated list of `<dest subnet> via <address>` |

***Note:*** In the initial implementation this may only include the
IPv4 gateway which is defined as `default via <ipv4 gateway>`.

### Operation permissions
[operation-permissions]: #operation-permissions

| Operation | Permissions  |
|-----------|--------------|
| `add`     | `Write-only` |
| `rm`      | `Write-only` |
| `list`    | `Read-only`  |

## Examples
[examples]: #examples

The simplicity of this design makes it increadibly easy to use. One
can easily add, remove, and monitor the routes, IPv4 addresses, and
neighbors associated with a given network interface using simple
shell utilities.

### `sh` examples
[sh-examples]: #sh-examples

Add the address `192.168.1.2/24` to interface `enp5s0`.

```sh
echo "192.168.1.2/24" > netcfg:/enp5s0/addr/add
```

Add `192.168.1.1` as the ipv4 gateway for interface `enp5s0`.

```sh
echo "default via 192.168.1.1 dev enp5s0" > netcfg:/route/add
```

List all addresses assigned to interface `enp5s0`.

```sh
cat netcfg:/enp5s0/addr/list
```

### `rust` examples

Remove the address `192.168.1.2/24` from interface `enp5s0`.

```rust
let fd = open("netcfg:/enp5s0/addr/rm", O_WRONLY);
let result = write(fd, b"192.168.1.2/24");
```

Iterate through all nodes in the ARP cache for interface `enp5s0`.

```rust
let fd = open("netcfg:/enp5s0/neigh/list", O_RDONLY);
let result = read(fd, &mut buffer[..]);
let data = String::from_utf8(buffer).unwrap();
for entry in data.split('\n') {
    // Do something with the ARP cache entry
}
```


# Drawbacks
[drawbacks]: #drawbacks

 - The addition of IPv6 will make this a bit more complex due to
   the increased complexity of default routes.
 - There is a potential for the number of endpoints under `netcfg` to
   grow to a very large number.

# Alternatives
[alternatives]: #alternatives

Implement a new netlink socket type and scheme endpoint at `netlink:`.
The implementation should loosly adhere to the netlink protocol used by linux.
The linux implementation and RFC should serve as inspiration. In particular,
the initial implementation may only implement `NETLINK_ROUTE` sockets. Relevant
portions of [RFC 3549] include sections [2.3.3], [3.1.1], and [3.1.2].
Additional info can be found in the linux `man` pages for [netlink] and
[rtnetlink].

# Unresolved questions
[unresolved]: #unresolved-questions

 - Would `del` be more intuitive than `rm`?
 - Is the expected increase in readability/usability from using the interface
   name instead of the interface index worth the expected performance impact?

[RFC 3549]: https://tools.ietf.org/html/rfc3549
[2.3.3]: https://tools.ietf.org/html/rfc3549#section-2.3.3
[3.1.1]: https://tools.ietf.org/html/rfc3549#section-3.1.1
[3.1.2]: https://tools.ietf.org/html/rfc3549#section-3.1.2
[netlink]: http://man7.org/linux/man-pages/man7/netlink.7.html
[rtnetlink]: http://man7.org/linux/man-pages/man7/rtnetlink.7.html
[smoltcp]: https://github.com/m-labs/smoltcp

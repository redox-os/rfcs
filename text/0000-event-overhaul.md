- Feature Name: event-overhaul
- Start Date: 2018-05-19
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Overhaul the kernel event system to support mio.

# Motivation
[motivation]: #motivation

The current kernel event system has one event queue per context, which is not
flexible enough to be used by mio.

# Detailed design
[design]: #detailed-design

- Make `fevent` system call return `ENOSYS`. The system call number will still
  be needed to support kernel <> scheme event communication.
- Remove old kernel event system.
- Produce a new kernel event system which can support the example
- Port existing event users to the new system
- Ensure that event generators will always trigger events when added to an
  event queue, and will be edge triggered after that
- Rebuild all packages
- Produce a new major release of Redox OS

```rust
// This is a psuedo-Rust example

// An event object, which can be converted to [u8] to be written to a file
#[derive(Copy, Clone, Debug, Default)]
#[repr(C)]
pub struct Event {
    pub id: usize,
    pub flags: usize,
    pub data: usize
}

// An example file, a network interface
let file = OpenOptions::new()
    .read(true)
    .write(true)
    .custom_flags(O_CLOEXEC | O_NONBLOCK)
    .open("network:").unwrap();

// Create a new event queue. This is tracked by file id
let mut event_queue = OpenOptions::new()
    .read(true)
    .write(true)
    .custom_flags(O_CLOEXEC)
    .open("event:").unwrap();

// Create a request for read events on the file, with a unique token
let event_request = Event {
    id: file.as_raw_fd(),
    flags: EVENT_READ,
    data: 0x1234
};

// Add the event request to this event queue
event_queue.write(&event_request).unwrap();

loop {
    // Wait for the next event
    let mut event = Event::default();
    let count = event_queue.read(&mut event).unwrap();
    if count == mem::size_of::<Event>() {
        // The event should have the id set to the network file, the flags set
        // to EVENT_READ, and the data set the same as the request
        assert_eq!(event, event_request);
    } else {
        panic!("invalid size of event: {}", count);
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

A number of programs will need to be updated to the new event system. The old
system will by necessity need to be removed.

# Alternatives
[alternatives]: #alternatives

The alternatives are to attempt to implement different event handling in a
userspace daemon.

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?

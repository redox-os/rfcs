- Feature Name: standardized app fs layout
- Start Date: 2026-06-12
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
Each app will have its own directory where it is name spaced.

# Motivation
Standardizing how/where apps keep all of their files will make standardizing the sandbox much easier. All of the app's resources (including icons and signatures) and info about the app will be stored in this directory. System apps can follow this same scheme, although the directory your would be in a different location.

# Detailed design
The structure of the layout would be similar to the following.
```
$USER/
├── Applications/
│   └── org.redox-os.app/
│       ├── bin/
│       ├── cache/
│       ├── config/
│       ├── data/
│       └── meta/
│           ├── certificates/
│           ├── icons/
│           └── manifest
├── Desktop/
└── Downloads/
...
```
System applications can simply be in `/System/Applications`.

# Drawbacks
This may be unfamiliar to most desktop Linux users, but NixOS, Android, and macOS use something vaguely similar. This wouldn't matter to users who don't spend a lot of time in a shell, but it is also fairly intuitive anyway.

# Alternatives
The FHS used by many Linux distros is not at all intuitive and can unnecessarily complicate sandboxing. There aren't many alternatives that are drastically different to this proposal.

# Unresolved questions
There are multiple different ways to store the signatures for executables, and it will partially depend on how the apps are packaged. Additionally, how metadata (such as a description) that sexist exists in `.desktop` can also be implemented in numerous ways.
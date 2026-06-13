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
Permissions could reasonably be inside `meta/` as only certain permissions could be requested by an app. What permissions are actually granted would be handled elsewhere are verified by the kernel or some other system component that they are actually valid.
System applications not installed by the user can simply be in `/System/Applications`. While this rfc is not about whether all apps are installed per user or not, something similar to the way Android does it would make sense. When installed, the app could optionally be made available to all users without needed to actually modify any core parts of the system. If one user updates the app, all users would have the updated version since there are no copies of the app. This would make it much easier to keep things updated, although if users want seperate installations, that (either scenario) can also be handled without making changes here.

# Drawbacks
This may be unfamiliar to most desktop Linux users, but NixOS, Android, and macOS (perhaps Windows too) use something vaguely similar. This wouldn't matter to users who don't spend a lot of time in a shell, but it is fairly intuitive anyway.

# Alternatives
The FHS used by many Linux distros is not at all intuitive and can unnecessarily complicate sandboxing. There aren't many alternatives that are drastically different to this proposal.

# Unresolved questions
There are multiple different ways to store the signatures for executables, and it will partially depend on how the apps are packaged. Additionally, how metadata (such as a description) that exists in `.desktop` can also be implemented in numerous ways.

- Auth scheme Interface: This is a more official proposal for what I've been yammering on about on mattermost for the past weeks.
- Start Date: 12-27-17 (Although this has been in the works for more than two weeks at this point)
- RFC PR: 
- Redox Issues: [Redox Users](https://github.com/redox-os/users/issues/6)
        

# Summary
[summary]: #summary

The goal is a language agnostic scheme interface for doing user authentication and getting user and group information from the system. In an ideal world, an interface that doesn't require any sort of static or dynamic library to be able to interact with, and possibly even be usable from the shell (fancy!).

# Motivation
[motivation]: #motivation

redox_users was proving to have some complications (see Alternatives) that using a scheme interface would solve. A scheme has it's own drawbacks (see Drawbacks), but the pros seem to outweigh the cons.

# Detailed design
[design]: #detailed-design 

The current design is comprised of a single scheme, `auth:` that provides a large number of urls. This may not be an entirely comprehensive list of all the nessasary functionality.

In this documentation, `login-name` is used as the name of the user that is inputted into login fields, and `comment` is used as the long username, the one that apears in a login-manager, for example.

* `auth:/` - Open returns error
  * `user/` - Interface for Mapping of all login-names to userids, like so:
    ```
    root:0
    user:1000
    ```
    Only openable in read mode. Any user on the system has access. Also a part of deeper URL's:
    
    * `<login-name>/`
      Calling read on this url provides a semicolon-delimited list of all user information:
      ```
      <login-name>;<uid>;<primary-gid>;<comment>;<home-dir>;<login-shell>
      ```
      Calling write on this url takes a semicolon delimited list of user information, as above, and throws an error if the list is not valid.
      * 
  * `group/` - Inteface for Mapping of all groupnames to groupids, like so:
    ```
    root:0
    sudo:1
    user:1000
    ```
    * `<groupname>/`
      Calling read on this url provides a semicolon-delimited list of all group information:
      ```
      <groupname>;<gid>;<username>,<username>,<etc>
      ```
      Only openable in read mode. Any user on the system has access. Also a part of deeper urls.
  * `useradd` - Interface for adding users
    Reads a string like so: 
    ```
    <login-name>;<uid>;<primary-gid>;<comment>;<home-dir>;<login-shell>
    ```
    Only openable in write mode
  * `groupadd` - Interface for adding groups
    Reads a string like so:
    ```
    <groupname>;<gid>;<username>,<username>,<etc>
    ```
    Only Openable in write mode
  * `userdel` - Deletes a user
    Reads the username. Throws error on `root`
    Only openable in write mode.
  * `groupdel` - Deletes Group
    Reads the groupname. Throws error on `root`
    Only openable in write mode
  
# Drawbacks
[drawbacks]: #drawbacks

A scheme is significantly larger and more complex than a static library (or crate). That exposes more surface area for security exploitation and the other things that come along with added complexity. The source code for an auth scheme would be in the range of 3-5 times the size of a static library.

There may also be some speed concerns, because a scheme requires programs to do file i/o instead of simply calling into an API.

# Alternatives
[alternatives]: #alternatives

A static library (crate) could be used instead (this is the current approach).

The problem is with access to the underlying files storing the data that the static libs access. Multiple executables attempting to access and potentially write to the same files at the same time can cause a lot of headache. Abstacting this behavior away to a single scheme that handles all the storage of that data itself and presents an adequately asynchronus interface to users solves this problem.

# Unresolved questions
[unresolved]: #unresolved-questions

The interface itself is pretty mutable at this point, and I'm not sure I've covered all the bases with what functionality it actually needs to have.

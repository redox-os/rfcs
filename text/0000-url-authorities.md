- Feature Name: url_authorities
- Start Date: 2020-07-28
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Preface

<!-- Non-standard section added to clarify important terminology and assumptions -->

An absolute URL, defined by [RFC 3986](https://tools.ietf.org/html/rfc3986), has the following syntax:

```
scheme : ┬─────────────────────────────┬ path
         └ // ┬────────┬ host ┬────────┤ 
              └ user @ ┘      └ : port ┘
     
              (    the  `authority`    )
```

We have relative URLs, too. For example, on the web page [http://example.com/foo](.), the relative URL [/](/) refers to [http://example.com](//) (the scheme and authority), and the relative URL [//](//) refers to [http:](//) (the scheme).

Yhe optional authority section applies to the `file:` scheme, too. See [Wikipedia § File URI Scheme](https://en.wikipedia.org/wiki/File_URI_scheme#How_many_slashes?) and [RFC 8089 § Appendix B](https://tools.ietf.org/html/rfc8089#appendix-B):

> __Appendix B.&nbsp;&nbsp;&nbsp;Example URIs__
>
> The syntax in Section 2 is intended to support file URIs that take the following forms:
> 
> _Local files:_
> * A traditional file URI for a local file with an empty authority. This is the most common format in use today. For example: 
>   * `file:///path/to/file`
> * The minimal representation of a local file with no authority field and an absolute path that begins with a slash "/".  For example:
>   * `file:/path/to/file`
> 
> _Non-local files:_
> * A non-local file with an explicit authority.  For example:
>   * `file://host.example.com/path/to/file`

Double slashes in paths to reference remote servers is [permitted by POSIX](https://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap04.html#tag_04_11), [supported by Bash](https://stackoverflow.com/questions/20690828/what-does-double-slash-in-cd-mean-in-linux), [used on Cygwin](https://unix.stackexchange.com/a/12291), and [user on Windows](https://docs.microsoft.com/en-us/windows/win32/fileio/naming-a-file?redirectedfrom=MSDN).

# Summary
[summary]: #summary

<!-- One para explanation of the feature. -->

This RFC proposes standardizing Redox URLs based on IEFT RFCs, learning from the lessons of the internet. Schemes would represent *protocols* (instead of *resources*) so that URLs self-document how they can be interacted with. File-protocol resources such as `env:` would be moved under `file:` as *authorities.* For example, `env:PATH` could be moved to `file://env/PATH`, which conforms to the `file:` scheme specification ([RFC 8089](https://tools.ietf.org/html/rfc8089#appendix-B)). A DNS-like system is proposed to handle allocation under the authority section along. Domains could also be used to re-implememt namespacing.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

Per IEFT, URL schemes should be *protocols*, such as [stfp, ssh, git, or http](https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml). This is opposed to URN schemes, such as [isbn, issn, or ieft](https://tools.ietf.org/html/rfc8141#section-1), that identify *resources* without specifying the protocol through which those resources are retrieved. [\[1\]][1] [\[2\]][2] [\[3\]][3]

[1]: https://webmasters.stackexchange.com/a/77783  
[2]: https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml  
[3]: https://www.w3.org/TR/uri-clarification/#uri-partitioning

Enforcing that schemes must represent protocols (as opposed to resources) would ensure that Redox URLs are similar to the types of URLs seen in web browsers. This change would mean that URLs self-document the method through which they can be interacted with. For example, a hypothetical `http:` scheme would be expected to implement the HTTP protocol rather than being some wrapper that expects web pages to be accessed as files.

Moving daemons into the authority section of URLs allows multiple daemons to be accessed from the same protocol, and it allows one daemon to be accessed through multiple protocols. The `file:` protocol could be used for multiple daemons, such as `file://sys` and `file://env`. This daemons may be accessible though multiple protocols, perhaps as a database at `sql://sys` or a REST API at `http://sys`.

Introducing the concept of domains (with subdomains) under the authority section of URLs would allow daemons to host resources such as `file://example.com.http` to translate from one protocol to another. It's proposed that a user can `chroot file://http`, in this example, to allow `file://example.com` to map to `file://example.com.http`.

This use of domains as namespaces is very user-friendly. A user could implement, for example, a `foobar` domain to route subdomains over the network to another Redox host. A user then could `chroot ://raspberry-pi.foobar` to enter a namespace where all URLs are seen from the pespective of a Redox-running Raspberry Pi on the LAN.


# Detailed design
[design]: #detailed-design



<!-- This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with the language to understand, and for somebody familiar with the compiler to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. -->

*TODO: researched how authority allocation and naming would be implemented*

# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

Redox software currently expects schemes such as `env:` and `sys:` to exist. This RFC would break that software (although the author would be happy to donate time to fix these things).

Another problem is having `//` different from `/`. Some software may calculate paths as `$PREFIX/lib`. If `$PREFIX` is `/`, the path would be `//lib`, which is problematic. Fortunately, though, this is easy to detect and fix if source code is available.

# Alternatives
[alternatives]: #alternatives

<!-- What other designs have been considered? What is the impact of not doing this? -->

# Unresolved questions
[unresolved]: #unresolved-questions

<!-- What parts of the design are still TBD? -->

How authority allocation and naming would be implemented should be discussed.



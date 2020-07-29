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

These relative paths, along with the optional authority section, apply to the `file:` scheme, too. See [Wikipedia § File URI Scheme](https://en.wikipedia.org/wiki/File_URI_scheme#How_many_slashes?) and [RFC 8089 § Appendix B](https://tools.ietf.org/html/rfc8089#appendix-B):

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


# Summary
[summary]: #summary

<!-- One para explanation of the feature. -->

This RFC proposes implementing a system for registering authorities under URLs. It also proposes standardizing Redox schemes as *protocols* instead of *resources*.

File-protocol resources such as `env:` would be moved under `file:` as *authorities.* For example, `env:PATH` could be moved to `file://env.localhost/PATH`, which conforms to the `file:` scheme specification ([RFC 8089](https://tools.ietf.org/html/rfc8089#appendix-B)).

This RFC additionally proposes requiring that the current working directory always be under the `file:` scheme so that all absolute-path file-like resources can always be referred to using addresses starting with `//` or `/`.

Together, these changes would allow significant improvements to Redox hygiene and Linux compatibility.


# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

Redox is a platform for building reliable systems while supporting existing Linux software. One of the ways Redox provides reliability is through modularity. This is reflected in Redox's concept of "everything is a URL" as opposed to Linux's more tightly-coupled "everything is a file."

Although the concept of "everything is a URL" brings a lot to the table, Redox's current *implementation* is inconsistent and causes compatibility issues with existing Linux software.

Per IEFT, URL schemes should be *protocols*, such as [stfp, ssh, git, or http](https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml). This is opposed to URN schemes, such as [isbn, issn, or ieft](https://tools.ietf.org/html/rfc8141#section-1), that identify *resources* without specifying the protocol through which those resources are retrieved. [\[1\]][1] [\[2\]][2] [\[3\]][3]

[1]: https://webmasters.stackexchange.com/a/77783  
[2]: https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml  
[3]: https://www.w3.org/TR/uri-clarification/#uri-partitioning

Enforcing that schemes must represent protocols (as opposed to resources) would ensure that Redox URLs are similar to the types of URLs seen in web browsers. This change would also mean that URLs self-document the method through which they can be interacted with. For example, a hypothetical `http:` scheme would be expected to implement the HTTP protocol rather than being some wrapper that expects web pages to be accessed as files.

Moving file-implementing resources such as `sys:` and `env:` under the `file:` scheme as authorities would allow these URLs to self-document their interface without dirtying the `file:/` tree. These resources, `sys:` and `env:` could also provide interfaces as authorites under multiple protocols, such at a database at `sql://sys.localhost` or a REST API at `http://sys.localhost`.

Enforcing that the current working directory implement the `file:` protocol would mean that the cwd must be under the `file:` scheme, in turn ensuring that all file-like resources can be referred to using addresses starting with `//` or `/`. This fixes compatibility issues with Linux software expecting absolute paths implementing the file protcol to start with a forward slash ([rust-lang/rust#52331](https://github.com/rust-lang/rust/issues/52331)).

Making full use of URL authorities would also allow for implementing a hypothetical `unix.localhost` host under `file:`. This could wrap existing `file:` authorities, directly accessing `disk:` for local parititons, to implement the [Filesystem Hierarchy Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard) without dirtying the file tree in `file:/`. Linux software expecting paths such as `/dev/urandom` could be launched via (for example) `( cd //$CURRENT_AUTHORITY@unix.localhost/$PWD && the_software_binary )` without a performance overhead.


# Detailed design
[design]: #detailed-design

<!-- This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with the language to understand, and for somebody familiar with the compiler to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. -->




<!-- 
In practice, the current working directory in Redox is almost always under the `file:` scheme. Thus, `/` on Redox should refer to the primary partition, and `//` should refer to the `file:` scheme itself.



This RFC proposes requiring that the current working directory implement the `file:` protocol (read, write, seek, mkdir, etc). In other words, because of the way URL schemes work, the cwd scheme must be `file:`. This would guarentee that `//` always refers to `file:`.

Our problem was that Linux software, expecting paths to resources implementing the file protocol, cannot handle paths not starting with a slash. This change would allow all file-implementing resources to be referred to using a slash prefix. For consistency, it's recommended that `file:` specifically is discouraged in favor of `//`.



It's proposed that there be a (modular) process for registering handlers under schemes.

For example, there could be authorities under `file:` for each disk partition. Or, `ftp:` schemes (which by themselves would implement the socket-like `ftp:` protocol) could "mount" themselves as authorities under `file:`.

Note that `http:` could provide an authority under `file:` for reading web pages using the shell without needing to strip headers or craft `GET` requests. The `http:` scheme would be used to reference a socket for the actual http protocol.


Note the environment variables in that shell invocation. It could helpful to have the shell or kernel define the environment variables `$CURRENT_HOST`, `$CURRENT_USERINFO`, and `$CURRENT_AUTHORITY`.

Authorities would be useful for many schemes, not just `file:`. For example, sqlite could register itself as an authority under a hypothetical backend-agnostic `sql:` scheme. This authority pointer could then be changed, if needed, to a remote database.

This separates protocols from implementations. A service could expose itself as accessible through multiple interfaces, such as `file:`, `sql:`, or even a localhost REST API under `http:`. -->

# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?


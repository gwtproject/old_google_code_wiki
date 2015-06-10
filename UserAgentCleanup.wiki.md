(work in progress -- please don't read too much into this yet)

# Introduction


# Details


# Related Issues
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=2815
  * http://code.google.com/p/google-web-toolkit/issues/detail?id=2938

# Notes
  * c.g.g.user.UserAgent needs to be moved into its own module, rather than being a part of .user.
  * Try to refactor classes deferred-bound on user-agent such that there is a "standard" fallback for all cases. In theory, this should allow us to support a "standard" browser correctly without having an explicit deferred-binding target for it.
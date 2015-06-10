# Introduction

We still need to look into this, but it appears that the allocation of obfuscated identifier names is sub-optimal. There are two places this appears to happen:

The first is a very simple case in JsObfuscateNamer, which allocates names using a simple base64 encoder. But in order to avoid using illegal leading characters, it switches to base32 for the first character. This excludes 22 valid characters (only numeric characters are illegal in this context).

The second still requires some research, but it appears from a cursory examination of the compiled output that fields and global identifiers are allocated from the same namespace (as opposed to locals, which are not). Fixing this could lead to significantly more efficient name allocation.

# Other Notes

Are there other characters that are valid in Javascript identifiers, that we're not currently using? The following is the current set:

```
  private static final char[] sBase64Chars = new char[] {
      'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n',
      'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', 'A', 'B',
      'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P',
      'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', '0', '1', '2', '3',
      '4', '5', '6', '7', '8', '9', '$', '_'};
```

# Performance Impact

At best, positive. At worst, zero.
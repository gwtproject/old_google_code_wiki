

# Ideas to Explore

## Chunking script blocks
Break compiled script across multiple smaller `<script>` tags. knorton has results to show that this allows the parser to speed up (on Firefox -- proving its not linear) while allowing the browser to stay more responsive, because it may be that it would parallelize parsing and downloading .cache.html in chunked responses.

## Non-recursive RPC deserialization
That is, a little language where the payload drives deserialization.Eval each chunk in a multipart response.
Good because
  * Avoiding recursion is good; we suspect that deep call stacks slow things down non-linearly
  * Returning the response using multipart would cause XHR to fire readyState = 3 events; thus pipelining deserialization
Things to keep in mind
  * Not all browsers support this
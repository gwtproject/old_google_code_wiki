# Introduction

Historically, the GWT compiler would produce the full Cartesian product of deferred-binding property/value pairs and then produce a permutation for each set of unique rebind answers.  For large projects, the number of permutations increases rapidly as these projects tend to have multiple permutation axes.  As as example, `user.agent * locale * log.level * user.kind` could easily result in hundres of permutations.  Because each permutation is optimized individually and produces a unique collection of JavaScript, reducing the number of permutations can shorten compile cycles and reduce storage requirements.

The GWT compiler offers these mechanisms for controlling the number of permutations produced:
  * [Conditional properties](ConditionalProperties.md) allow the permutation matrix to be reduced by expressing relationships between deferred-binding properties.
  * [Soft permutations](SoftPermutations.md) use runtime selection of implementation types within a permutation.
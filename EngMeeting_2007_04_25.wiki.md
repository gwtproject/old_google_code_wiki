# Agenda
  * Pre-planning for after 1.4
  * Not talk about 1.4 status for a change :-)

# Assessment of Current GWT Functionality
The goal of this discussion was to decide, _very_ loosely, how far along we are in the evolution of various functional areas of GWT. We took a really rough pass at dividing things up into functional areas and assigning each a score from 1-10, where the score represents "how much we have done compared to what we think we can do".

## Deployment of your compiled app into production
  * _Score: 5_
  * Could auto-generate WARs, Gadgets
  * This low score probably comes from bleed-over from the complete and utter absence of deployment guidance in our doc

## Documentation
  * _Score: 3_
  * Lacking deployment, best practices, portability concerns, security, layout, integration
  * Needs better TOC and an index
  * Could document IDE-specific How-Tos, step-by-step

## Hosted mode startup speed
  * _Score: 3_
  * Major pain point is JUnit startup speed (score is -1!)
    * This is particularly bad because it discourages good practices (e.g. running tets before commits, test-driven development)

## Hosted mode value-add
  * _Score: 2_
  * Incompatible browsers  burned into hosted mode is a major pain point (and will only get worse)
  * People can't use the other cool plug-ins that help with web dev
  * Lots of ways that hosted mode could provide value beyond just running your app

## IDE Integration
  * _Score: 1_
  * Not easy enough to launch apps without creating config

## UI
  * _Score: 4_
  * Not pretty; need infrastructure to suppor themes
  * APIs could be more consistent (e.g. listeners vs. handlers)
  * Declarative UI
  * Drag and drop
  * Effects

## RPC
  * _Score: 4_
  * Type mapping
  * POJO integration (e.g. Hibernate use case)
  * Response deserialization SSW
  * Server-push
  * Targeting other backends than Java (e.g. PHP)

## Testing
  * _Score: 3_
  * Need more platforms in test environment
  * Should revisit our basic approach (not necessarily b/c we plan to change it)
  * Need to integrate something like selenium
  * Pull bug sink in
  * Improved coverage of our own source
  * Incorporate benchmarking and check for regressions
  * Automatic API compatbility regression detection
  * Making JUnit faster/parallel testing

## Java compliance
  * _Score: 5_
  * Need to support Java 1.5+
  * Go through each JRE thingy and make an explicit decision and document why/why not
  * Stack traces and assertion support

## Compiler speed
  * _Score: 1_
  * Uses lots of memory and it's friggin' slow, especially bad for lots of perms
  * Can do way less work and compile permutations in parallel

## Compiler output quality
  * _Score: 5_
  * Inlining Java, JSNI
  * Intrinsics too costly
  * Control flow analysis
  * Better multi-type analysis
  * No way to indicate never-null during optimizations, which would let us optimize out null tests, etc.
  * String table
  * Object recycling (based on escape analysis)
  * Honoring assertions and/or semantic annotations for the purpose of compilation

## Compiler features
  * _Score: 4_
  * Possibility of factoring out common code (and maybe stitching it together in a servlet or something)
  * Subsets of permutations in GWT
  * Import (zero-overhead)/export of JS APIs
  * Could output gzip directly



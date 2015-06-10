_NOTE: This design document is out of date.  For the new design as of GWT 2.0, please see CompilationUnit._

# How Compilation Units are Managed in the GWT Infrastructure
## Overview
In GWT, both running hosted mode and compiling for web mode share infrastructure for building `TypeOracle` and managing Generators.  Up to now we have not had a good name or well-defined boundaries for this viscera, but hopefully this design doc will shed some light on things related to the `CompilationUnit`.  All doc is circa GWT 1.5 RC1, when a big refactoring effort over all this infrastructure occurred, code named "Flaming Sword of Death".

Update: _**GRAVEYARD** is now implemented in Gwt trunk (r5048 (on Google Code))._

## Goals
  * Read Java source files off the classpath based on module source inclusions
  * Compile included Java source using JDT
  * Perform custom error checking for GWT-specific constructs
  * Log any compilation errors
  * Build and maintain a `TypeOracle`, which reflects the full body of compilation units
  * Assimilate new compilation units from Generators
  * Produce Java class files for use in hosted mode
  * Efficiently refresh from disk (important for hosted mode refreshes)
  * Provide compiled class files for hosted mode execution without additional compilation

## Glossary of Moving Parts
### `CompilationState`
> Represents the total set of `CompilationUnits` for a module; a member of a GWT module.  The `CompilationState` is reponsible for managing the set of `CompilationUnits` and ensuring orderly state transitions.  The `CompilationState` also manages the module's `TypeOracle`.

### `CompilationUnit`
> The central piece of the whole puzzle.  A `CompilationUnit` represents a single Java source file, but includes all accumulated state and output as compilation and validation proceeds.  Subparts include:
    * the parsed JDT AST for the `CompilationUnit`
    * a set of `CompiledClasses`, which contain class-specific artifacts

### `CompilationUnit.State`
> Any given `CompilationUnit` can be in exactly one of several discreet states.  Here are the possible states and associated invariants:
  * **FRESH**: All internal state is cleared; the unit's source has not yet been compiled by JDT.
  * **COMPILED**: In this intermediate state, the unit's source has been compiled by JDT.  The unit will contain a set of `CompiledClasses`.
  * **ERROR**: In this final state, the unit was compiled, but contained one or more errors.  Those errors are cached inside the unit, but all other internal state is cleared.
  * **CHECKED**: In this final state, the unit has been compiled and is error free.  Additionally, all other units this unit depends on (transitively) are also error free.  The unit contains a set of checked `CompiledClasses`.  The unit and each contained `CompiledClass` releases all references to the JDT AST.  Each class contains a reference to a valid JRealClassType, which has been added to the module's `TypeOracle`, as well as byte code, JSNI methods, and all other final state.
  * **GRAVEYARD**: A **CHECKED** generated unit enters this state at the start of a refresh.  If a generator generates the same unit with identical source, the unit bypasses costly compilation, validation, and `TypeOracle` building. Note that a **GRAVEYARD** unit is invalidated in any of the following cases:
    * if its source changes
    * if the state of any units that it depends on has become **FRESH** or **ERROR**
    * if a source unit with the same main type is found

### `CompiledClass`
> Represents a single compiled Java class; a member of a `CompilationUnit`.  Subparts include:
  * the parsed JDT AST for the class
  * a reference to a JRealClassType in the module's `TypeOracle`
  * the actual byte code of the class file
  * a set of JSNI methods associated with the class

### `TypeOracleMediator`
> Helper class to handle the details of reflecting the JDT AST nodes provided by the `CompilationState` as a `TypeOracle`.

### `JdtCompiler`
> Helper class to handle the details of transforming Java source files into both JDT AST nodes and compiled Java byte code.

### `JavaSourceOracle`
> An abstraction that provides a single unified view of all source files available to the current module.  Implemented on top of a `ResourceOracle`.

### `ResourceOracle`
> A classpath abstraction that provides a unified view of all resources available on the Java classpath given a set of included packages and optional filters.  This is used for both source files and public files, and the set of included packages is determined by the module's source and public declarations.

## Class Connection Diagram

![http://google-web-toolkit.googlecode.com/svn/wiki/TypeOracleCompilationDesign.png](http://google-web-toolkit.googlecode.com/svn/wiki/TypeOracleCompilationDesign.png)

## Algorithm Overview for Various Use Cases
### #1: `ModuleDef`.getTypeOracle() called the first time
  1. `CompilationState` compiles
  1. `JdtCompiler` compiles all **FRESH** and **ERROR** units; those units become **COMPILED**
  1. All valid type names for classes with corresponding source code are recorded for validation
  1. Any units with JDT errors transition to **ERROR** state; all errors are logged
  1. Custom validation is performed over **COMPILED** units, recording new errors
    1. JSO restrictions checked using JDT AST
    1. Primitive long access from JSNI methods checked using JDT AST
    1. References to binary types checked using set of valid source types and JDT AST
    1. Any units with custom errors transition to **ERROR** state; all errors are logged
  1. Any units (transitively) depending on **ERROR** units are invalidated and become **FRESH**
    1. Each **COMPILED** unit lazily computes its direct references to other units
    1. All units depending on **FRESH** or **ERROR** units are transitively invalidated and become **FRESH**
    1. Each unit that is invalidated generates an error log message
  1. JSNI methods are collected from each unit that remains **COMPILED**
  1. `TypeOracleMediator` builds a JRealClassType for each class that belongs to a **COMPILED** unit, and places that JRealClassType into `TypeOracle`
  1. All **COMPILED** units become **CHECKED**
    1. Unneeded internal state (such as references to JDT's AST) are now cleared

### #2: `ModuleDef` wants to refresh
  1. `ResourceOracle` for source path refreshes against file system
  1. `JavaSourceOracle` refreshes against `ResourceOracle` if `ResourceOracle` state changed
  1. `CompilationState` refreshes against `JavaSourceOracle`
    1. All **GRAVEYARD** units are removed
    1. All generated units become **GRAVEYARD**
      1. Any contained JRealClassTypes are removed from `TypeOracle`
    1. Any source files whose corresponding file was changed in `JavaSourceOracle` are invalidated and become **FRESH**
    1. The graveyard units are checked against new source units
    1. All units whose corresponding file was removed from `JavaSourceOracle` are removed
    1. Any new source files in `JavaSourceOracle` have new `CompilationUnits` created
    1. All units depending on **FRESH** units are transitively invalidated and become **FRESH**
  1. The sequence of events in use case #1 occurs

### #3: Newly generated units need to be assimilated
  1. A set of **FRESH** `CompilationUnits` contained generated source are added to the `CompilationState`
  1. For each new unit, if a **GRAVEYARD** unit with the same type name already exists, the source of the new unit is compared to the source of the existing unit.
    1. If the new unit's source is identical to the old unit's source, the new unit is discarded
      1. All JRealClassTypes in the contained `CompiledClasses` are re-added to `TypeOracle`
    1. If the new unit's source differs from the old unit's source, the old unit is discarded and the new unit is added
  1. If any **FRESH** units remain, the sequence of events in use case #1 occurs

### #4: A `CompilationUnit` becomes invalid
  1. The reference to the JDT cud is removed
  1. The set of cached references to other units is cleared
  1. For each `CompiledClass`
    1. The contained JRealClassType is removed from `TypeOracle`
    1. All internal state is cleared
  1. The set of `CompiledClasses` is cleared
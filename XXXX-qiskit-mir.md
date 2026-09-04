# Qiskit middle-end IR for fault-tolerant compilation

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | #### |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com), Kit Barton (kbarton@ca.ibm.com) |
| **Submitted**     | 2026-08-31 |


## Summary

Qiskit will introduce a new middle-end intermediate representation (IR) for fault-tolerant compilation.
This will be one section of full-path compilation from high-level program description down to specific hardware.

It will be the primary target for higher-level applications representations to lower to, both from inside and outside Qiskit.
The IR will support hardware-informed optimisations and rewrites, including incremental physical-qubit/module assignments, but not be a concrete representation of any hardware/error-code family.
The IR will be subsequently lowered to a variety of code/hardware-specific representations, depending on lower-level backend requirements.

The quantum instruction set will permit several different computational models, and expect pipelines to specific backends to narrow the used set down before lowering.
It will include Pauli-based computation, Clifford tableaus and explicit low-arity gates.

Some examples of transformations we expect to able to perform on this IR:

- placement of "virtual" qubits onto hardware qubits and modules
- operation translation
- explicit Clifford-correction tracking
- continuous-rotation synthesis to discrete gates (e.g. `rz` synthesis)

The IR will permit a small amount of classical computation and control flow, but this will deliberately limited during the first designs.

The IR will simultaneously model both "virtual" and "physical" qubit addressing.
Virtual addressing uses qubits, groups and connectivity defined inline in the IR, prior to allocation to specific physical qubits and groups.
The properties of each qubit and group in the physical-addressing mode will be communicated by an abstract interface implemented by backends.


## Motivation

Fault-tolerant compilation will have a different shape to NISQ-era compilation.
Qiskit's existing representations are designed and optimised for gate-model computation.
Trying to evolve `QuantumCircuit`/`DAGCircuit` in place risks comprising their suitability on current devices, and the split focus will stymie development of new forms.

Neither the ideal input forms nor the exact quantum-computer ISAs are precisely known yet, but we must begin trying to graduate individual research prototypes of components into research pipelines.
The clearest required component right now is a middle-end representation that can be targeted by various high-level algorithm-implementation languages, then lowered to a variety of implementation-specific backends (e.g. a compiler to one particular bicycle ISA).

### Detail on deficiencies in `QuantumCircuit`/`DAGCircuit`

- There is no concept of "qubit grouping".
  We expect many FT backends to have instructions that "address" qubits in groups.
  `DAGCircuit` cannot reason about this; it assumes operations on disjoint qubits are parallelisable.

- `DAGCircuit` has poor abstractions for partial hardware assignments.
  It is hard to reason about qubit allocation and de-allocation, other than globally.

- The classical-value representations are very poor.
  `Clbit` semantics are inherited from `Qubit`, but classical bits are not generally linear types.
  Measurement feed-forward has unnecessary restrictions on re-ordering in `DAGCircuit`,.

- The representations of Pauli-based compilation are inefficient.
  There is some scope for in-place improvement, but it's still bounded by historical baggage.

- The representations of multi-qubit instructions are inefficient.
  `QuantumCircuit`/`DAGCircuit` are very memory efficient in representing gates on small numbers of qubits, where the combinations are small.
  These benefits do not scale well to many-qubit instructions, such as those found in PBC.

- `DAGCircuit` is a _pure_ data-flow representation defined only by qubit/clbit/var objects.
  There is no (common/easy) way to choose a specific linearisation during lowering, and the costs of maintain the data-flow analyasis must be paid by every mutation.
  Backend compilers have been known to be sensitive to the precise linearisation.


## User Benefit

There are several user personas in mind, which have significant overlap in the general field of "compiler research" and "end user".
Here we pull out a couple of the more separable parts:

- *High-level algorithms language designers*.
  Will have a centralised IR to target, without being _required_ to know about specifics of the QEC backend structure.
  We anticipate that any Qiskit MIR program will be _possible_ to lower to any target backend, though we will provide abstractions in the IR (such as a generic concept of qubit "groups") that can be used to specialise lowerings on a per-pipeline basis.

- *Backend ISA developers*.
  Qiskit will provide a centralised abstraction / quantum virtual-machine representation for middle-end optimisation.
  We intend that large amounts of the compilation _can_ be done at the Qiskit MIR level, for any hypothetical backend (and Qiskit MIR will expand as we identify more places for suitable abstraction), and the ISA-specific lowering will be relatively cheap.
  This allows Qiskit to shoulder the greater burden of IR specification, compiler infrastructure, code optimisation, etc, which in turn allows the same effort to be shared among more users.

- *Quantum-computer users*.
  Most obviously: we need a platform for developing fault-tolerant compilers so users can access the hardware.
  These users benefit by us accelerating the work to get them access to the hardware.


## Design Proposal

### Design principles

Some general principles that guide decisions throughout:

- The initial design will not be complete and must permit extension without breakage.

- The API boundaries at the upper and lower ends are both movable.
  These are still active fields of research; we should support on-the-fly custom operations, and consider moving re-usable parts into the core IR.

- The first concern is producing a _valid_ full-path compilation pipeline; optimisation comes second.

- The IR must be scalable to fault-tolerant sizes.

- The IR must be extensible from compiled and interpreted languages, to allow faster prototyping.

- The IR is a base that is used in different ways by different compiler pipelines.
  It will contain efficient representations of certain objects that may not be used in all pipelines.

### Scope of this design

This design is purely concerned with the IR instruction set and data structures.
The associated pass-manager infrastructure and the specifics of particular pipelines are being developed separately.

This design is an MVP.
We expect significant future expansion after implementation.

### IR semantics

This is intended high-level semantics of what is representable in Qiskit MIR.
We don't get into detail of _how_ to represent this yet, only what should be possible.

#### IR structure

There are several top-level components to Qiskit MIR.
These include:

- **Instruction list**

  A linearisation of all the instructions in the circuit.
  It supports random access, efficient insertion and removal.

- **Quantum memory table**.
  MIR supports addressing both qubits and abstract groups (see [Quantum types](#quantum-types)).

  "Virtual" groups can be defined on-the-fly by passes as part of the IR, then used as single operands.
  A quantum memory table records the overlaps of groups (approximately: which qubits are in which groups), and can contain custom metadata for each group.

  We expect that one part of a future backend abstraction will be to provide a version of this table for the physical-qubit/-module layout.

- **Symbol table**

  A data structure that links symbol identifiers to information about the symbols, such as their types, whether they are literals (and if so, with what value), and so on.


Notably absent here: we are not defining a control-flow graph or the concept of a "basic block" in the initial implementation.
We expect to add structured control flow over blocks later, but it is not designed in this document.
Some of the above structure may change when we introduce this.


#### Instructions

An instruction tracks four explicit components:[^instruction-parts]

- **operation**: compile-time fixed information about the action itself.
  This includes some representation of which family of op code (e.g. `pauli_rotation` vs `cx`),
  but may also include operation-specific compile-time fixed information specific to that operation.

- **quantum arguments**: zero or more `qid`s, which the operation acts on.

- **classical symbols**: the classical values the operation acts on.

- **classical return**: optionally, the classical value(s) returned.


This section does not comment on the implementation of the instruction object, just on the concepts it permits representation of.
The quantum arguments all follow "qubit semantics" (see next section), while the classical symbol uses and the return follow the semantics of the type of value the symbol refers to.
In this first draft, the only classical types we define have value semantics.[^memory-semantics]

**TODO**: if more stuff (like Pauli strings, state-prep state, etc) moves into the argument list, revisit the motivation for the quantum-arument/classical-symbol split.

[^instruction-parts]: We may want to add further metadata/annotations/whatever to individual instructions in the future.
We need to make sure the implementation and APIs don't make this unnecessarily hard.
[^memory-semantics]: This doesn't preclude us adding the concept of "arrays" or other references to memory in the future.


#### Quantum types

The quantum data model of Qiskit MIR is abstract, and designed to allow concrete high-level algorithmic languages and concrete backends to model their own semantics.

MIR provides abstractions for quantum data-flow analysis that permits working with ad-hoc groups of qubits, while maintaining the linearity of individual qubits.
It allows multiple simultaneous representations of quantum memory, so high-level algorithms can use a free-form "virtual" layout of their choosing that is progressively lowered to a concrete hardware model.

We introduce three terms:

- **qid**: a "quantum identifier", which refers to zero or more qubits.
  These are the operands of instructions.

- **resource space**: each qid is part of exactly one resource space, and each resource space contains many qids.
  The two initial built-in resource spaces are called "virtual" and "physical".
  We may add more resource spaces in the future[^id-space-expansion].

- **qubit semantics**: the rules for how the underlying "quantum resources" can be modified/copied by instructions.[^qubit-semnatics]

[^id-space-expansion]: Approximately, I'm expecting that we might introduce one to handle loops over qubits, or function calls that can be applied to different qubits without instantiating a new function per location.  You don't need this concept in classical computing, where one function call always has the same register uses, but the CPU-register–physical-qubit analogy doesn't hold here.
[^qubit-semantics]: We don't have a full formal specification of "qubit semantics"  yet, but it will follow.
For now, approximately think of them as "a resource that each instruction can use at most once and cannot copy".

An "instruction" in Qiskit MIR takes zero or more `qid`s as arguments.
A `qid` has an optional "qubit width"; this defines how the built-in Pauli instructions act on it.
A `qid` can "contain" other `qid`s.
"Qubit" is not an explicit data type in Qiskit MIR.
A `qid` might directly represent a qubit or a set of qubits, however.

It is allowed and expected to have qubit overlap between different `qid`s used in the program; the set of "contained-in" relations between all `qid`s in the same resource space is an arbitrary DAG.
This allows a backend to define a "module" (for example) that contains other qubits, and have instructions that act on the entire module as a single named entity.
A Qiskit MIR program is invalid if any single instruction acts on two `qid`s that overlap.

There are two (initial) "resource spaces": virtual and physical.
At any given time, a Qiskit MIR program can use a mix of virtual and physical resources.
An individual instruction may have a mix of virtual and physical operands.

The "memory layout" of the virtual resource space is a property of the IR, and transformations are allowed to add to it, including "adding" new qubits and groups.
This is tracked in the _quantum memory table_ as part of MIR.

The "memory layout" of the physical resource space is a property of the target backend, not owned by an individual MIR program.
The memory table for the physical resource space will likely use the same data structure as the physical one.

*TODO*:
- define the "contains-in" relations (e.g. bit layout?), how data-flow analysis is defined, and how "overlap" is defined.


> [!NOTE]
> I (Jake) like to think of `qid`s as similar to LLVM's modelling of virtual and CPU registers.
> The analogy is not perfect, and do not assume that any unenumerated CPU-register semantics apply to `qid`, but it may help understand the spirit.

#### Classical types

**TODO**:

- bool
- Pauli string?
- compound?

Note: still a question of whether to make "Pauli string" always a symbol/argument or whether to make it a built-in of the op code.
We can always add "dynamic Pauli" variants later that take the Pauli string as an argument, allowing it to be a runtime symbol.
Need to consider whether there's a _large_ immediate memory advantage to giving it a special place in the opcodes.

#### Available operations

The purpose of built-in operations is to provide re-usable components that have high memory efficiency and can be acted on with very low overhead.
It must always be possible to have a "dynamic" operations that is defined by the runtime user of Qiskit MIR, to allow extension from outside Qiskit.
We may, in the future, add a "middle" mechanism that allows more efficiency than the "full dynamic" form, but doesn't require the first-party integration of the built-in set.

Not all pipelines will want to use all built-in operations.
We want to define the set of operations that we expect to both have widespread applicability and be so common that they motivate high efficiency or centralised algorithms to deal with their semantics.

There is a scale of how tightly we define the semantics of built-in assumptions.
The tighter, the more methods and behaviour we can give first-class re-usable support to.
The looser, the more situations the objects are re-usable in.

In all discussion here:

- "Pauli string" is a representation of a multi-qubit Pauli operator.
  We discuss below how Pauli strings relate to their `qid` arguments.

- "arity" means the number of arguments the operations needs.
  We're primarily concerned with the arity of `qid`s and of the implied qubits.

- "variadic" means a list of length that isn't fixed per operations.
  For example, `pauli_measure` with a 12-qubit Pauli string might take 12 `qid`s of single qubits, or 1 `qid` of a 12-qubit group.

The operations set includes:

**TODO**: revisit all of these and decide whether each argument is part of the "custom" compile-time fixed data of the operation, or gets a type in the symbol table.

- `pauli_measure` that takes a Pauli string and variadic `qid`s. Produces a bool representing the measurement outcome.

- `pauli_rotate` that takes a Pauli string, an angle, and variadic `qid`s.  Produces nothing.

- `pauli_project` that takes a Pauli string, a bool, and variadic `qid`s.  Projects the qargs into the state that would produce the same Boolean if `pauli_measure` applied to it.

- `pauli_correct` that takes a Pauli string and variadic qargs.  Equivalent to `pauli_rotate` with an angle of `pi` (and a global-phase correction).  Intended for representational efficiency, and potentially as a target for "predicated operations".

- `clifford_tableau` that takes some explicit Clifford tableau and variadic qargs.  Produces nothing.

- various explicit low-arity explicit Clifford "gates" (c.f. the `stim` operations set).
  Each take a number of `qid`s equal to the arity of the gate.
  *TODO*: define qubit-width semantics.

- certain low-arity non-Clifford gates (e.g. `t`).
  Same considerations on `qid` count and semantics as above.

- state preparation, which takes a variadic number of `qid`s and a "state" that they are reset to.

- dynamic extension operation, which will be objects defined externally to the main IR definition (either in Qiskit or elsewhere), and the builr-in IR methods will only interact with via a fixed interface.
  These are for extension types.

Note: we are currently experimenting with particular forms of "predicated" instructions in particular pipelines.
We are not currently in a place where we are confident on the representation or use, so will use the "dynamic operation" form to represent them at firts.


## Implementation Detail

*TODO*: implementation sketch.

## Alternative Approaches

## Questions

## Future Extensions

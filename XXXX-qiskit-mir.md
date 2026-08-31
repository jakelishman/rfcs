# Qiskit middle-end IR for fault-tolerant compilation

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | #### |
| **Authors**       | Jake Lishman (jake.lishman@ibm.com) |
| **Submitted**     | 2026-08-27 |


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
- instruction translation
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

- The IR will be akin to a complex instruction set.
  We will allow multiple representations at the higher end, and expect pipelines to translate to suitable sets for further lowering.

### Scope of this design

This design is purely concerned with the IR instruction set and data structures.
The associated pass-manager infrastructure and the specifics of particular pipelines are being developed separately.

This design is an MVP.
We expect significant future expansion after implementation.

### IR semantics

This is intended high-level semantics of what is representable in Qiskit MIR.
We don't get into detail of _how_ to represent this yet, only what should be possible.

#### Quantum types

*TODO*: I need to come back and finish this.  (At the moment it's risking getting too mathematical.)

The quantum data model of Qiskit MIR is abstract, and designed to allow concrete high-level algorithmic languages and concrete backends to model their own semantics.

We introduce two terms:

- `qid`: a "quantum identifier", which refers to zero or more qubits.
  Each `qid` is a unit that may be an instruction operand.

- address space: each `qid` is part of exactly one address space, and ecah address space contains many `qid`s.
  The two initial built-in "address spaces" are called "virtual" and "physical".

"Qubit" is not an explicit type in Qiskit MIR instructions, but it is strongly expected that in most pipelines, each physical qubit will be assigned a `qid`.

An "instruction" in Qiskit MIR takes zero or more `qid`s (for quantum identifier) as arguments.
A `qid` can "contain" other `qid`s.
It is allowed and expected to have qubit overlap between different `qid`s used in the program; the set of "contained-in" relations between all `qid`s in the same address space is an arbitrary DAG.
this allows a backend to define a "module" (for example) that contains other qubits, and have instructions that act on the entire module as a single named entity.

A Qiskit MIR program is invalid if any single instruction acts on two `qid`s that overlap.

#### Available instructions

The initial version of Qiskit MIR will add support for a limited but over-complete set of instructions related to Pauli and Clifford-T representations.
Several of these instructions may be "overloaded" / have multiple variants.

In all discussion here:

- "Pauli string" is some representation of a multi-qubit Pauli operator.
- "qargs" is some representation of an object that _could_ be unwrapped to an explicit list of qubits (though will permit using "qubit groups" in a way that does not require complete unwrapping, for efficiency).
- "arity" means the number of arguments the instruction needs.
  (It's only qargs we're worried about, in practice.)
  Note that this can be _either_ a low number of individual qubits, or a low number of "groups", depending on the instruction.
  "Low" means approximately `<=3`, mostly `<=2`.

The instruction set includes:

- `pauli_measure` that takes a Pauli string and qargs. Produces a bool representing the measurement outcome.
- `pauli_rotate` that takes a Pauli string, an angle, and qargs.  Produces nothing.
- `pauli_project` that takes a Pauli string, a bool, and qargs.  Projects the qargs into the state that would produce the same Boolean if `pauli_measure` applied to it.
- `pauli_correct` that takes a Pauli string and qargs.  Equivalent to `pauli_rotate` with an angle of `pi` (and a global-phase correction).  Intended for representational efficiency, and potentially as a target for "predicated instructions".
- `clifford_tableau` that takes some explicit Clifford tableau and qargs.  Produces nothing.
- various explicit low-arity explicit Clifford "gates" (c.f. the `stim` instruction set).
- certain low-arity non-Clifford gates (e.g. `t`)
- magic-state preparation / injection (*TODO*: detail)

We don't expect to implement all of these in the MVP, but these are indicative of the CISC-style ISA and scope of the IR.
Implementation order will depend on need.

#### Non-quantum types


## Detailed Design

*TODO*: implementation sketch.

## Alternative Approaches

## Questions

## Future Extensions

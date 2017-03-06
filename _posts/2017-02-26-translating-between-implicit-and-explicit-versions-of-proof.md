---
layout: post
title: Translating between implicit and explicit versions of proof
excerpt_separator: <!--more-->
---

This page organizes the supporting materials for the paper *Translating between
implicit and explicit versions of proof* by Roberto Blanco, Zakaria Chihani
and Dale Miller. Materials are divided in three parts.

 * General-purpose checkers and certificate definitions implementing the
   Foundational Proof Certificate framework, written in λProlog. These include
   FPC definitions for binary resolution, pairing, and maximally explicit
   elaborations, among others.

 * A checker written in OCaml that does not implement backchaining search nor
   unification. This checker, called MaxChecker, checks certificates in only
   the maximally explicit FPC.

 * A collection of proofs produced by the Prover9 theorem prover. Instructions
   are given to use the output from the prover to generate binary resolution
   certificates (for the λProlog kernel) and supporting declarations (for both
   λProlog and OCaml checkers). The λProlog checker can then check the theorem
   against a resolution certificate, and produce a maximally explicit
   elaboration that can be verified independently by the OCaml checker.

A description of each part follows.

<!--more-->

## λProlog checker and FPCs

[Public repository.](https://github.com/proofcert/lkf-snapshot)

First of all, a λProlog runtime must be chosen as the backend. We support
directly two implementations: one more mature (Teyjus), another newer and
scalable (ELPI).

### Teyjus backend

One possibility is to use a recent version of the
[Teyjus](https://github.com/teyjus/teyjus/) system as a backend. Our λProlog
repository contains a Makefile that can be used to build a set of modules with
the Teyjus compiler. By default it relies on a standard location for the Teyjus
binaries.

{% highlight make %}
TJROOT = /usr/local/bin
{% endhighlight %}

This variable can be changed if necessary. Afterwards, the file can be used to
build the entire set of default modules.

{% highlight bash %}
make
{% endhighlight %}

Alternatively, a `.lp` module target can be used to compile a specific module,
included or not in one of the default targets.

{% highlight bash %}
make resolution-elab.lp
{% endhighlight %}

The resulting program is ready to run on the Teyjus simulator, either
interactively (as in the example) or in batch mode.

{% highlight bash %}
tjsim resolution-elab
{% endhighlight %}

### ELPI backend

The [Embeddable λProlog Interpreter
(ELPI)](http://lpcic.gforge.inria.fr/system.html?content=systems) can be used
as a backend as well. We have used the [LFMTP 2016
release](http://lpcic.gforge.inria.fr/elpi-LFMTP16.tar.gz) built with an OCaml
4.02.1 toolchain.

No compilation is necessary. It suffices to load the interpreter with the
`.mod` file of the desired module, for example:

{% highlight bash %}
elpi resolution-elab.mod
{% endhighlight %}

### Code organization

The repository contains a checker implementing the FPC framework for classical
logic, and a collection of FPCs. Three categories of modules are of interest to
prospective users.

First, self-contained **concrete FPCs**, given in pairs of modules: `*-fpc`
contains the definition of the FPC proper, and `*-examples` creates an instance
of the checker by accumulating the kernel and the FPC definition, and adding
predicates to define and execute test cases, i.e., check theorems against
certificates. The following module pairs are provided.

 * `binarysans`: binary resolution with factoring, with ordered applications,
   without instantiation information. This is called `ordered-without` in the
   paper.

 * `binarysubst`: binary resolution with factoring, with ordered applications,
   with instantiation information. This is called `ordered-with` in the paper.

 * `binary-unordered`: binary resolution with factoring, with unordered
   applications, without instantiation information. This is called
   `unordered-without` in the paper.

 * `canonical`: maximally explicit elaboration with native integers as indexes.
   This is called `maximal` in the paper.

 * `canonical-inductive`: maximally explicit elaboration with inductively
   defined natural numbers as indexes.

 * `cnf`: conjunctive normal form as a decision procedure for propositional
   formulas.

 * `dd`: decide depth as means of controlling the decide rule.

 * `exp`: expansion trees.

 * `mating`: matings. The `mating-examples` module performs a sort of
   elaboration by attempting to find a mating for a given formula. The
   sub-module `mating-explicit-examples` adds a concrete mating to each example
   and actually checks the formula using this mating.

 * `oracle`: oracle strings.

Each `*-examples` module defines its test cases through an `example` predicate.
Each example is given a numeric identifier (ideally unique) and what
information is necessary to specify a formula (to be checked for theoremhood)
and a certificate (to check the formula). A `test` predicate takes an example
identifier as input, fetches both formula and certificate from the corresponding
`example` and checks this pair (this is called `test_resol` in the resolution
modules). As an alternative, `test_all` runs all examples.

No examples are given for the `canonical` and `canonical-inductive` modules.
These are meant to be exercised through the certificates generated by
elaboration to the maximally explicit format.

Second, there is the **FPC combinator**, `pairing`. Elaboration and
distillation strategies are obtained by combining two concrete FPCs through
pairing. FPCs must use disjoint sets of constructors.

Third, a group of **paired FPCs**, derived from concrete FPCs by the pairing
combinator, which perform elaboration and distillation. The checker is
instantiated in only a slightly different manner, accumulating two FPC
definitions instead of one as well as the pairing combinator. Each `example`
provides a certificate for the first member of the pair. This concrete
certificate is paired with a logic variable standing in for the second member,
and the resulting certificate pair is passed to the kernel for checking. Thus,
these paired FPCs are variants of the `*-examples` modules of the first group
that rely on existing `*-fpc` modules. The following combinations are included
in the distribution.

 * `cnf-mating-elab`: elaborate `cnf` to `mating`.

 * `dd-canon-elab`: elaborate `dd` to `canonical`.

 * `dd-oracle-elab`: elaborate `dd` to `oracle`.

 * `mating-canon-elab`: elaborate `mating` to `canonical`.

 * `oracle-dd-elab`: distill `oracle` to `dd`.

 * `p-distill`: distill `binarysubst` to `binarysans`. *

 * `ps-elab`: elaborate `binarysans` to `binarysubst`.

 * `sans-canon-elab`: elaborate `binarysans` to `canonical`.

 * `sp-elab`: distill `binarysubst` to `binarysans`.

 * `subst-canon-elab`: elaborate `binarysubst` to `canonical`.

 * `unordered-sans-elab`: elaborate `binary-unordered` to `binarysans`.

The purpose of these modules is to output the second half of the certificate
pair, that is, the new certificate that checks the formula and is not part of
the example definition. The predicate `test_elab` performs this variation on
checking.

In addition to the modules above, `resolution-elab` has been prepared to
encapsulate all operations related to the manipulation of resolution proofs. It
will be discussed in more detail as part of the certification scheme for
Prover9 proofs.

## OCaml maximal checker

[Public repository.](https://github.com/proofcert/maxchecker-snapshot)

MaxChecker is a functional kernel, written in OCaml, that implements a
determinate portion of the FPC framework, i.e., checking without backtracking
or unification. It can be used with any FPC definition sufficiently detailed to
part ways with both features.

It is presented pre-loaded with the maximally explicit certificate, enabling
direct use with exported formulas and certificates produced by
`resolution-elab` (see below), which emits code compatible with MaxChecker's
formulas and the built-in maximal FPC. The user must complete two
micro-modules, `Atom` and `Lkf_term`, each defining its corresponding type, and
make a call to the kernel with a clean formula and maximal certificate built
from these types.

We illustrate this in the `Test` module, which includes placeholders to declare
a (possibly exported) formula and certificate, and call the kernel, instantiated
with the maximal FPC, on the pair, reporting the result. It can be built using
the provided Makefile.

{% highlight bash %}
make
./test.native
{% endhighlight %}

## Prover9 certification

[Public repository.](https://github.com/proofcert/p9-snapshot)

In this section, we describe how to check resolution proofs produced by an
automated theorem prover, like Prover9, based on a resolution calculus. The
most permissive of the binary resolution FPCs that have been discussed ---where
applications are unordered and there is no substitution information--- subsumes
the inference rules used by Prover9 (with the exception of paramodulation).

### A multi-pairing FPC

The module `resolution-elab` accumulates several instances of pairing between
an unordered binary resolution certificate without substitution information and
various other, more explicit certificate placeholders, which are completed by
means of elaboration. These form a framework for our experiments in
certification of Prover9 proofs and "translating between implicit and explicit
versions of proof", echoing the title of the paper.

The repository contains an empty module, that is, it does not define any
non-logical constants or any examples of proofs by resolution based on those
constants: while it can be readily compiled, it is inert. It provides
placeholders where the following groups of elements may be supplied.

 * In `resolution-elab.sig`, constructors for atoms (of type `bool`) and terms
   (of type `i`), both of which may have term arguments. Each of these must be
   complemented in `resolution-elab.mod` by a clause of `pred_pname` or
   `fun_pname`, respectively. This constitutes the user signature. The module
   `test-constants` contains examples of this kind used by many of the
   `*-examples` modules.

 * In `resolution-elab.mod`, `example` clauses based on the signature. These
   are the combination of a numeric identifier, and a triple of lists: a list
   of base clauses, a list of derived clauses, and a list of justifications of
   the derived clauses representing a proof by binary resolution. See
   `binary-unordered.mod` for a collection of examples in this format. Note that
   the input formula is derived from the base clauses and is therefore given
   implicitly in clausal normal form.

The following utility predicates are defined by the module and can be used as
goals. Each takes a number as argument and operates on the `example` identified
by that number.

 * `check_unordered`: check the `binary-unordered` certificate specified by the
   example without any additional operations.

 * `elab_to_sans`: pair the `binary-unordered` certificate with, and elaborate
   into, a `binarysans` certificate through checking.

 * `check_sans`: perform the elaboration in `elab_to_sans`, followed by an
   independent check of the resulting `binarysans` certificate.

 * `print_sans`: perform the elaboration in `elab_to_sans`, followed by
   printing of problem size statistics.

 * `elab_to_subst`, `check_subst`, `print_subst`: like the `*_sans` predicates
   above, but pairing and elaborating the example with a `binarysubst`
   certificate.

 * `elab_to_max`, `check_max`, `print_max`: like the `*_sans` and `*_subst`
   predicates above, but pairing and elaborating the example with a `canonical`
   certificate.

 * `elab_and_export`: pair the `binary-unordered` certificate specified by the
   example with a `canonical` placeholder, as `elab_to_max`, and output the
   certified formula and maximally explicit certificate as OCaml code, ready to
   be imported in MaxChecker.

For all three pairings, predicates are given in triads of elaboration,
elaboration and checking of the new certificate, and elaboration and reporting.
The decomposition in steps is geared towards producing accurate measurements,
from which exact checking times can be derived. This approach has two practical
advantages. First, experiments can be carried out from a single λProlog
program, without exporting and importing intermediate results. Second, it
circumvents certain limitations in Teyjus; as a result, a direct comparison with
other implementations, like ELPI, is possible.

The `check_*` and `elab_to_*` predicates work without any further additions.
Reporting predicates rely on auxiliary operations on user-defined atom and term
constructors as follows.

 * `print_*` require a `size_bool` clause for each atom constructor and a
   cut-terminated `size_term` clause for each term constructor. The size of a
   constructor is defined as 1 (the constructor itself), plus the sum of the
   sizes of its term arguments.

 * `elab_and_export` requires a `print_name` clause for each atom constructor
   and a cut-terminated `print_term` clause for each term constructor. Atom
   names relate the string representation of atoms given in `pred_pname` and a
   valid OCaml identifier. Terms relate the constructor to a valid OCaml
   identifier and to the printed representations of its arguments.

All these additions can be easily generated automatically. To import a formula
and certificate into MaxChecker, the atom and term signatures must agree with
the identifiers given by the translation.

### Getting the data

The experimental dataset is derived from the [TSTP
section](http://www.cs.miami.edu/~tptp/cgi-bin/SeeTPTP?Category=Solutions) of
the [TPTP problem library](http://www.cs.miami.edu/~tptp/). Although the
official release tarball does not include problem solutions, it is possible to
reconstruct the proofs from it. Alternatively, they can be retrieved from the
web, say, using a tool like Wget with a call similar to this one.

{% highlight bash %}
wget --mirror --no-directories --follow-tags=a --accept-regex="\?Category=Solutions(&Domain=[^&]+(&File=[^&]+(&System=Prover9[^.]*.THM.*)?)?)?$" http://www.cs.miami.edu/~tptp/cgi-bin/SeeTPTP?Category=Solutions
{% endhighlight %}

This would collect `.s` theorem files in a single folder, together with
intermediate navigation pages. We can then extract the output of Prover9 from
the HTML and use its Prooftrans tool to elaborate this extracted proof text
into a lightweight, detailed proof outline. For this we use this simple call.

{% highlight bash %}
prooftrans expand renumber striplabels
{% endhighlight %}

This step requires having Prover9 installed in your system. Note that
Prooftrans does not strip all labels in all proofs, but these are easily
removed. We include clean proof scripts in two flavors.

 * `~/proofs` contains the detailed proof scripts obtained by applying the
   prescribed extraction procedure. The naming scheme is simplified and uses
   TPTP problem identifiers only.

 * `~/clean` presents the same proofs with one small change: in this set of
   files, redundant assumptions have been removed in favor of their
   "clausified" decompositions.

Because Prover9 does not require inputs to be in clausal normal form, not all
of its assumptions are guaranteed to be clauses. Non-clauses are transformed
into clauses via the `clausify` tactic and inserted in the proof scripts as
they are used. This has two consequences. First, non-clausal assumptions are
redundant and will not be considered by the resolution FPC, which accepts
problems in clausal normal form. Second, there is no guarantee that all of the
clauses that result from an application of `clausify` will be used by, and end
up in, the derivation, and therefore the certificate may operate on a
strengthened version of the input formula.

### Certification workflow

The process of certifying a Prover9 proof comprises three main steps.

 1. Extract the signature from the Prover9 proof script. From this, declare
    constructors and auxiliary clauses in the `resolution-elab` module. Map
    each proof step into the sets of clauses and justifications and define an
    `example` instance in `resolution-elab`. If MaxChecker is used, derive
    identifiers and constructors to complete the `Atom` and `Lkf_term` type
    declarations.

 2. Execute `resolution-elab.mod` on the λProlog backend to certify the
    extracted formula with the extracted certificate. The baseline goal
    `check_unordered` checks the certificate as given in the `example`; further
    exploration is possible.

 3. If MaxChecker is used, solve goal `elab_and_export`, create an OCaml
    example with the translated formula and certificate, and run the functional
    checker to obtain an independent verification (i.e., complete the `Test`
    module, build and execute).

The sequence can be fully automated. A translation from Prover9 to either
kernel should generate valid identifiers for each language, and particularly in
λProlog avoid clashes between names, possibly by using a dedicated namespace.
Note that renumbering of clauses may be necessary, given that the FPC numbers
those by order as they are given in the certificate, first the base clauses and
after those the derived clauses; contrast this with Prover9 proofs, in which
assumptions can be introduced at any point in the proof.

The directory `~/transpiler` contains a small program that takes a clean proof
(cf. `~/clean`) and generates code for both λProlog and OCaml kernels.
Identifiers are suffixed `_p9`, thus ensuring the absence of name clashes. To
build it, use the provided Makefile and execute it on a clean proof, as in the
example.

{% highlight bash %}
make
./transpiler.native ../clean/AGT001+1.p9
{% endhighlight %}

In this version, if the translator succeeds, output is given in lines of text.
Each line constitutes a unit of code which can be plugged in its corresponding
module.

 * `type` declarations for λProlog atoms and terms, to be added to
   `resolution-elab.sig`.

 * λProlog auxiliary clauses (`pred_pname` and `size_bool` for atoms,
   `fun_pname` and `size_term` for terms), to be added to
   `resolution-elab.mod`.

 * OCaml atom and term constructors, prefixed `%%atom%%` and `%%term%%`,
   respectively, to be added to MaxChecker's `Atom` and `Lkf_term` modules.

### Teyjus vs. ELPI

The size of the programs generated by translating proofs into λProlog is enough
to expose some behavioral differences between Teyjus and ELPI; ELPI generally
scales better. Here is a summary of these differences.

 * There are moderate limits to the size of the terms Teyjus can parse, both in
   the compiler `tjcc` and in the simulator `tjsim`. While these have not
   impeded the list-based formulation of resolution certificates, exporting and
   importing large proofs and formulas is problematic. This motivates the
   additive composition of elaboration and checking steps seen in
   `resolution-elab`, in which once the unordered certificate is read all
   computation is performed in-memory.

 * The intermediate compilation step in Teyjus, absent from the ELPI
   interpreter, has scalability issues of its own. Compilation times are seen
   to grow substantially once a certain threshold in the size of the proof
   translation is reached. For the very largest examples in the corpus, this
   grows to make Teyjus unusable, to the point of compilation possibly failing
   to terminate.

 * In some of the larger examples, the process of elaboration has been observed
   to surpass the capacity of Teyjus' internal data structures, causing a
   premature stack overflow and a termination of checking. This can be observed
   first when combining elaboration and checking in the `check_*` predicates.

 * Teyjus does not implement a predicate to measure execution times inside the
   language, whereas ELPI reports the execution time of goals by default.
   Therefore Teyjus must rely on external tools and we need find a way to
   separate the time taken to load the program from the proper user time
   required by elaboration and checking operations.

There are observable performance differences between the two systems. Generally
speaking, ELPI runs faster, although the two systems show different patterns of
behavior, especially in the relative cost of running elaboration through paired
certificates, compared with the checking time of each individual certificate.

In its favor, the interface of Teyjus is more complete and more amenable to
scripting. Although workarounds can be found for ELPI, batch reporting in
Teyjus remains more informative.

# Design

## Dependencies

Suppose a program needs to be deployed to a wide range of users. How should the
dependencies be managed?

### Statically Linked

This is the best option if it's available. Suppose a program is created. On one
system, it works fine. On another, it fails. This happens because some
dependencies are different between the systems:

 - if it links against the libraries on the system, then what are the versions
   of those libraries?
 - if it's a script, then what will be interpreting it? what version?
 - if the language imports modules, can these be frozen (like python virtualenv)
   and shipped with the package?

Containerization can help, but has other issues.
  - the user needs a container runtime installed
  - the container might need to be minimalized and hardened depending on
    deployment requirements

This problem can be mitigated by compiling a statically linked binary (check
with `ldd <binary>`). Ship everything the binary needs, and create a build per
platform.

```go
package main
import "os"
func main() {
    // sys call caller logic statically linked in binary. but, still depends on:
    //  - compilation architecture (e.g. arm64)
    //  - operating system ABI (e.g. darwin)
    err := os.Remove("test.txt")
    ...
}
```

### Dynamically Linked

This is required when there are some libraries that are only available on the
target system. For example, graphics related libs are front ends to the GPUs of
those systems.

It can also reduce the size of the binary, but that's typically not worth the
tradeoff.

#### Explicit

Typical dynamic linking is implicit; the [ELF](https://wiki.osdev.org/ELF)
format is loaded and if the libraries aren't available then it will fail to link
at the beginning of the program's execution. This happens automatically.

In contrast, a shared library can be loaded explicitly at runtime via
[dlopen](https://man7.org/linux/man-pages/man3/dlopen.3.html). This provides
more control (what specific symbols are needed and how should failure be
handled) and is useful in a variety of cases:

- JIT compilation
- fallback (link to GPU frontend if available, or fallback to CPU)
- defer; only if necessary then dynamic link. e.g. (trusted) plugin systems

### External Process

Suppose the functionality of a system tool is required, but that tool does not
expose a linkable library. In that case it must be invoked using its command
line interface. Similar drawbacks to dynamic linking but also needs to avoid any
cli related quirks.

```go
package main
import("os/exec";"os")
func main(){
  // relies on command available and on $PATH
  //
  // prevent file name from being interpret as a flag, like name "-f"
  //                      VV
  err:=exec.Command("rm","--","test.txt").Run()
  ...
}
```

## Ergonomics

It should be easy for developers to do what they need to do.

### Clashing

Suppose there is some large tech stack being used; the whole system can't be run
on a single developer's machine. This means that there is some environment which
is shared between developers. There must be an explicit system so that
developers are not interfering with each-other. This could be as simple as a
messaging thread or a lockout system. There should be no ambiguity on what is
currently deployed.

### Debug Cycle 

If a developer needs to test a feature it should be easy to do so. In a perfect
scenario, the developer should be able to wave their hand and receive
instantaneous feedback on their code.

 - the whole stack should not need to be redeployed; modularization should allow
   a "fast path", so one part can be redeployed quickly
 - logs or other feedback mechanisms must be available and easy to use; ideally
   an event streaming service like kafka will allow a developer to view what
   they want and only what they want.

### Documentation

Documentation is multi faceted; each contribute to maintainability:

 - comment are required to explain the context; the notion that code can be self
documenting is [not true](https://queue.acm.org/detail.cfm?id=1053354)
 - tests help maintain backward compatibility since expected behavior is
stated unambiguously
 - modularity and readability: bad code is harder to read than good code
 - README: auxiliary docs forms a directed graph which is fully reachable from
   this entrypoint; explain what the project is, how to use it, where it is
   used, and what it uses

If a newbie can't figure out what something does nor how it fits into the bigger
picture, then that is a huge problem caused by a neglect of documentation. Poor
documentation is technical debt and a barrier to collaboration.

### Errors

Errors shouldn't drop information. A noncompliant example: `Child process
failed: err="exitCode: 1"`.

This reports that there is an error but it doesn't indicate what the underlying
reason for it is. Errors should be like stack traces, propagating information
from callee to caller. In this case, the child process's stderr should be
returned as part of the error message.

## Forward Compatibility

Once something starts getting used everywhere, its design gets locked in place
pretty fast. All the dependees need to be updated. This might not be possible
for on premise deployments. There needs to be a plan to support forwards
compatibility before things go out. Add a version field to schemas and leave
future placeholders in library API.

### Feedback

In order: gather functional requirements, derive design parameters, review and
audit, then implement. Major design changes late in a project interfere with
backwards compatibility.

### Horizontal Scalability

Have a plan for scalability. It could be as simple as hash based sharding.

## Modularity

Loosely coupled components linked by well defined interfaces is a must; bugs can
then be isolated along boundaries (and if a component is bad it can easily be
replaced). If the interface is poorly defined, then all components which depend
on it will suffer.

### Overburdened Source

Suppose there's an application which sends some data to an output. Unfortunately
it exists in a demanding ecosystem; the interface is complicated because it
needs to satisfy everyone's needs and maintain backwards compatibility:

```
$ ./redacted --help
Usage of ./redacted:
  -compatibility-flag-v1
        compatibility mode for client 1 (sends output with other schema)
  -compatibility-flag-v2
        compatibility mode for client 2 (sends output to ingestion server)
  ...
  -compatibility-flag-vN
        compatibility mode for client N (do other things)
  -version
        Prints application version
```

There are ways of managing this complexity:

 - the project could be forked, but now multiple forks need to be kept up to
   date with bug fixes
 - an interface for "send output" could be created, with runtime or compile-time
   flags used to swap out the underlying implementation

But that shouldn't be necessary. There should be an enforced standard for
sending the output in _one way_. It is then the responsibility of whatever sits
on the other side of the interface to do whatever it needs to for that data. If
the project is future proofed (schema versioning + deprecation strategy), then
this compatibility matrix should not be necessary.

### Overburdened Sink

Suppose there are many sources and one sink. This could be over a network, or
just following the flow through some code.

It is the responsibility of each source to coax its data into the form that the
sink is expecting. It is not the responsibility of the sink to mangle everyone's
formats into one form; this causes too much logical burden and interleaving.
Each source only has to handle its domain, so it keeps a clear division of where
logic must be applied; it allows the problem to be modularized as sub-problems
which can be more easily handled individually.

#### Requirements for Centralization

There may be the temptation to centralize some of the logical processing to the
sink. Suppose there is some processing common to all sources. Rather than
repeating logic, put it in
[one](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) place. However,
this can cause problems; there must be the guarantee that the logic will be
applied to _all_ sources, including all reasonable hypothetical ones.

If it only applies to some of the sources, then those sources should explicitly
opt in to that logic on their end! Otherwise, there will be logical scope creep
at the sink. e.g. "This field should be lowercase. One of the sources did not
follow this, so I'll lowercase it for everyone". Now there is implicit
downstream logic which a source is forced to use. And it's muddied the interface
between source and sink.

Repeating code in this way is ok; common code can be managed as a library. Each
source can explicitly opt-in to common processing by importing it.

#### Composition Over Inheritance

The above "Requirements for Centralization" **is the same reasoning for why
inheritance is bad**. Logic should be explicitly opted-in, not implicitly
inherited (and forced); implicit code flow is not readable. Rust's traits (or
equivalent) is the correct way of writing code (having traits which are
implemented by types, with all function calls being explicit). Especially when
hitting something like the [diamond
problem](https://en.wikipedia.org/wiki/Multiple_inheritance), inheritance just
doesn't model the world in a good way.

### Naming

Names should be related to the thing being named. For example a [Class
D](https://www.ccohs.ca/oshanswers/safety_haz/fire_extinguishers.html) fire is
poorly named. A better name would be "Metal Fire". Otherwise there is an
arbitrary mapping which needs to be memorized between the name and the thing
being named.

Code words are used in settings where the layman shouldn't be informed. Like
"code black" in a hospital is said to deliberately obfuscate and prevent panic.
That's not applicable to communication in engineering.

Similarly, although acronyms do have a use, they are a form of obfuscation which
should be avoided.

Sometimes there isn't an obvious name to choose. For example, if naming
operating system versions, the difference between them can be considered an
arbitrary collection of unrelated things which can't be named easily. In this
case consider following [Ubuntu's Naming
Convention](https://en.wikipedia.org/wiki/Ubuntu_version_history#Naming_convention)
(alphabetical code names). Create a unique name that isn't associated to any
other names.

#### Overfitting

Building off of the previous example, a bad name for a "Metal Fire" would be a
"Magnesium or Titanium or Sodium or Potassium Fire". This is over-fitting; there
is a memorization of the information which the thing represents without creating
a simpler abstraction. Individual parts should be harmonized into their shared
identity.

Preventing over-fitting might introduce errors (there are non flammable metals).
But consider that a name lives in a context (it's not useful to communicate the
metals that _are not_ on fire). Otherwise, a balance is struck between an
information dense name and the name's error. Similarly to above, if the only
suitably accurate name is too information dense, then create a unique name that
takes on its own special meaning.

## Parsimony

The simplest solution which fulfills requirements is the correct solution.
Complexity breeds complexity, as it requires a larger mental load to break apart
a complex solution and describe it in a better way.

In the short term, it is always easier to staple a feature on top of an existing
project. It is harder, but sometime necessary, to rethink the existing solution
in a way which better integrates with a feature. Entropy must be actively
opposed.

### Decision Paralysis

Design work necessitates a focus on the relevant subset of
possibilities. Document decisions and move on.

#### Over Abstraction

If there is only one realistic implementor of an interface, then that interface
should not be created; it's just adding extra boilerplate and can hinder
readability. Prefer instead directly calling the functionality.

### Exceptions

Do not use exception. There's many
[pitfalls](https://en.wiktionary.org/wiki/footgun#:~:text=footgun%20(plural%20footguns),shoot%20themselves%20in%20the%20foot.):

 - throwing an exception while unwinding the stack crashes the program
 - the caller might forget to handle implicit control flow (should be explicit)
 - complicates the object lifecycle; creating a [partially
   constructed](https://rollbar.com/blog/throw-exceptions-in-cpp-constructors/)
   object is dangerous
 - don't forget that move ctors should always be noexcept!

Compare the above mental load with how
Rust handles erroneous conditions:

```rust
let thing: Result<Value, Error> = Value::might_err_ctor();
// or
let thing: Value = Value::might_err_ctor()?;
```

That is all there should be! Explicitly, it either returns the constructed
object, or it doesn't. Compared to exceptions, this solution is simpler while
still providing the same functionality.

### Leaky Abstraction

Implementation details shouldn't be exposed. If they are, then they can't be
changed later (someone might be using it!). Plus it makes the interface more
difficult to use. The most useful abstractions handle all cases while requiring
the least amount of effort to use.

### Orthogonality

[Paraphrased](http://erights.org/elib/distrib/pipeline.html): "Modular
abstractions export orthogonal composable primitives."

From the client's point of view, if an idiom is expected, then a specialization
can be created to serve that case. However, this specialization should be
implemented using those same primitives, and to prevent confusion it should be
transparent that it is doing so.

### SQL over NoSQL

NoSQL does have a niche. If the data is truly unstructured then it's
suitable for the task. But verify: is this true? Or has there not been enough
planning on the schemas to be stored. Ensure that it's not "kicking the can
down the road" by dumping whatever data into a NoSQL db, as this complicates
logic in the future. Ensure that the underlying type used to represent the data
is both explicit and the most restrictive model that works.

## Readability

Here a function creates a resty (http) client request and sets a token. The
returned instance can then be used to finalize the details of the request and
send it.

```go
func (r *Redacted) r() *resty.Request {
    r := r.resty.R()
    r.SetCookie(rl.authCookie)
    return r
}
```

This might seem ok at first glance. Maybe it would make sense if this one small
section is repeated numerous times. But otherwise, this function is too small.
Taken to the extreme, the code can become unreadable:

```go
func ToPointer[V any](value V) *V {
    return &value
}
```

This is only a hindrance since now more mental load is required to map to what
the author wrote instead of just writing: `&value`. It's an unwillingness to
adapt to the surrounding code base or ecosystem.

Abstractions should cause less work, not more work. They should map to tangible
steps of a procedure which can be easily thought about. This preserves
readability. 

## State Synchronization

Having multiple states which must be kept in sync is bug prone. In these cases,
there should be an abstraction to prevent error (should set state in
multiple places automatically).

### Documentation

Documentation should be checked into the version control system as it is the
single source of truth. There should not be some auxiliary source of
documentation like a Google Doc or Microsoft Loop (which will then have to be
  kept in sync). They should be checked in and can be written in,
  [Markdown](https://github.blog/developer-skills/github/include-diagrams-markdown-files-mermaid/),
  [LaTeX](https://stackoverflow.com/questions/6188780/git-latex-workflow), or
  even
  [docx](https://stackoverflow.com/questions/22439517/view-docx-file-on-github-and-use-git-diff-on-docx-file-format).

For developers, tools like [Read The Docs](https://about.readthedocs.com/) or
[swagger ui](https://swagger.io/tools/swagger-ui/) allow for documentation to be
generated from the code itself.

### Data Transfer Objects

Suppose there is a codebase which has been fractured into multiple languages.
Along the borders of these programs, data transfer objects (DTOs) are encoded
and sent through the wire. The expected schema on the sending and receiving ends
need to be kept in sync.

Because of the language barrier, it's tempting to simply redefine (copy) the DTO
on both sides (and keep them in sync). A better mechanism defines the type in
only one place, and everywhere that needs it can then import it, or use auto
generate bindings to the language of choice. Ideally the entire code base
consist of one language; lint time feedback shows dependents. It's hard to make
changes, let alone changes which harmonize with the rest of the code base if
it's difficult to find what even uses it!

## Tests

Suppose a mission critical program is being developed. There's plenty of
precautions that can be taken:

 - unit tests (100% code coverage + visualization)
 - [fuzzing](https://en.wikipedia.org/wiki/Fuzzing) to ensure invariants hold
 - [mocking](https://en.wikipedia.org/wiki/Mock_object) frameworks; simulate dependencies
 - formal modeling tools like [alloy](https://alloytools.org/tutorials/online/)
 - static analysis like [semgrep](https://semgrep.dev/) 
 - a quality assurance plan; ideally the requester of a feature should also QA!

However, these precautions should be done only when it makes sense to do so!
This can be modelled as an optimization problem under uncertainty; how can the
most utility be obtained with the available time; should more precautions be put
in place, or should other responsibilities be addressed first. It could make
more sense to trial in the field, since the tester is limited to their own
ability to anticipate issues.

## Time and Space

In theory, the best algorithms have the best time and space complexity. This is widely
understood and is the subject of technical interviews. But understanding the
concepts is different from recognizing where they are applied in the field.

For example, does the whole file really need to be loaded into memory before
use? Or instead, can it be read and sent as output in a continuous stream
processing way with bounded memory. Keep in mind extreme situations which could
fill all available disk / memory.

### Tradeoffs

Suppose a finite random 2D dungeon is being generated with non overlapping
rectangular rooms.

A naive way would guess and check: randomly generate a position for the next
room. If there is overlap, try again. This way is simple but will take an
unbounded amount of time (as it could keep generating an overlapping position,
forever). Even worse, if the dungeon is too dense then the generation algorithm
_will_ get stuck. 

A better way would utilize techniques like [binary space
partitioning](https://www.roguebasin.com/index.php/Basic_BSP_Dungeon_generation)
or [poisson disc sampling](https://www.jasondavies.com/poisson-disc/).

An (arguably) even better way would randomly select from a suitably large set of
pregenerate dungeons. _Whatever gets the job done!_

### Recursion

Avoid recursive functions:

 - accidental infinite recursion -> stack overflow
 - doesn't mesh well with language features like RAII since resources are freed
   at the end of the lexical scope or function (depending on language) which
   happens _after_ return. error prone

All recursive algorithms can instead be written with a stack. Rather than an
implicit stack (each recursive call’s stack frame), use an explicit data
structure (and if it really matters [heap
allocation](https://crates.io/crates/smallvec) can still be avoided).

Similarly, avoid the use of [variable length arrays
(VLA)](https://en.wikipedia.org/wiki/Variable-length_array) as it could cause
stack exhaustion.

## Tool Choice

Shell scripts do have a niche. It can reduce the verbosity of writing a
procedure when compared to os/exec'ing external processes. They are pretty small
and auditable as well (just open up the script to read what it does!).
However...

Shell scripts are highly coupled with a single deployment target.
 - depends on what program (and version!) will be interpreting the script.
 - depends on every single program which is used in the script. What if `curl`
   or `zip` isn't available on a system?
 - needs to be re-written for Windows OS; a translation layer / sandbox like
   Cygwin, WSL, or Git Bash could be used.

Shell scripts are [bug
prone](https://blog.cloudflare.com/pipefail-how-a-missing-shell-option-slowed-cloudflare-down/?utm_source=chatgpt.com/).
Consider setting [strict
mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/):

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'
```

Using the command line for passing input is bug prone.

```bash
rm $FILE        # WRONG. $FILE can expand: `file_name; malicious code here`
rm "$FILE"      # WRONG. $FILE can be a flag like "-f". interpreted by CLI
rm -- "$FILE"   # CORRECT. now check all tools for where this is needed
```

There are safer alternatives:
 - os/exec'ing a subprocess directly passes the arguments to the tool without
being interpreted by the shell.
 - calling the lib API avoids any CLI related quirks (don't forget the "--" arg!)

It _is possible_ to write a shell program of equal quality to other methods,
just the same way that hand written assembly could be written with equal
quality. Evaluate if it is the right tool for the job.

## Type System - Prefer Strong Types

Prefer strong typing over weak typing. It prevents mistakes and is a form of
documentation.

```rust
struct Bytes(pub u32) // NOT BITS
```

Especially when dealing with variance in the data. Only valid options should be
representable (parsimony):

```rust
// don't represent as string!
//  - SHA-256
//  - SHA256
//  - sha 256
//  - SHA2-256
// ...
enum Algorithm {
    SHA128,
    SHA256,
    // ...
}
```

This also has implications for a database; a fixed length type which uses
integer comparison is handled better than a variable length type (memory issues)
which uses a string comparison (slower).

Unrepresentable states should be impossible. This is one of the reasons why C is
bad; it's not (reasonably) possible in the language:

```C
enum ThingType {
    THING_TYPE_INT,
    THING_TYPE_FLOAT
}

union ThingValue {
    int thing_int;     // only if THING_TYPE_INT variant
    float thing_float; // only if THING_TYPE_FLOAT variant
}

struct Thing {
    ThingType type;
    ThingValue value;
}

Thing thing;
thing.type = THING_TYPE_INT;
thing.value.thing_float = 1.0; // OOPS
```

### Errors

If the caller might handle different types of errors in different ways then it
is the responsibility of the callee to identify and enumerate them with an
explicit type representation. Having an opaque error string which must be parsed
for the error reason is not ideal, and if possible the callee should do the work
to determine what the error's type is.

### Value Aliasing

A common anti-pattern is to assign a value with _special meaning_. This is an
implicit representation which is error prone!

For example, suppose the number of things in a shopping cart is being
represented. It is not possible to order 0 things (because then the order would
simply not exist in the first place). One might be tempted to allow 0 to
represent an error state instead.

This is ok but error prone:

```rust
let num_items: u32 = 0;
```

This prevents errors and conveys more information (which the Rust compiler can
use for memory layout optimization):

```rust
let num_items: Option<NonZeroU32> = NonZeroU32::new(0);
```

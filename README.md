<center>

# Developer Guidelines


Broad technical and non technical guidelines. Top level headers organized
alphabetically.

</center>

# Ask

Put in a reasonable effort to understand something. If that doesn't work, ask
for help. It is irresponsible to neglect to use the resources that are
available, including the support of others.

### Low Stakes Questions

From a technical perspective there is a correct way of asking a question;
the details have been narrowed down to exact input and output with a [minimal
reproducible
example](https://stackoverflow.com/help/minimal-reproducible-example) and the
difference between the desired and expected behaviour is contrasted.

That style of question is good for when there is a limited domain and an
_immediate tangible goal_, but it's not the only way of generating value. An
open question like "what does this part do and why is it done this way?" is
great for technical discussion and is a lot less work. Things don't have to be
rigid.

# Decision Paralysis

Design work necessitates a focus on the relevant subset of
possibilities. Document decisions and move on.

### Over Abstraction

If there is only one realistic implementor of an interface, then that interface
should not be created.

# Dependencies - Reduce

Organized from best to worst. In summary, prefer dependency-less code
and avoid shell scripts.

 - Static Link
 - Dynamic Link
 - External Process ([execl](https://linux.die.net/man/3/execl))
 - Shell Scripts

### Static Linked

Suppose a program is created. On your system, it works fine. On a different
system, it fails. It could fail because the shell version is different. Or maybe
an external library is an incompatible version, etc.

Containerization can help, but has other issues.
  - The user needs a container runtime installed
  - The container might need to be minimalized and hardened depending on
    deployment requirements

This problem can be remediated by creating a statically linked program (check
with `ldd <binary>`).

Here's a simple but extensible example. Of note from a shell script point of
view, globbing and other features typically associated with the shell are
available in your language of choice.

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
target system (e.g. graphics related libs). It can also reduce the size of the
binary, but that's typically not worth the tradeoff.

```go
package main // -ldflags="-linkmode=external"

/*
#include <stdio.h>
#include <stdlib.h>
*/
import "C"
import "unsafe"
func main() {
	msg := C.CString("Hello from C!")
	defer C.free(unsafe.Pointer(msg))
	if C.puts(msg) < 0 { /* ... */ }
}
```

Even just this simple step will fail to link on systems with an incompatible
GLIBC version.

### External Process

Suppose you need the functionality of some system tool, but that tool does not
expose a linkable library. In that case it must be invoked using its command
line interface.

```go
package main
import("os/exec";"os")
func main(){
    // relies on command available and on $PATH
	err:=exec.Command("rm","--","test.txt").Run()
    ...
}
```

### Shell

Shell scripts do have a niche. If can reduce the verbosity of writing a
procedure when compared to os/exec'ing external processes. They are pretty small
and auditable as well (just open up the script to read what it does!).
However...

Shell scripts are highly coupled with a single deployment target.
 - Depends on what program (and version!) will be interpreting the script.
 - Depends on every single program which is used in the script. What if `curl`
   or `zip` isn't available on a system?
 - Needs to be re-written for Windows OS. Or a translation layer / sandbox like
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

All of this shell nonsense can be avoided:
 - os/exec'ing a subprocess directly passes the arguments to the tool without
being interpreted by the shell.
 - calling the lib API avoids any CLI related quirks (don't forget the "--" arg!)

Shell scripts are _generally the wrong tool for the job_. It _is possible_ to
write a shell program of equal quality to other methods, just the same way that
hand written assembly could be written with equal quality. Theoretically yes,
but practically no.

# Effort Matching

If something is taking significantly more effort that it should, then it's
likely that the wrong path is being taken. Take a step back and re-evaluate.
Don't "tread the road less traveled", as it doesn't leverage the work that
others have done.

Though, it could be the side effect of a competitive setting, or there could be
[hidden blockers](https://youtu.be/w3MDM0CmG-o). The important part is
recognizing when effort isn't being matched (lest you needlessly exhaust
yourself).

# Ergonomics

It should be easy for developers to do what they need to do.

Suppose there is some large tech stack being used; the whole system can't be run
on a single developer's machine. This means that there is some environment which
is shared between developers (an individual component will be deployed which
interacts with the whole stack). There must be an explicit system so that
developers are not interfering with each-other. This could be as simple as a
messaging thread.

If a developer needs to test a feature it should be easy to do so.
 - There should be no ambiguity on what version is deployed.
 - Redeployment should be fast. Developers work off of rapid iteration, and
   waiting for a CI/CD pipeline lengthens the debugging cycle. The whole stack
   should not need to be redeployed; it should be properly modularized so a
   single part can be swapped out as desired, ideally instantaneously.
 - The logs or other feedback mechanisms must be available and easy to use.
   Ideally an event streaming service like kafka will allow a developer to view
   what they want and only what they want.

# Feedback

Feedback is not a personal attack; it doesn't reflect on you as a person. It is
only reflective of that particular work item. It is a typical part of the
review process - no feedback is abnormal.

Always be open to feedback and discussion on your work. Always try to understand
others' perspectives, and in turn, explain your perspective to them. A long
comment is indicative of _interest_.

In line with the above: when giving feedback, try to be concise and write with a
suggestive or question form. "Have you considered... ?"

# Forward Compatibility

Once something starts getting used everywhere, its design gets locked in place
pretty fast. All the dependees needs to be updated, and good luck if its
deployed on premise. There needs to be a plan to support forwards compatibility
before things go out. This could be something as simple as adding a version
field to schemas or leaving future placeholders in library API.

### Horizontal Scalability

Have a plan for scalability. It could be as simple as hash based sharding.

# Fragmentation

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

This might seem innocent enough. Maybe it would make sense if this one small
section is repeated numerous times. But otherwise, this function is too small.
Good luck reading through code that's littered with tiny functions!
Obfuscation is a pernicious art form.

Here's an extreme example:

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

# Moderation

This document is intended as an augmentation, not a substitution, for your own
thinking. Understand the situation and use your judgment.

# Modularity

Loosely coupled components linked by well defined interfaces is a must; bugs can
then be isolated along boundaries (and if a component is bad it can easily be
replaced). If the interface is poorly defined, then all components which depend
on it will suffer.

### Overburdened Source

Suppose there's an application which sends some data to an output. Unfortunately
it exists in a rigid and demanding ecosystem; the interface is complicated
because it needs to satisfy everyone's needs:

```
$ ./redacted --help
Usage of ./redacted:
  -compatability-flag-v1
        compatability mode for client 1 (sends output with other schema)
  -compatability-flag-v2
        compatability mode for client 2 (sends output to ingestion server)
  ...
  -compatability-flag-v3
        compatability mode for client n (sends output to stdout with other schema)
  -version
        Prints application version
```

Instead, the application should send its output in _one way_. It is then the
responsibility of whatever sits on the other side of the interface to do
whatever it needs to for that data.

### Overburdened Sink

Suppose there are many sources and one sink. This could be over a network, or
just following the flow through some code. The sources all do different things.
They all handle some cleanup of upstream data, maybe some enrichment,
transformation, and processing which is specific to each source itself. 

It is the responsibility of each source to coax its data into the form that the
sink is expecting. It is not the responsibility of the sink to mangle everyone's
formats into one form - this causes too much logical burden and interleaving (in
an extreme case a _machine learning algorithm_ was proposed to do this...). Each
source only has to handle its domain, so it keeps a clear division of where
logic must be applied; it allows the problem to be modularized as sub-problems
which can be more easily handled individually.

### Requirements for Centralization

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

### Composition Over Inheritance

The above "Requirements for Centralization" **is the same reasoning for why
inheritance is bad**. Logic should be explicitly opted-in, not implicitly
inherited (and forced); implicit code flow is not readable. Rust's traits (or
equivalent) is the correct way of writing code (having traits which are
implemented by types, with all function calls being explicit). Especially when
hitting something like the [diamond
problem](https://en.wikipedia.org/wiki/Multiple_inheritance), inheritance just
doesn't model the world in a good way.

# Naming

Names should be related to the thing being named. A violating example: do you
know what a [Class
D](https://www.ccohs.ca/oshanswers/safety_haz/fire_extinguishers.html) fire is?
Of course not! A better name would be "Metal Fire". Otherwise there is an
arbitrary mapping which needs to be memorized between the name and the thing
being named.

Sometimes that doesn't work because there isn't a clearly identifiable division
between the things. For example, if naming operating system versions, the
difference between them can be considered an arbitrary collection of unrelated
things which can't easily be named. In this case consider following [Ubuntu's
Naming
Convention](https://en.wikipedia.org/wiki/Ubuntu_version_history#Naming_convention)
(code names).

# Ownership

Projects should have owners. From a management point of view there is a point
of contact who has responsibility. From a technical perspective context
switching is more work.

That being said, it is _ok_ for there to be overlap; it reduces the [bus
factor](https://activecollab.com/blog/project-management/bus-factor), encourages
team cohesion, and allows people to better leverage their expertise. Ownership
does _not_ mean siloing; helping out across projects should be encouraged as
long as responsibilities aren't neglected. Just ensure the project owner is
apprised.

### Hierarchy

Project owners existing implies an organizational hierarchy. It is the
responsibility of leadership to articulate expectations, and it is the
responsibility of project owners to clarify and _provide feedback_ on those
expectations before implementation.

Generally, functional requirements should flow down the hierarchy and design
parameters should flow up the hierarchy. Assuming specialization, leadership
should not be involved in design parameters and project owners should not be
involved in functional requirements.

# Parsimoniety

The simplest solution which fullfills requirements is the correct solution.
Complexity breeds complexity, as it requires a larger mental load to break apart
a complex solution and describe it in a better way.

In the short term, it is always easier to staple a feature on top of an existing
project. It is harder, but sometime necessary, to rethink the existing solution
in a way which better integrates with a feature. Entropy must be actively
opposed.

### Leaky Abstraction

Implementation details shouldn't be exposed. If they are, then they can't be
changed later (someone might be using it!). Plus it makes the interface more
difficult to use. The most useful abstractions handle all cases while requiring
the least amount of effort to use.

### Exceptions

... are bad. Do not use.

There's tons of [footguns](https://en.wiktionary.org/wiki/footgun#:~:text=footgun%20(plural%20footguns),shoot%20themselves%20in%20the%20foot.):

 - throwing an exception while unwinding the stack crashes the program
 - it is implicit control flow which the caller might not handle (should be explicit)
 - it complicates the object lifecycle. Creating a [partially
   constructed](https://rollbar.com/blog/throw-exceptions-in-cpp-constructors/)
   object is dangerous
 - don't forget that move ctors should always be noexcept!

There's so much nonsense and niche rules. Compare the above mental load with how
Rust handles erroneous conditions:

```rust
let thing: Result<Value, Error> = Value::might_err_ctor();
// or
let thing: Value = Value::might_err_ctor()?;
```

That is all there should be! Explicitly, it either returns the constructed
object, or it doesn't. This is the simplest representation that solves the
problem, so its the correct one.

### SQL over NoSQL

NoSQL does have a niche. If the data is truly unstructured then it's
suitable for the task. But verify: is this true? Or has there not been enough
planning on the schemas to be stored. Ensure that's it's not "kicking the can
down the road" by dumping whatever data into a NoSQL db, as this complicates
logic in the future. Ensure that the underlying type used to represent the data
is both explicit and the most restrictive model that works.

# Reuse Work

Doing something like this (using an established library):

```go
details := auth.NewArtifactoryDetails()
details.SetUrl(url)
details.SetUser(cfg.Username)
details.SetAccessToken(cfg.Token)
serviceConfig, err := config.NewConfigBuilder().
    SetServiceDetails(details).
    Build()
if err != nil { ... }
rtManager, err := artifactory.New(serviceConfig)
if err != nil { ... }
bytes, err := rtManager.Ping()
..
...
```

is WAY easier and WAY safer than constructing the api calls yourself
via manual string manipulation + unmarshalling the rest responses:

```go
url := cfg.BitbucketURL + "/repositories/" + cfg.BitbucketWorkspace
     + "?fields=next,values.name,values.updated_on&sort=updated_on"
if lastCheck != "" {
    url += "&q=updated_on>" + strings.Replace(lastCheck, "+", "%2B", -1)
}
 
req, err := http.NewRequest("GET", url, nil)
if err != nil { ... }
req.SetBasicAuth(cfg.BitbucketUsername, cfg.BitbucketAppPW)

rsp, err := http.DefaultClient.Do(req)
if err != nil { ... }
defer rsp.Body.Close()
if rsp.StatusCode != http.StatusOK {
    ...
}
var buf bytes.Buffer
tee := io.TeeReader(rsp.Body, &buf)
b, err := io.ReadAll(tee)
...
```

# Review with Effort

Good heavens! Here, a function is declared with an argument. The argument is
immediately discarded.

```go
func (r *Redacted) do_something(repo string) error {
    _ = repo
    // ...
}
```

Writing quality the first time is easier then going back and fixing things up.
Fortunately this case is easy to fix (it's known what the original intent was,
and it can be fixed without having unforeseen side effects), but if it can
happen here then it can happen in more difficult cases as well. Take time on
reviews, spend effort to understand the context, and don't approve merge
requests haphazardly!

# State Synchronization

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

### Data Transfer Objects

Suppose there is a codebase which has been fractured into multiple languages. Along the borders of these programs, data transfer objects (DTOs) are encoded and sent through the wire. The expected schema on the sending and receiving ends need to be kept in sync.

Because of the language barrier, it's tempting to simply redefine (copy) the DTO on
both sides (and keep them in sync). A better mechanism defines the type in only one place, and everywhere that needs it can then import it, or use auto generate bindings to the language of choice. Ideally the entire code base consist of one language; lint time feedback shows dependents. It's hard to make changes, let alone changes which harmonize with the rest of the code base if it's difficult to find what even uses it!

# Team Cohesion

Working online is great. There's no need to commute to work. Productivity on an
individual scale increases (the office is distracting). However, team cohesion
suffers. There needs to be an effort to ensure inter and intra team
synchronization. If the left hand doesn't know what the right hand is doing,
then that's not good. From base principals, there should be an understand of how
the company runs, and holistically what clients want.

# Tests Because Tests

Suppose you're developing a mission critical component and you have a ton of
time. In that case it makes sense to add unit tests for complete coverage, and
why not add in some fuzzing and static analysis in there as well. If there's a
network dependency, then use a mocking framework. Ad nauseam. Those
circumstances do pop up, but in a majority of cases it isn't worth it.
Especially if the thing is simple, there's a deadline, and maybe there's just a
bunch of components to get through this week. Practicality gets in the way.

Testing should be done where it makes sense to do so! Anecdotally a
disproportionate push for testing was because _surrounding or previous
components were clearly lacking in quality_. Time might be better spent writing
verifiable quality in the first place:

 - using higher level abstractions (libraries, and reduce manual memory management)
 - rethinking complexity in terms of [axiomatic design](https://en.wikipedia.org/wiki/Axiomatic_design)
 - evaluating surrounding infrastructure (misconfigured devops, etc.)
 - a comprehensive QA list and plan (the requester of a feature should QA!)

One can iterate on a program for a long time (with diminishing returns), but at
some point it makes more sense to expose it to the real world. There are some
things which tests just won't catch.

# Type System - Prefer Strong Types

Prefer strong typing over weak typing. It prevents mistakes and is a form of
documentation.


```rust
struct Bytes(pub u32) // NOT BITS
```

Especially when dealing with variance in the data. Only valid options should be
representable (parsimoniety):

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
    THING_TYPE_INT
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


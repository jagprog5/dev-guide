# Behavioral

## Ask

Put in a reasonable effort to understand something. If that doesn't work, ask
for help. It is irresponsible to neglect to use the resources that are
available, including the support of others.

Don't necessarily spend time to create a [minimal reproducible
example](https://stackoverflow.com/help/minimal-reproducible-example). It's not
possible to know what you don't know, and there may be a better approach. Broad
questions promote technical discussion.

## Communication

Communication should be tailored to the audience; implementation details might
not matter. Functional differences (and how the implementation details relate to
them) do matter.

## Effort Aware Decision Making

If something is taking significantly more effort than it should, then it's
possible that the wrong choices were made. Take a step back and consider the
overall context. Be aware of the sunk cost fallacy.

## Feedback

Feedback is not a personal attack; it does not reflect on the person as an
individual. It is only reflective of that particular work item and is a normal
part of the review process.

The reviewee should remain open to feedback and discussion on their work. Effort
should be made to understand others' perspectives, and in turn, the reviewee
should explain their own perspective. A long comment is indicative of the
reviewer's _interest_ in the subject.

In line with the above: when providing feedback, try to be concise and write
with a suggestive or question form, such as: "Have you considered… ?"

## Inert Client

Don't assume that a user is willing to do more than is necessary. It should
"just work" from their perspective. Exploration is required to determine what
exactly that means.

However, there must be an achievable domain of responsibility. Otherwise you hit
[Zawinski's Law](https://en.wikipedia.org/wiki/Jamie_Zawinski#Zawinski's_Law),
in which software bloats to become universal platforms. Taking on more
responsibility is a tradeoff; there is more outreach but more to maintain.

## Ownership

Projects should have owners. From a management point of view there is a point
of contact who has responsibility. From a technical perspective, context
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
responsibility of project owners to clarify expectations before implementation.

Generally, functional requirements should flow down the hierarchy and design
parameters should flow up the hierarchy (with ample communication throughout).

#### Broken Telephone

There should be as few layers as possible between the developer and the source
of functional requirements. Ideally the developer should experience the context
first hand.

## Presentation

The way that things are said can matter more than how they are said. Be mindful
of professionalism, especially when client facing.

## Qualification

Some domains are inherently difficult (e.g.
[cryptography](https://gotchas.salusa.dev/)). Suppose a message needs to be sent
over an untrusted channel (which requires integrity but not confidentiality).
Someone who is familiar with software but not necessarily cryptography might
naively implement an authentication code as follows (to be sent along with the
message):

`hash( concatenate( secret, message ) )`

If the message is tampered with then the code will not match. Additionally,
since the code is derived from the secret, others can't create their own spoofed
messages. However this is vulnerable to [length extension
attacks](https://en.wikipedia.org/wiki/Length_extension_attack). Suppose the
  receiver uses a string comparison function instead of a constant time
comparison, then this would be vulnerable to [timing
attacks](https://security.stackexchange.com/a/74552).

Developers need general knowledge to know what they don't know and mitigate
potential issues.

## Research

Don't stick to an inferior option because it's familiar. For example, prefer
exploration and find an established library: 

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

The above is significantly safer and easier than constructing the api calls
yourself via manual string manipulation + unmarshalling the rest responses:

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

## Team Cohesion

If the left hand doesn't know what the right hand is doing then that's not
good. There should be an effort to maintain inter- and intra-team cohesion.
Especially for remote or hybrid work; although productivity on an individual
scale increases, team cohesions can suffer if it's not executed correctly.
Implementing a [daily standup](https://www.atlassian.com/agile/scrum/standups)
and infrequent all-hands meetings can help. Just ensure that meetings add value.

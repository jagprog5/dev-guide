# Behavioral

## Ask

Put in a reasonable effort to understand something. If that doesn't work, ask
for help. It is irresponsible to neglect to use the resources that are
available, including the support of others.

Don't necessarily spend time to create a [minimal reproducible
example](https://stackoverflow.com/help/minimal-reproducible-example). It's not
possible to know what you don't know, and there may be a better approach. Broad
questions promote technical discussion.

## Effort Aware Decision Making

If something is taking significantly more effort than it should, then it's
possible that the wrong choices were made. Take a step back and consider the
overall context. Be aware of the sunk cost fallacy.

## Feedback

Feedback is not a personal attack; it doesn't reflect on you as a person. It is
only reflective of that particular work item. It is a typical part of the review
process.

Always be open to feedback and discussion on your work. Always try to understand
others' perspectives, and in turn, explain your perspective to them. A long
comment is indicative of the reviewer's _interest_ in the subject.

In line with the above: when giving feedback, try to be concise and write with a
suggestive or question form. "Have you considered... ?"

## Inert Client

Don't assume that a user is willing to do more than is necessary. It should
"just work" from their perspective. Exploration is required to determine what
exactly that means.

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

## Presentation

The way that things are said can matter more than how they are said. Be mindful
of professionalism, especially when client facing.

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

If the left hand doesn't know what the right hand is doing, then that's not
good. There should be an effort to maintain inter- and intra-team cohesion.
Especially for remote or hybrid work; although productivity on an individual
scale increases, team cohesions can suffer if it's not executed correctly.
Implementing a [daily standup](https://www.atlassian.com/agile/scrum/standups)
and infrequent all-hands meetings can help. Just ensure that meetings add value.

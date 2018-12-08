# Bringing Sanity to your Dependencies with Go Modules and Athens

_Go Advent, Dec. 15, 2018_

As many of us know, Go version 1.11 introduced [Go Modules](https://github.com/golang/go/wiki/Modules), a brand new dependency management system. 

## A Little Bit About Modules

Before 1.11, our dependencies were a collection of Go packages with a single version number attached to it. As packages evolved, their version was changed. Tools like [Godep](https://github.com/tools/godep), [glide](https://github.com/Masterminds/glide), [gb](https://github.com/constabulary/gb) and [dep](https://github.com/golang/dep) made it really easy for us to fetch all the packages that our apps needed, at the versions we need.

Then, Go modules came along and changed the way we refer to dependencies. _Modules_ are a collection of Go packages that can have multiple versions attached to them. Most commonly, these versions follow the [semver](https://semver.org/) format, which is important to the modules tooling.

But, module features don't stop there. Here are a few more cool features that I like:

- It can ingest dependency files from other dependency managers (e.g. `glide.yaml`, `Gopkg.toml`)
- No need to put your code into the `GOPATH` (if you don't want)
- Dependency management is built right into the `go` CLI (i.e. `go get` is now version aware)
- Most importantly for today, modules supports a [download API](https://docs.gomods.io/intro/protocol/)

## What The Modules System Still Lacks

Just like the original `go get` and glide, dep, and friends, Go modules by default fetches code from version control systems (VCS) directly:

![direct from VCS](/img/go-modules-before.png)

The direct-from-VCS approach has served us pretty well since the beginning of dependencies in Go, but it has led to some issues for the community. Here are a few that stand out in my mind:

- When Google Code shut down, we lost some packages that were important to the community
- The [go-bindata package](https://github.com/go-bindata/go-bindata) was removed from GitHub and then pushed to another repository, and everybody who depended it had to create a package alias
- Some modules are massive when you `git clone` them. For example, Kubernetes is over 250M when checked out (using `git clone --depth=1`)

### GitHub Isn't A CDN

The above issues reveal a common underlying problem. Our module _code_ is not separate from module _artifacts_, and since we don't have that separation, we have to use the same systems to develop our modules and to serve those very same modules to users who depend on them.

GitHub and other source code hosting systems are great tools for collaborating on code, making changes, and tracking history. But we've been also relying on the same platforms as highly available hosts for module artifacts, and limitations naturally show. As the community continues to grow, these limitations will come up more and more often.

We need to move from fetching modules directly from VCS:

![go modules before](/img/go-modules-before.png)

To fetching modules from some kind of content delivery network (CDN):

![cdn diagram](/img/go-modules-with-cdn.png)

## The HTTP Download API

The download API is the feature of modules that lets us add a layer of indirection between the VCS and the module consumer (the CDN in the above diagram). In very real terms, we can design an architecture that lets us use the same VCS tools we already use, but we can also build and host _proxy servers_ that implement the API and serve module artifacts to consumers.

It's a pretty straightforward HTTP API that anyone can implement.

## How Athens Fits

At the highest level, Athens is the "CDN" in the above diagram. It implements the HTTP download protocol, stores its dependencies in an immutable database, and fills that database from upstream VCS hosts:

![athens-diagram](/img/athens-diagram.png)

The project started as a few lines of code I wrote one evening to a community of over 50 contributors across 3 continents and, most importantly to me, a nice, supportive and inclusive place in which to participate.

## Experiences with Athens

We've tested Athens out in lots of different situations:

- As a server on `localhost`
- A server just for a team's private code
- A public, global proxy (experimental)

As we've gathered feedback and reviewed our experiences in each situation, we've found some common benefits.

### Builds are Faster

Go modules maintains a local on-disk cache (at `$GOPATH/pkg/mod`), which of course makes builds faster than fetching modules from the network! But, for cold caches (like in CI/CD systems), builds are faster than with fetch-from-VCS situations.

We get speed for a pretty simple reason - Athens serves zip files with module code in them, and no VCS history. It's the same thing as if you ran `git clone --depth 1` and then zipped up the result.

### A Vendor in The Cloud

Athens holds an immutable database, so we're insulated from changes to upstream repositories, even if we delete the `vendor` directory. Simply put, that lets us delete code from our repository while still keeping our builds fast and, more importantly, reliable.

### HTTP Download API

Because the download API is HTTP, we can take advantage of all the battle tested web technologies we know and love. For example, internal teams can load balance across many Athens servers, and we can put a global caching proxy (a proxy in a proxy - inception!) in front of the public proxy.

### Federation

This is my favorite topic because it promotes collaboration and open-ness. The download API is the _lingua franca_ of the Go modules ecosystem and Athens implements it. Anyone can build their own proxy or run Athens themselves and participate in the module proxy ecosystem.

## Get Involved!

We spend a lot of time documenting how to set up and use Athens, so we'd love for you to [try it out](https://docs.gomods.io/install/) and tell us what you think.

If you're interested in contributing, come chat with us in the `#athens` channel on the [Gophers slack](https://gophersinvite.herokuapp.com/), [file an issue](https://github.com/gomods/athens/issues/new/choose), or [choose another way](https://docs.gomods.io/contributing/community/participating/).

We're a nice, respectful, and welcoming group, and [absolutely everybody is welcome](https://arschles.com/blog/absolutely-everybody-is-welcome/) to join us.

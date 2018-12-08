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
- Some modules are massive when you `git clone` them. For example, Kubernetes is over 800M when checked out

### GitHub Isn't A CDN

The above issues reveal a common underlying problem. Our module _code_ is not separate from module _artifacts_, and since we don't have that separation, we need to use the same systems to develop our modules and to serve those very same modules to users who depend on them.

GitHub and other source code hosting systems are great tools for collaborating on code, making changes, and tracking history. But we've been also relying on the same platforms as highly available hosts for module artifacts, and limitations naturally show. As the community continues to grow, these limitations will come up more and more often.

In diagrams, we need to move from fetching modules directly from VCS:

![go modules before](/img/go-modules-before.png)

To fetching modules from a content delivery network:

![cdn diagram](/img/go-modules-with-cdn.png)

## The Download API

The download API is the feature of modules that lets us add a layer of indirection between the VCS and the module consumer. In very real terms, we can design an architecture that lets us use the same VCS tools we already use, but we can also host _proxy servers_ that implement the API and serve module artifacts to module consumers.

It's a pretty straightforward HTTP API that anyone can implement.

## How Athens Fits

I was aware of the limitations of serving module artifacts directly from GitHub based on prior experience, and had been trying to build something like a proxy server for some time. A few years ago, I tried building a git proxy called `goprox` that stored code in an immutable database instead of redirecting to the upstream git repository. That way the code would be available regardless of what happened upstream. It was very buggy but it showed me that proxies could work even before the download API came out :grinning:.

When I first read the vgo design papers - specifically the [article describing the download protocol](https://research.swtch.com/vgo-module) - I saw a great opportunity to build a new kind of proxy server. I build a little server called `vgoprox` one evening, following the concepts I tested in `goprox`, and was very excited to see it working with `vgo` pretty quickly. I shared the code with a few other folks and we renamed it to [Athens](https://github.com/gomods/athens).

At the highest level, Athens is the "CDN" in the above diagram. It implements the HTTP download protocol, stores its dependencies in an immutable database, and fills that database from upstream VCS hosts:

![athens-diagram](/img/athens-diagram.png)

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

We have some interesting challenges in front of us, and can use help in lots of areas. Whether you're an experienced Gopher, just starting, love writing docs, or anything else, we'd love to have you.

We're a nice, respectful, and welcoming group, and [absolutely everybody is welcome](https://arschles.com/blog/absolutely-everybody-is-welcome/) to join us.

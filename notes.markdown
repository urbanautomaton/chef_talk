# Notes

## Outline

1. Introduction
  - Who am I?
  - What's this talk about?
    * Informed by my experience introducing Chef in stages in a small
      team with no dedicated ops people
    * Hoped I could speed things up for other small teams following the
      same route
    * Murray went Steve Ballmer on me - developers! developers!
      developers!
    * So as well as the above, I'm going to try to explain why we as
      developers should be excited about configuration management
  - Who is it aimed at?
    * Small teams with hand-configured servers looking to move up
    * Individual devs looking to gain a better understanding of their
      environment
    * Seen many talks by ops guys selling devops to devs - I'm going to
      try the other way around.
2. Quick Chef overview
  - What is it?
    * Configuration management system, duh
    * Wut?
    * Tool for comparing a system's current state to a desired state,
      and taking it from the former to the latter ("converging")
  - Basics - resources, attributes, convergence
  - Chef solo vs. Chef server
3. What is it to developers?
  - Better understanding of our systems
  - More production-like development environments
  - Ownership of how our code runs when it's live
  - Playing with virtual machines is REALLY COOL
  - Ops is like programming - really?
    * Is it more than fancy scripting?
    * Identifying abstractions
    * Testing!
  - ENOUGH KOOL-AID - Can I set up my laptop with this nonsense?
4. Tools, tips and tricks
  - Pick your platform - Chef should make migration easier, so commit.
  - Virtualisation! Specifically vagrant. Miraculous.
  - Dependency management: berkshelf, librarian
  - Packages: build them yourself with fpm!
5. Conclusions
  - Does Chef live up to its promises?
    * Abstractions: low-level good, higher-level, er...
    * Convergence: configuration drift, confidence in later runs
  - If it doesn't quite, should we still use it?
  - Who *shouldn't* use it?

## Levels of ops

1. hand-configured servers
2. golden image from hand-configured
3. Bunch of setup scripts
4. Some sort of remote automation (capistrano?)
5. Single machine config management
6. Clustered config management

## Pros/Cons of Config Management

+ Repeatability
+ Testability
+ Abstraction / reduce cognitive load
+ Versioning
+ Dependency management
+ Portability / potentially reduced brittleness

- Learning curve
- More dependencies to bootstrap
- Genericity comes at a cost - can you rely on community?

Question is whether the cost of learning the system is offset by the
gains in terms of using the system's abstractions to simplify your
configuration.

## Bootstrapping your laptop

For multi-machine setups, or for regularly-repeated installs, the above
pros are all huge. For a single-machine process you might perform once
every three years, the benefits aren't so clear.

### OSX

* Need to bootstrap with system ruby
* Chef supports ruby 1.8.7 (I think), but do your cookbooks?
* requires a pre-existing toolchain unless you vendor your gems, which
  defeats some of the purpose - from-scratch automation would be nice.
* Iterating on your cookbooks is astonishingly painful
  - Got a spare mac?
  - Got 3-4 hours spare per cycle?
  - Some of the lighter testing tools (minitest-chef-handler?) can
    alleviate this, but still not too much fun
* Homebrew is a valiant effort to bring a sane package system to OSX,
  but realistically you're not getting anything like the control over
  e.g. versions that you do with a proper package manager.
* The chances that what you automate will work on your next laptop are
  very low (although better than those of a simple bash script)

### Linux

* Access to a much wider variety of cookbook
* Potential to provision your workstation with the same recipes as your
  servers
* Portability across common distributions is pretty good for major
  cookbooks
* VM testing is easy so writing your cookbook in the first place will be
  less painful

### Personal thoughts

I've written a basic OSX workstation cookbook, and mostly I don't think
it adds much value that a simple bash script couldn't provide.
Installation of basic toolchain stuff is procedural and uninteresting,
and even the dotfiles resource I wrote seems pointlessly OTT, when it
really only replaces a couple of lines of bash.

Beyond the bootstrap process, the setup and configuration of
dependencies is much flakier, and support for OSX becomes weaker. If
this whole process has drawn me to any conclusion, it's that I'm
increasingly suspicious of developing against homebrew-installed
versions of $SOFTWARE. Versions matter, configurations matter, and 

## Contributing

Chef is open source, as are the cookbooks, and many useful ones are
community-maintained. Contributing to Opscode-maintained cookbooks has
all the effortless charm we've come to expect of the OSS community:

1. Get a Jira account
2. Fill out a contributor's licence agreement
3. Wait for Opscode's legal department to approve you
4. Wait for said approval to attach to your Jira account
5. Open a Jira ticket
6. Open a Github PR, referring to the Jira ticket
7. Mark Jira as "fix provided"
8. Wait...
9. Wait...
10. Get told your fix is unacceptable
11. Fork the community-maintained cookbook

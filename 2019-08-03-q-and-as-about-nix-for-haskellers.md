title = Questions and Answers about Nix for Haskellers
tags = haskell, nixos
summary = Overview of Nix for Haskell development
draft = true

------------------------------------------------------

Last week Dmitrii Kovanikov ([@ChShersh](https://twitter.com/ChShersh)) posted
a [request](https://twitter.com/ChShersh/status/1154782785967542272?s=20) on
Twitter asking if anyone had the time to answer some questions about how to
understand Nix as it applies to Haskell programming.

Dmitrii is working on a scaffolding tool for Haskell projects called
[summoner](https://github.com/kowainik/summoner).  There is an
[open PR](https://github.com/kowainik/summoner/pull/283) adding Nix support.
However, Dmitrii doesn't have any Nix experience, so he'd like to learn more
about using Nix for Haskell development before tackling the PR.

I've been using Nix for about a year and half, both at work and on my own
projects.  I think there is a big lack of documentation surrounding Nix and
Haskell, so I told him I'd be able to answer any questions he had.

He wrote me back with a _great_ list of questions.  I thought it would be an
injustice to the Haskell community if I didn't share my answers, so I decided
to turn his questions and my answers into a an article.

This article is mainly for Haskellers that have heard about Nix (on Twitter,
Reddit, Youtube, etc), know about its advantages, but haven't actually played
around with it.  It is mainly a **high-level overview** of how Nix works with
Haskell.  It is not a hands-on tutorial or walk-through.

The first section of this article is an intro to Nix, followed by answers
to each of Dmitrii's questions.[^7]

## Intro

You may have heard that Nix is complicated and hard to learn.  In my
experience, this is true.

It is similar to Haskell.  The learning curve is steep, but once you get the
hang of it, it is very powerful and flexible.

*What makes it so hard?*

To start off with, Nix refers to two things:

1. A programming language for defining _derivations_

2. A set of command line tools for actually building and installing _derivations_

What is a _derivation_?  You can think of it as something similar
to a "package" in a distribution like Debian or Fedora.

The Nix language provides a way of defining derivations.  The following code
shows an example of defining a derivation for the GNU
[hello](https://www.gnu.org/software/hello/hello.html) package[^1]:

```nix
stdenv.mkDerivation {
  name = "hello-2.1.1";
  builder = ./builder.sh;
  src = fetchurl {
    url = ftp://ftp.nluug.nl/pub/gnu/hello/hello-2.1.1.tar.gz;
    sha256 = "1md7jsfd8pa45z73bz1kszpp01yw6x5ljkjk2hx7wl800any6465";
  };
```

As far as the Nix language is concerned, you can think of a derivation as a
dictionary of key-value pairs.  Some of the keys (and corresponding values) are
considered "special".  For instance, in the above definition, `builder` is a
special key that corresponds to a shell script.  The shell script contains
commands like `./configure && make && make install` that will be used to
build and install the package.

Nix has a few well-known CLI tools.  In this article I will mostly talk about
`nix-build` and `nix-shell`.  `nix-build` would take the above derivation and
actually build it.  `nix-build` uses the "special" keys in the
above derivation to run shell scripts to build and install the binaries, shared
objects, header files, and other files associated with the package.[^2]

For instance, if the above derivation is saved in a file called
`my-first-derivation.nix`, you can build it with the following command:[^3]

```
$ nix-build ./my-first-derivation.nix
```

This produces a directory in the Nix store called something like
`/nix/store/06vykrz1hmxgxir8i74fwjl6r9bb2gpg-hello-2.10`.  This contains, among
other things, the `hello` binary:

```
$ /nix/store/06vykrz1hmxgxir8i74fwjl6r9bb2gpg-hello-2.10/bin/hello
Hello, world!
```

*That doesn't sound too bad.  Where is the difficulty?*

If this was all there is to Nix, it wouldn't be too difficult.  However, it
also wouldn't be that helpful.

The other big part of Nix is a huge set of pre-defined derivations and helper
functions in a repo called [nixpkgs](https://github.com/NixOS/nixpkgs).
This repository contains tens of thousands of derivations, as well as hundreds
of helper functions.  In practice, almost no one uses the bare Nix language by
itself.  Almost everyone uses Nix with nixpkgs.

So what makes learning Nix hard?  In my opinion, it is the following three things:

1.  The documentation for Nix is both terrible and great at the same time.  The
    main documentation for the Nix language and CLI tools is in the
    [Nix manual](https://nixos.org/nix/manual/).  The main documentation for the
    derivations and helper functions in nixpkgs is the
    [nixpkgs manual](https://nixos.org/nixpkgs/manual/).

    For the most part, these manuals are written as reference manuals rather
    than beginner-oriented HOW-TO manuals. As a beginner, it is quite difficult
    to figure out how to do things just from the above manuals.[^4]

2.  Nix is a dynamic lanuage.  Unlike Haskell, you don't get "documentation for
    free" with type signatures.

    There are no tools similar to [Haddock](https://www.haskell.org/haddock/)
    or [doctest](https://github.com/sol/doctest) for Nix.

3.  nixpkgs is a lot of things to a lot of people.  There are a large number of
    people using nixpkgs for widely different things.  There are people who use
    it purely as a repository for system packages (like zlib, glibc, GTK, etc).
    There are people who use it for easily generating a GHC package database
    containing packages like aeson, lens, etc for Haskell development.  There
    are people who use it for doing Python development.  There are people who
    use it for cross compiling ARM binaries on x86.  And many more.

    All of these groups of people have their own code in nixpkgs that others
    often don't venture into.  For example, I've been using Nix for
    about a year and a half, but I don't really have any idea how the
    Python libraries are structured, how cross compiling works, etc.

    If I wanted to play around with cross compiling, it would be time-consuming
    to come up to speed with how everything works.

Learning Nix is a non-trivial endeavour.  However, Nix is extremely flexible.
Like Haskell, I'd say the time-investment can be worth it depending on how
extensively you're planning on using it.

With the above knowledge, it should be possible to dive into the questions from
Dmitrii.

## Questions and Answers

Below, I list Dmitrii's questions and answer them one-by-one.[^7]  These answers
should be understandable by people with no experience using Nix.

### 1. What is the difference between `nix-build` and `nix-shell`?

`nix-build` and `nix-shell` are the two most widely used Nix tools.

They both evaluate derivations in a similar way, but their final results are
different.

When `nix-build` is finished evaluating a derivation, it produces a directory
in the Nix store containing the binaries, shared objects, header files, etc for
the derivation.

When `nix-shell` is finished evaluating a derivation, it launches a Bash shell.
It makes all the inputs to the derivation available in the Bash shell.  For
instance, imagine you had a derivation that depends on `git`.  If you ran
`nix-shell` on that derivation, it would put you in a Bash shell with an
environment where the `git` CLI tools available.  From `nix-shell`,
it should be possible to build the derivation manually (running commands like
`./configure && make && make install`).

By default, if you give no arguments to `nix-build`, it looks for a file called
`default.nix` in the current working directory.  It assumes this file defines a
derivation.  It builds this derivation.

If you give no arguments to `nix-shell`, it looks for a file called `shell.nix`
in the current working directory.  It assumes this files defines a derivation.
It takes all the inputs from this derivation and launches a Bash shell with all
these inputs available.

In later questions, I talk about how `nix-build` and `nix-shell` are related to
doing Haskell development.

### 2. When would somebody want to use Nix instead of cabal or stack?

In general, there are three ways to use Nix to build Haskell projects: using
`nix-build` directly, using `cabal` + Nix, or using `stack` + Nix.

But before I describe each of these methods, I need to talk about Haskell
package sets in Nix.

As explained in the previous section, the Nix language provides a way to define
a _derivation_. A derivation describes the build steps for building a library,
a binary, etc.

The nixpkgs repo provides functions that make it easy to define a derivation
for a Haskell package.

A derivation for a Haskell package might look like the following:

```nix
mkDerivation {
  pname = "conduit";
  version = "1.3.1.1";
  sha256 = "84dfafc92e9553c7bae4b4fe0cba3da29b37def606f88b989db95ee2dc933fa2";
  libraryHaskellDepends = [
    base bytestring directory exceptions filepath mono-traversable mtl
    primitive resourcet text transformers unix unliftio-core vector
  ];
  doHaddock = false;
  doCheck = false;
  homepage = "http://github.com/snoyberg/conduit";
  description = "Streaming data processing library";
  license = stdenv.lib.licenses.mit;
}
```

This is a derivation for building `conduit`.  You can see that it just
duplicates some of the info from the `.cabal` file.[^5]

nixpkgs provides a set of these Haskell derivations for building all the
Haskell packages from Hackage.  nixpkgs provides easy ways of injecting your
own packages into this package set, or changing the versions of packages in
this package set (you can see above that the `conduit` version is fixed to
version 1.3.1.1).  nixpkgs also provides a way to define your own custom
Haskell package set.

The Haskell package set is used in different ways depending on whether you're
using the Nix CLI tools, using `cabal` + Nix, or using `stack` + Nix.  I
describe each of these methods below:

1.  _Use `nix-build` from the CLI to build Haskell projects._

    This method doesn't directly involve `cabal` or `stack`.[^6]

    In this method, the developer generates a derivation corresponding to the
    Haskell package they are developing.  They then call `nix-build` on this
    derivation.  When `nix-build` tries to build the derivation, it gets
    Haskell dependencies from the nixpkgs Haskell package set.  It also gets
    system dependencies (like `zlib` or PostgreSQL bindings) from nixpkgs.

    `nix-build` is not good for interative development, since it doesn't know
    how to only rebuild parts of your code that have changed.  It does a full
    rebuild every time it is run.

    `nix-build` is mainly used in two different situations.  One is by
    end-users who are trying to install an exectuable written in Haskell.
    For instance, if you want to try out the terminal emulator
    [Termonad](https://github.com/cdepillabout/termonad),
    you can just clone the repository and run `nix-build`.  You don't have to
    worry about having `stack` or `cabal` on your system, or making sure any of
    the GTK libraries are installed.  `nix-build` handles everything for you.

    The other situation is in CI.  Using `nix-build` in CI gives extremely high
    confidence that your build is actually reproducible.  In practice this ends
    up being much nicer than directly calling `stack` or `cabal`.

2.  _Use `cabal` + Nix to build Haskell projects._

    This method uses a combination of `cabal` and Nix.

    In this method, Nix is used to build a GHC package database of all your
    Haskell dependencies (as well as their required system packages).
    For instance, if you're developing a Haskell package that uses
    [`postgresql-simple`](http://hackage.haskell.org/package/postgresql-simple),
    Nix will compile `postgresql-simple` as well as its Haskell dependencies
    like `aeson`, `attoparsec`, etc.  It will then produce a GHC package
    database with all these dependencies available.

    Normally, you will run `nix-shell` to get into a Bash shell with this GHC
    package database available.  The shell will have the required system
    libraries available, like `libpq` and `zlib`.  The shell will also provide
    build tools like `cabal` and `ghcid`.

    From here, you can run `cabal` commands like normal.  `cabal` will be able
    to see the GHC package database provided by Nix. `cabal` won't have to compile
    any dependencies when you run commands like `cabal new-build all` for the
    first time.  The only thing it will ever need to build is your own project.

    This method is nice for interactive, everyday development.

3.  _Use `stack` + Nix to build Haskell projects._

    `stack` has built-in support for working with Nix, although it works
    slightly different than `cabal`.

    When using `stack` with Nix, you need to do two things.  One is to pass the
    `--nix` flag to `stack`.  The other is to edit the `stack.yaml` file and
    specify a location of a Nix file containing a derivation.  This derivation
    should contain all the system libraries that are dependencies for the
    Haskell libraries you are using.  For instance, if you are developing a
    package that depends on the `postgresql-simple` package, the derivation
    needs to contain the `libpq` and `zlib` system libraries.

    When you run `stack --nix build`, `stack` internally executes `nix-shell`
    with the derivation you have provided.  It then re-executes itself in this
    new environment.  Now that it is running in an environment with all the
    required system libraries, it starts to compile all your Haskell
    dependencies.  Finally, it compiles your actual project.

    Unlike with `cabal`, `stack` has no way of directly using a GHC package
    database provided by the user.  Unfortunately, there is no way of building
    Haskell packages with Nix and making `stack` use them.

    This method is nice for people and teams already using `stack`, but have
    complicated system dependencies.

    The `stack` documentation has a
    [nice section](https://docs.haskellstack.org/en/stable/nix_integration/)
    about integration with Nix.

If you're looking to use Nix for Haskell development, I recommend setting up
your CI to use `nix-build`, and then using `cabal` + Nix for everyday
development.  If your team is already using `stack`, then using Nix to provide
system dependencies can be a nice way to make your project fully reproducible
(since `stack` only deals with making the Haskell-part reproducible).

### 3. How can I specify the version of Haskell packages to use?

When building with `nix-build` directly, or using `cabal` + Nix, you need a way
to specify to Nix the versions of the Haskell packages you depend on.  For
instance, if you depend on an older version of `aeson`, you need to make sure
that Nix doesn't accidentally use a recent version.

In general, there are two ways to fix Haskell package versions.  One way is to
use the Haskell package set defined in nixpkgs, and overwrite individual
packages that should be different.  The other way is to completely generate your
own Nix Haskell package set.

1.  _Overwrite individual entries in the Haskell package set in nixpkgs._

    As explained above, nixpkgs has a built-in Haskell package set.  In
    general, each package is set to the latest version on Hackage[^8].

    If you need to use an older version of a package, you can easily overwrite
    packages in the nixpkgs Haskell package set.  For instance, if you need to
    an older version of `aeson`, you can just specify it in your Nix files.

    If you're working on a small open source package with few dependencies,
    this is an easy method to implement.

    If you're working on a large proprietary application, this method can be
    quite a lot of tedious work.  You're effectively performing the work of
    the `cabal` solver, only manually.

2.  _Generate your own Nix Haskell package set._

    If you're working on a large Haskell project with many dependencies, it is
    often convenient to generate a Haskell package set for use in Nix.

    There are two main ways people generally generate these Haskell package sets.

    One is by using the output of the `cabal` solver and translating the build
    plan to a set of Nix derivations.  The other is by using a Stackage LTS
    package set and translating it to a set of Nix derivations.

    Generating your own Nix Haskell package sets can make sure that you are
    getting the exact same versions of your Haskell dependencies whether you're
    building with `nix-build`, plain `cabal`, `cabal` + Nix, plain `stack`, or
    `stack` + Nix.  This can make Nix easy to use on teams where some people
    prefer `cabal` (without involving Nix at all), some people prefer `stack`
    with Nix integration, and some people want to run `nix-build` on CI, etc.

    There are tools for generating a set of Nix derivations from either the
    output of the `cabal` solver or a Stackage package set.  The two most
    well-known tools are
    [`haskell.nix`](https://github.com/input-output-hk/haskell.nix)
    and [`stack2nix2`](https://github.com/input-output-hk/stack2nix).

    I wrote up a short
    [comparison](https://discourse.nixos.org/t/backport-ghc-8-6-5-to-19-03/3040/7?u=cdepillabout)
    of these two tools on the offical
    [Nix Discourse](https://discourse.nixos.org).

    Generating your own Nix Haskell package set is often overkill if you're
    only writing a small library, but it works very well if you're on a large
    team or working on a big project.

### 4. Minimum required configuration for Haskell with Nix

_What is the minimum required configuration for a Haskell project to be built
with nix?_

This is a difficult question to answer.  The downside of the flexibility of Nix
is that it provides many different ways to integrate with Haskell projects.

The minimum required configuration changes depending on whether you want to
build purely with `nix-build`, you want to provide a `cabal` + Nix development
environment, you want to allow building with `stack`, etc.  It also depends on
whether you want to use the Haskell package set defined in nixpkgs, or you want to
generate your own Haskell package set (as described in the previous question).

I've setup a
[repo that showcases](https://github.com/cdepillabout/nix-cabal-example-project)
a minimum required configuration for building a non-trivial Haskell project
with both `nix-build` and `cabal` + Nix.  It uses the nixpkgs Haskell package
set.  The README describes how it can be used.

If you want to build with both `nix-build` and `cabal` + Nix, but use a package
set generated from the `cabal` solver, you can use
[`haskell.nix`](https://github.com/input-output-hk/haskell.nix).

If you want to build with `nix-build`, `cabal` + Nix, and `stack`, while using
a package set based on a Stackage LTS release, you should also be able to use
`haskell.nix`.

If you want to build with `stack`, using Nix only for system dependencies, look
at the Stack documentation on
[Nix integration](https://docs.haskellstack.org/en/stable/nix_integration/).

### 5. Difference between `nix-build` and `nix-shell` for developing Haskell projects

_What is the difference between `nix-build` and `nix-shell` with respect to
developing Haskell projects?_

As described above, by default `nix-build` looks for a file called
`default.nix`.  This file should define a derivation.  `nix-build` builds this
derivation.

`nix-build` is used to build a Haskell package.  The resulting Haskell
libraries (shared objects), Haskell executables, documentation, etc are created
in a directory in the Nix store.  The Haskell executables can be run directly,
moved to other systems, etc.

By default `nix-shell` looks for a file called `shell.nix`.  This file should
define a derivation.  `nix-shell` pulls out all the input derivations for the
given derivation.  `nix-shell` launches a Bash shell in an environment with all
the input derivations available.

`nix-shell` is used in two ways for Haskell development.

1.  With `cabal`.

    `nix-shell` compiles all your system dependencies, as well as all your
    Haskell package dependencies.  It launches you into a Bash shell with
    access to a GHC package database containing of all these dependencies.  If
    you run a command like `cabal new-build all`, `cabal` will be able to see
    the GHC package database.  `cabal` will not have to compile any of these
    Haskell dependencies.

2.  With `stack`.

    This method uses `nix-shell` to provide system-level dependencies, while
    using `stack` to compile all Haskell packages.

    When running `stack --nix build`, `stack` internally executes `nix-shell`,
    and re-executes itself inside that environment.  This is described in more
    detail above.

A third possibility would be to use `nix-shell` to provide system-level
dependencies, but use `cabal` to compile all Haskell packages.  This would be
similar to how Stack's Nix integration works.  This method is not often used.

### 6. default.nix, shell.nix, and release.nix

_I've heard about some common Nix files: default.nix, shell.nix, and
release.nix. What is the purpose of these files?_

As described above, by default, `nix-build` reads `default.nix`.  `nix-shell`
reads `shell.nix`.

As for other commonly seen files, like `release.nix`, `nixpkgs.nix`, etc,
none of them have any special meaning.  There is no standard for what to name
these other files.  There are no widely-followed best practices reguarding
naming.

That said, most repositories meant to built with Nix have at least three files.
The first two files are `default.nix` and `shell.nix`. The third file often
defines the nixpkgs version you want to use, as well as an overlay that defines
your own packages.

This isn't super important to understand right now, but you can see an example
of it in action in the
[example repo](https://github.com/cdepillabout/nix-cabal-example-project).
I've called this third file
[`nixpkgs.nix`](https://github.com/cdepillabout/nix-cabal-example-project/blob/master/nixpkgs.nix).

### 7. What is the recommended workflow when developing packages with nix?

The recommended workflow depends on whether you want to use `cabal` or `stack`.

1.  With `cabal`.

    First, you run `nix-shell` to get into a Bash shell.  This Bash shell has
    tools like `cabal-install` and `ghcid` available.  There should also be a
    GHC package database with all your Haskell dependencies available.  From
    this Bash shell, you should directly be able to run commands like the
    following:

    - `cabal new-build`
    - `cabal new-test`
    - `ghcid -c 'cabal new-repl'`

    You can play around with the
    [example repo](https://github.com/cdepillabout/nix-cabal-example-project)
    for an idea of what this workflow feels like.

2.  With `stack`.

    You can directly run commands like the following:

    - `stack --nix build`
    - `stack --nix test`
    - `ghcid -c 'stack --nix repl'` (given that you already have `ghcid`
      available in your environment)

    In the above commands, `stack` makes sure to re-exec itself in an
    environment with all your required system libraries.

    See Stack's
    [Nix integration](https://docs.haskellstack.org/en/stable/nix_integration/)
    documentation for more information.

When building on CI, you should set everything up so your Haskell package is
built with `nix-build` directly.

### 8. Haskell + Nix uses too much disk space?

_I know that nix has its own store for storing packages. Is it possible to
configure it to share the store for Haskell packages with cabal
(`~/.cabal/store`) and stack (`~/.stack`)? I'm building with both build tools
and caches for them eat a lot of my disk space. I don't really want to
duplicate every Haskell package 3 times for nix as well._

This is a tricky question.  The answer depends on the following questions:

1.  Do you want to use `stack` or `cabal`?

2.  If you answered `cabal`, then do you always want to get Haskell dependencies
    from Nix?  Or do you want to build some packages fully with `cabal`, and
    some packages with `cabal` + Nix?

If you are content with doing all development with `stack`, then you can
continue to use `stack` as normal (with or without the Nix integration).
All the Haskell dependencies `stack` builds will go under `~/.stack` like
normal.

There is no way to get `stack` to use Haskell dependencies built with Nix.

If you want to use `cabal`, then you have to decide whether you want to use
`cabal` + Nix for some projects and use normal `cabal` (without Nix) for some
projects, or you want to use `cabal` + Nix for ALL projects.

If you want to use `cabal` + Nix for ALL projects, then in theory you can keep
nearly all Haskell dependencies in the Nix store.  However, if you use
different versions of nixpkgs to get Haskell dependencies for different
projects, or if you use a tool like `haskell.nix` to generate a custom Haskell
package set, then it is possible for many duplicate Haskell packages to end up
in the Nix store.  This can eat up disk space.

If you want to use `cabal` + Nix for some projects and normal `cabal` (without
Nix) for some projects, then you will end up with duplicate Haskell packages in
the Nix store and `~/.cabal/store`.

If you want to use a combination of Nix, `cabal`, and `stack`, then you will
end up with a bunch of duplicate Haskell packages.

The only solution to this problem is to use `cabal` + Nix while getting all
your Haskell dependencies from the same Nix Haskell package set for all your
projects.  Unfortunately, this is often not a reasonable solution.

### 9. What is the best way to learn Nix for Haskell development?

If you want to use Nix for Haskell development, I recommend reading the
resources below in the order they are listed:

1.  [Nix Pills](https://nixos.org/nixos/nix-pills/)

    This is a long-form walk-through on how Nix works.  It doesn't have
    anything directly to do with Haskell, but it brings you up to speed on Nix.

    This is quite long.  Expect to spend 10 to 20 hours working through this.

    This may sound like a lot, but keep in mind that Nix is very similar to
    Haskell.  If you are a Haskell beginner (and Haskell is your first
    statically-typed functional language), you can't dive right into writing web
    apps.  Instead, you have to slowly and methodically study the language.
    After you have enough of the beginner-level info under your belt, then you
    can work on harder things like web apps.  Nix is the same.

2.  [Gabriel Gonzalez's haskell-nix walk-through](https://github.com/Gabriel439/haskell-nix)

    This more-or-less takes up from where the Nix Pills leave off.  It is a
    beginner-oriented walk-through for how to use Nix for Haskell development.

    Personally, I don't like some of the decisions they made with how to
    present Nix for Haskell development, but it is currently the best
    beginner-oriented tutorial available.

3.  [nixpkgs manual on Haskell development](https://nixos.org/nixpkgs/manual/#users-guide-to-the-haskell-infrastructure)

    As explained above, the nixpkgs manual has many problems.  It is somewhat
    hard to understand unless you've already read the Nix Pills and Gabriel's
    intro.

4.  [Nix manual](https://nixos.org/nix/manual/) and [nixpkgs manual](https://nixos.org/nixpkgs/manual/)

    These are not required reading, but you may want to skim through them.  The
    more you understand Nix, the more helpful the manuals become.

Once you've read the first three resources (and possibly the fourth), you
should be ready to start using Nix for Haskell development.

I recommend disregarding the advice from both the haskell-nix walk-through and
the nixpkgs manual.  Instead, you should setup your repos similar to how the
[example repo](https://github.com/cdepillabout/nix-cabal-example-project) is
structured.

For larger projects, you should use a tool like
[`haskell.nix`](https://github.com/input-output-hk/haskell.nix) or
[`stack2nix`](https://github.com/input-output-hk/stack2nix).

### 10. Should Haskell beginners learn Nix?

**No**.

If your goal is to learn Haskell, you can get by just fine with `stack` or
`cabal`.[^9]  Nix requires a large upfront investment in time, and it doesn't
directly help you on your quest to learn Haskell.  You should stay away from
Nix, at least for now.

If you are an intermediate to advanced Haskeller and are looking for a powerful
and flexible way to deal with dependencies, then investigating Nix might be a
good choice.

If you're not particularly interested in Haskell, but are looking for a novel
Linux package manager, then checkout Nix!

## Conclusion

Nix can be nice to use for Haskell development.  Being so flexible, it provides
many different ways to integrate with common Haskell tools.  However, it has a
steep learning curve, so the decision to plunge head-first into Nix should not
be made lightly.

## Footnotes

[^1]: This example has been simplified.  It won't work as-is.  If
    you're interested in playing around with this, I recommend you read the
    relevant section in the
    [Nix manual](https://nixos.org/nix/manual/#ch-simple-expression).

[^2]: This quite a big simplification of what actually happens, but it is a
    reasonable first approximation.

[^3]: Again, this won't actually work as-is.  It is mostly to show the relation
    between the Nix language derivations, and the Nix CLI tools.  If you're
    interested in actually learning to build your own derivations, I recommend
    reading through the informative
    [Nix Pills](https://nixos.org/nixos/nix-pills/).

[^4]: One of my friends, [Dave Della Costa](http://davedellacosta.com/), wrote
    an article about
    [getting into Nix as a new user](http://davedellacosta.com/posts/2019-03-29-why-nixos-is-hard-and-how-to-fix.html)
    He talks about what makes Nix difficult for a new user.  I strongly agree
    with most of the points he makes.

[^5]: For the most part, it only duplicates the information important
    for figuring out the dependencies of `conduit`.  For instance, `conduit`
    depends on the `resourcet` Haskell package.  When trying to build the
    `conduit` derivation, the Nix CLI tools need to know that first the
    `resourcet` derivation should be built.

[^6] The `Cabal` library is used directly by the build process internally.

[^7] Some of the questions are from me, not Dmitrii.  I added them to try to
    make this article easier to read from top to bottom without needing to
    jump around.

[^8] This isn't completely true.  The actual setup is a little more complex.

    nixpkgs generally tracks the most recent Stackage LTS release.  At the time
    of this writing, the most recent Stackage LTS release is LTS-13.30.  This
    contains, for example,
    [`servant-0.15`](https://www.stackage.org/lts-13.30/package/servant-0.15),
    even though the most recent release on Hackage is
    [`servant-0.16.1`](https://hackage.haskell.org/package/servant-0.16.1).

    The `servant` derivation in nixpkgs currently refers to `servant-0.15`.  However,
    there is also a separate derivation called `servant_0_16_1` that refers to
    `servant-0.16.1`.

    The reason that nixpkgs tracks the most recent Stackage LTS release, and
    not the latest current version on Hackage, is to hopefully increase the
    number of Haskell packages that successfully compile with each other.

[^9] I'd recommend `stack` over `cabal` for most beginners.  `stack` is often
    easier to get things working.

    However, some people strongly dislike `stack` and swear by `cabal`.  Your
    mileage may vary.

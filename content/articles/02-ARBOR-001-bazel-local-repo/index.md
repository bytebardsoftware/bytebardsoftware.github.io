+++
title = 'ARBOR 001 - The Bazel Local Registry Pattern for Large Monorepo-shaped Polyrepos'
date = 2026-05-24T08:30:00-07:00
draft = false
tags = ["Bazel", "Monorepos", "Legacy", "ARBOR"]
+++

_Short for "Approaches to Refactoring Big Old Repositories", the ARBOR series covers the techniques I've found and implemented when working on large project repositories, typically during a migration into Bazel. I do not claim these techniques are Bazel-idiomatic, but rather "as idiomatic as possible while working with the unique constraints of particular legacy project". If your project has a similar set of constraints, maybe they'll be useful!_

In this article, I'll cover how I stood up a custom Bazel registry, why I did it in a subdirectory of a monorepo, and why I think that madness was a good idea in the first place.

You might find this pattern useful if you're migrating a project that:
- spans many repositories that need to build independently,
- but those repositories depend on each other,
- and don't want to push any of them to the BCR.

But first, an introduction. I'm Borja, I've been working with Bazel for almost a decade in large companies like Twitter and Apple, and now I help organizations solve their Bazel woes. If you have Bazel problems, [drop me a line](https://bytebard.software/enquire/?utm_campaign=blog-arbor-001)!

Anyway, here's the story:

# Setting The Stage

_This background is important to understand the tradeoffs that led to this design. But if you know it already, or just don't care, you can skip to the implementation by clicking here._

The SONiC project is fascinating -- it's a complex set of tools and services that get loaded into high-frequency network switches -- you know, the ones Google, Meta, Microsoft and all the other hyper-scalers use to route traffic inside their massive data-centers. It's an amazing project, it shows how a huge, disjoint industry of competitors can come together to cooperate on a problem. It has also grown a lot in different directions, with contributions from dozens of companies across decades.

Through my relationship with Aspect Build, I found myself working on modernizing the Make-based, homegrown build system of the SONiC project into Bazel. You can read more about our work in [their blog](https://blog.aspect.build/bazel-for-sonic), or watch my recent presentation to the SONiC community:

![](https://www.youtube.com/watch?v=uSKCNDWuXjc)

## The Unique Constraints of the SONiC Build System

The SONiC build system is unique and wonderful because it has to accommodate a wide array of unusual architectures. This presents a lot of Bazel challenges, and required a lot of rule development work. In this article, we will ignore all of that[^1].

Instead, we're going to focus on the outer shape of the project, and how its repositories relate to one another:

SONiC is made up of **components**. For our purposes, a component is "a service or tool that will be deployed to the switch in a Docker container". A **component** can come from different **sources**. Typically, these sources will be either a (possibly patched) version of a third-party open-source utility (e.g. Redis), or a first-party component in its own repository (e.g. https://github.com/sonic-net/sonic-swss). 

There is a central repository,[ **sonic-buildimage**](https://github.com/sonic-net/sonic-buildimage), which (through a hand-written Make-based build system) imports these **sources** and bundles them into Docker images ready to be deployed to switches. First-party code is imported via git submodules ([example](https://github.com/sonic-net/sonic-buildimage/tree/master/src)), whereas third party code is downloaded from a Debian repository before being patched and recompiled ([example](https://github.com/sonic-net/sonic-buildimage/blob/master/src/libnl3/Makefile)).

We're always expected to build deployable artifacts from the **sonic-buildimage** repository. However, we do want every component to be buildable on its own so that we can iterate on it quickly.

So far, this is easy to model in Bazel, right?

If we assume that we're always going to check out **sonic-buildimage** with all its submodules, we can more or less treat it like a monorepo: For first party components, we import them via a `git_override` or `local_path_override`. Job done.

Third party components are a bit trickier: We do have to migrate them to Bazel from their Debian builds, but in practice most were either already in the BCR (like `flex` and `bison`), or were relatively stable and easy to migrate at a minimal maintenance cost (e.g. [our port of libnl3](https://github.com/thesayyn/sonic-buildimage/tree/master/src/libnl3)). When we're done, we create a big `sonic-buildimage/MODULE.bazel` file with all that hard work and we're done, right?

Well... not quite.

Turns out, first party components can depend on each other. And neither `git_override` nor `local_path_override` are transitive. So, if `sonic-swss` depends on `sonic-swss-common`, which depends on `sonic-buildimage/src/libnl3`, then `sonic-swss` _had_ to have either a `git_override` or a `local_path_override` for `sonic-buildimage/src/libnl3`.

Oops.

Suddenly, even the smallest MODULE.bazel file ended up being around a hundred lines long. Migrating a new component meant touching a couple dozen files just to make sure everyone was up to date. If we patched a third-party package, the patches had to travel across all the transitive dependency chain. Not great.

Even worse: Most SONiC users have extremely custom setups. Almost everybody has forked _at least_ `sonic-buildimage` and several components. Therefore, when _they_ try to consume these changes, they may have to touch _all_ the components to point them to the right forks. Double not-great.

Enter, the Bazel Local Registry.

# The Bazel Local Registry: An alternative to the Bazel Central Registry

At its core, the idea behind this is very simple: We'd like the ability to perform dependency resolution across Bazel modules (i.e. to use `bazel_dep`), but our code is not really useful to others, so it doesn't make sense to contribute it to the BCR.

Well, why not stand up our own little BCR? Like, a _local_ version of the BCR. Like, _Bazel Local Registry_.

If we have a Bazel registry that we (the project, as in "The SONiC Project") own, our problems are solved:
- `MODULE.bazel` files are slimmed down: Every component needs only its first party dependencies as `bazel_deps`, and uses the Bazel Local Registry to resolve its transitive deps.
- Third party dependency patching is trivial: Host the patches in the registry, right beside the Bazel module declaration. 
- Updates are trivial: Instead of modifying every component, we just have to update the version in the registry and bump versions in the components that need the changes--the bzlmod resolution algorithm will do the rest.
## Implementation

The implementation is a simple as the idea. In fact, bzlmod was designed from the start to support this use case[^2]. 

A Bazel registry is just a directory with the right shape. And the right shape is:

```shell
$ tree -L1
.
├── bazel_registry.json 
└── modules
    ├── <dependency 1>
    ├── <dependency 2>
    ├── ...
```

The `bazel-registry.json` file contains just the `{ "module_base_path": ...}` JSON, where `module_base_path` is the root from which we can refer to modules (it becomes clearer later). The `modules` subdirectory contains one subdirectory per Bazel module we want to import into the repository, with one subdirectory further per version. In the BCR, for instance, version `9.6.1` of `rules_java` is placed under `<root>/modules/rules_java/9.6.1`. More information in the [official documentation](https://bazel.build/external/registry#index_registry).

> _SONiC Example_
> 
> In SONiC, we placed this directory in `sonic-buildimage`, because the community was already used to the flow of "check out the entire build repository, and build deployment artifacts from there". See it [here](https://github.com/thesayyn/sonic-buildimage/tree/master/tools/bazel/registry).

Now we're ready to start adding Bazel modules for our different components. In general, you should be fine following the [documentation on registries](https://bazel.build/external/registry#index_registry) and looking at the BCR for examples. However, there are a couple of tricks that are likely not explored in the BCR, but will make your life a lot easier. the BCR's guidelines on adding new modules.
### Iterate Quickly on First Party Dependencies

This only works if your dependencies are checked out at a stable path, ideally relative to the `module_base_path` we mentioned above. In those cases, you can create a Bazel package that will point to the local directory where the dependency will live. For that, you create a version of the dependency as normal, but then set its `type` to be `local_path`, as per [the documentation](https://bazel.build/external/registry#source-json):

```json
{
  "type": "local_path",
  "path": "path/under/module_base_path"
}
```

Then, symlink the real `MODULE.bazel` file into the Bazel Local Registry Entry:
```shell
$ ls path/to/registry/modules/mydep/0.0.0/MODULE.bazel
path/to/registry/modules/mydep/0.0.0/MODULE.bazel -> /home/blorente/code/github.com/myorg/mydep/MODULE.bazel
```

As a consequence, changes in the local source of `mydep` are reflected immediately on every dependency that consumes `mydep@0.0.0`, without having to modify the Bazel Local Registry.

> _SONiC Example_
> 
> SONiC depends on a patched version of `libnl3`, which lives in `sonic-buildimage`, under `sonic-buildimage/src/libnl3`.
> So we added it as a `local_path` as above, and symlinked `sonic-buildimage/.../registry/modules/libnl3/3.7.0/MODULE.bazel` to the actual `MODULE.bazel` file in `sonic-buildimage/src/libnl3`. See the code [here](https://github.com/thesayyn/sonic-buildimage/tree/master/tools/bazel/registry/modules/libnl3).

### Easily Maintain Patched Versions of Third Party Dependencies

Sometimes, you need to keep a patched version of a third party dependency around. Maybe you need to add support for a proprietary platform. Maybe you've added features that they don't want upstream, or maybe the patches you need are just required for the Bazel build. Whatever the case, you have a bunch of patches, and they're going to be around for a while, even as new versions of the dependency keep coming out.

This problem is easy to solve in a Bazel Local Registry: Simply create patched versions of that dependency, and host the patches in the registry:

```shell
$ tree modules/rules_go/0.60.0.patched
modules/rules_go/0.60.0.patched
├── MODULE.bazel
├── patches
│   ├── 0001_add_extra_taco_sauce.patch
│   ├── 0002_even_more_taco_sauce.patch
│   └── 0003_despond_the_combobulators.patch
└── source.json
```
Anyone in your first party dependencies that depends on `rules_go` should then depend on the patched version:

```starlark
# my_dep/MODULE.bazel

bazel_dep(name="rules_go", version="0.60.0.patched")
```

It's easy to see how this can be enforced automatically by pre-commit hooks or [AXL](https://github.com/aspect-extensions) rules, so I'll leave that as an exercise to the reader.

> _SONiC Example_
> 
> That example above is taken almost verbatim from SONiC's [`rules_go` entry](https://github.com/thesayyn/sonic-buildimage/tree/master/tools/bazel/registry/modules/rules_go). We stored patches that we mean to upstream, but either haven't merged yet, or haven't been released yet.

# Tradeoffs & Alternatives

As with any engineering, and _especially_ with software patterns, it's vital to discuss tradeoffs before adopting them. Otherwise, we run the risk of introducing complexity just for the sake of complexity.

On the bright side, the Bazel Local Repository pattern is a great stepping stone for a Bazel migration like [the one I described](#setting-the-stage). It makes dependency resolution between different parts of your project easy. It makes maintaining long-term patches trivial, and it offers a clear path forward towards more idiomatic practices. For instance, the local repository could be moved wholesale into a community-owned Bazel Registry, which could then, with time, have pieces of it split off to the BCR without having to change a single line of code in the consumers.

However, it also means that:
- We lose some Bazel goodness. Because `local_path` modules are backed by `local_repository` repo rule, we have found that sometimes we had to run `bazel shutdown` to pick up changes to a local dep. I believe this is solvable, but I haven't solved it yet.
- Individual components now depend on the repository with the Bazel registry. This is not a problem if your project is roughly monorepo-shaped (like SONiC), but the less you can rely on your checkout of the registry to be in sync with your components, the more painful your life will be. If this is your case, you may be better off avoiding `local_path` sources in your Bazel repository entirely, and just relying on traditional `archive` paths.

# Conclusion

I feel like 2.5k words is too many to explain such a simple idea, but I hope at least some of it has been helpful to you. This is not a pattern to use often, but it has definitely been the right decision for SONiC.

That's it for now.

If you found this helpful, or you like the idea of the ARBOR series, or you'd like to see some of your patterns featured here, or anything else, please feel free to reach out at borja@bytebard.software! Hearing from folks would be the best motivation to keep writing these.

Also, if you have Bazel problems that you'd like help with, [let me know](https://bytebard.software/enquire/?utm_campaign=blog-arbor-001)! Chances are I can help.

-- Borja

[^1]: See the [video](https://www.youtube.com/watch?v=uSKCNDWuXjc) or [read the article](https://blog.aspect.build/bazel-for-sonic) if you're interested in learning about them

[^2]: See https://bazel.build/external/registry

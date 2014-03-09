---
title: Composable builds for on-premise products
layout: post
published: false
---

I've been invited to talk about how we handle builds for GitHub Enterprise at the next RubyConf Philipines. I thought that, in preparation for my talk, it would be also a good idea to write about those topics here.

The Enterprise builds consisted on two scripts, when I joined GitHub two years ago. The first one built our Ubuntu base image, and the second one built the package that included our apps. We executed these scripts on a remote server via Hubot. I'm going to focus this first post on one of the main problems these scripts had. They were really hard to maintain and customize for further needs.

The basic command that we use to build packages looks like this:

`hubot build ghp VERSION`

This sent a request to the server to run the build script for the package. That command also accepted several flags to build packages for topic branches. More often than not we ended up running something like this:

`hubot build ghp VERSION --github-branch TOPIC --cookbooks-branch TOPIC --gist-branch TOPIC...`

Underneath, the script was responsible of several things:

1. Configure the environment for the current build.
2. Pull each project and topic branch that Enterprise needs to run.
3. Build debian packages for each one of those projects.
4. Pack all debian packages into a single deliverable.
5. Upload that deliverable to our storage server.
6. Notify the person that ran the command.

As you can imagine, those are a lot of responsibilities for a single script. We decided to rewrite the script and the api and start from scratch. Entering composable builds.

The first thing we did was to separate every one of those steps into smaller meaningful pieces. We followed a pretty common pattern, the middleware pattern. We called each one of those pieces "Conduit". For instance, the step to build a debian package looked like this:

```ruby
class BuildGitHubDeb
  def run(payload)
    # call brew2deb to generate a debian package
    payload['github_deb'] = debian_path
  end
end
```

And the steps to configure our cloud service looked like this:

```ruby
class SetupCompute
  def run(payload)
    payload['compute'] = Fog::Compute.new(...)
  end
end

class SetupStorage
  def run(payload)
    payload['storage'] = Fog::Storage.new(...)
  end
end
```

As you might have guessed already, every conduit shares that `payload` argument. So, the final conduit that packs the debian files into the tarball looks like this:

```ruby
class BuildGhp
  def run(payload)
     debs = payload.each_with_object([]) do |(k, v), debs|
       debs << v if k =~ /_deb$/
     end
     # call tar with all the deb paths
  end
end
```

We compose what we call `Pipeline` putting together several of these conduits:

```ruby
BuildEnterprisePackage = Pipeline[
   SetupCompute,
   SetupStorage,
   ...
   BuildGitHubDeb,
   ...
   BuildGhp,
   NotifyCampfire
]
```

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
    storage = Fog::Storage.new(...)
    
    payload['storage'] = storage
    payload['storage_directory'] = storage.directory("enterprise")
  end
end
```

As you might have guessed already, every conduit shares that `payload` argument.

The conduit that packs the debian files into the tarball looks like this:

```ruby
class BuildGhp
  def run(payload)
     debs = payload.each_with_object([]) do |(k, v), debs|
       debs << v if k =~ /_deb$/
     end
     # call tar with all the deb paths
     payload['ghp_path'] = tar_path
  end
end
```

And the conduit that uploads the tarball to our internal storage looks like this:

```ruby
class UploadGhp
  def run(payload)
    version = payload['version']
    
    file = payload['storage_directory'].files.create \
      key: "github-enterprise-#{version}",
      body: File.open(payload['ghp_path'])
    
    notify "Package #{version} uploaded to #{file.public_url}"
  end
end
```

We compose what we call a `Pipeline` by putting together several of these conduits:

```ruby
BuildEnterprisePackage = Pipeline[
   SetupCompute,
   SetupStorage,
   ...
   BuildGitHubDeb,
   ...
   BuildGhp,
   UploadGhp
]
```

By separating each concern completely we got way more simple snippets of code. Now, we can reuse them to extend our tools beyond the two initial scripts we had with everything coupled.

In the next post I'll explain how we handle the expectations for each conduit and the flags that we provide via our chat clients.

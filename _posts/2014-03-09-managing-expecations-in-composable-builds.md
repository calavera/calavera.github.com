---
title: Managing expectations in composable builds
layout: post
---

In my [previous blog post](http://calavera.github.io/2014/03/09/composable-builds-for-on-premise-products.html), I revealed how we separated two monolithic scripts into small reusable pieces. In this one, I'll explain how we handle the expectations for each one of them.

Like you probably remember, we call the pipelines via hubot. A normal call looks like this:

{% highlight bash %}
hubot build ghp VERSION --github-branch TOPIC
{% endhighlight %}

Each one of those commands can have several flags. The payload that we pass to each conduit stores those flags. So, we can write something like this, if a conduit needs to know the github branch to build:

{% highlight ruby %}
class BuildGitHubDeb
  def run(payload)
     branch = payload.fetch('github_branch', 'master')
     payload['github_deb'] = build_deb('github', branch)
  end
end
{% endhighlight %}

Internally, we normalize each flag name to always use underscores rather than dashes. That way, the behavior is predictable. You always get flag values from the payload using underscores.

There was only one problem with managing keys and values in the payload. We couldn't guarantee that the payload contained everything a conduit needed to run once. Especially, because some conduits required information only provided by other conduits.

We added some sanity checks to the pipelines. Before running, they gather information about each conduit and decide what to do. At the same time, each conduit needs to specify their expectations. They also need to specify the information they provide to future conduits.

Following the previous example, this is what the conduit that builds the github package looks like:

{% highlight ruby %}
class BuildGitHubDeb
  expects 'version'
  expects 'github_branch', default: 'master'

  provides 'github_deb'

  def run(payload)
    ...
  end
end
{% endhighlight %}

A pipeline checks the expectations for each conduit in order. It doesn't start if it recognizes that there will be missing keys in the payload. The sanity check adds the keys marked as `provides` to the payload for future verifications, and so on.

Let's see this with an example. Giving this pipeline:

{% highlight ruby %}
EnterpriseBuild = Pipeline[
   BuildGitHubDeb,
   BuildGhp
]
{% endhighlight %}

The `BuildGhp` conduit could look like this:

{% highlight ruby %}
class BuildGhp
  expects 'github_deb'
  expects 'gist_deb'
  
  provides 'ghp_path'

  def run(payload)
  end
end
{% endhighlight %}

Before the `EnterpriseBuild` starts, it runs the sanity check. When it arrives to the `BuildGhp`, it detects that BuildGitHubDeb provides `github_deb`. It also expects `gist_deb`, but nobody provides it. In that case, the pipeline sends a notification that there is a missing expectation, and therefore doesn't start the build.

We use RSpec in this project for unit testing. We have custom rspec expectations for these sanity checks. This way, we don't need to run a pipeline in production. We can write unit tests for it. A test to verify that a pipeline is correct looks like this:

{% highlight ruby %}
describe EnterpriseBuild do
  it { should be_valid_provided_with('version') }
end
{% endhighlight %}

That runs the sanity check to verify that the pipeline is correct when the initial payload only contains the key `version.`

Using this simple api, we provide fast fails and improve our feedback loop. We don't need to wait several minutes to realize that something is not going to work, as we expect when we run a command in production.

I'll explain how we make all this faster in my next blog post.

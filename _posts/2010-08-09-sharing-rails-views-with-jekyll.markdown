---
title: Sharing Rails views with Jekyll
author: Andy
layout: post
categories: technology
---

In my last post I discussed how we [share a single layout between Rails apps](http://numbers.brighterplanet.com/2010/07/26/sharing-views-across-rails-3-apps). This has been a lifesaver for us as we manage a half-dozen production apps. But a couple of our sites---our [developer hub](http://brighterplanet.github.com) and this here blog---don't use Rails. They're both [Jekyll](http://github.com/mojombo/jekyll) sites running on [GitHub Pages](http://pages.github.com).

Obviously we can't rely on the Rails Engine features in our [shared layout](http://github.com/brighterplanet/brighter_planet_layout) gem to load the layout into the right places. What are we to do?

### One step at a time ###

Luckily Jekyll already includes the basic building blocks of our solution: layouts and includes. Layouts, described [here](http://wiki.github.com/mojombo/jekyll/usage) are [Liquid](http://github.com/tobi/liquid) templates, and Jekyll ships with a custom [Liquid extension that enables includes](http://wiki.github.com/mojombo/jekyll/liquid-extensions).

All we need to do then is transform our Rails layout into a Jekyll layout and use includes instead of partials. Ready, set, go.

### When in Rome ###

Jekyll is a static site generator. Following this lead, our transformations will be manually executed and staticly stored within your Jekyll site. The easiest way to get started is to set up a task in your Rakefile:

{% highlight ruby %}
require 'net/http'
require 'uri'
require 'erb'
require 'lib/stubs'
namespace :layout do
  task :build do
    File.open File.join(File.dirname(__FILE__), '_layouts', 'default.html'), 'w' do |f|
      f.puts ERB.new(Net::HTTP.get(URI.parse('http://github.com/brighterplanet/brighter_planet_layout/raw/master/app/views/layouts/brighter_planet.html.erb'))).result(Layout.new.get_binding  { |*pages| '{ { content } }' if pages.empty? })
    end
  end
end
{% endhighlight %}

What's going on here? Rake is fetching the raw ERB of your layout from the gem's repository, sending it to ERB for processing, and then storing the result as your Jekyll site's `default` layout.

I should call your attention to a couple of tricky bits here.

### Bindings ###
First, this business about bindings. ERB needs a "binding" to work--that is, a context within which it can access instance methods, variables, etc. Rails takes care of this for you, but since we're invoking ERB here directly, we have to tell it where to bind. Why is this important? Your layout probably uses methods like `stylesheet_link_tag` or `render` to get its job done. If we don't provide those methods in ERB's context, we'll get `NoMethodError` all over the place. The easiest way to fool ERB is with a fake context, which we'll put in `lib/stubs.rb`:

{% highlight ruby %}
class Layout
  def stylesheet_link_tag(*sheets)
    sheets.collect do |sheet|
      "<link rel=\"stylesheet\" type=\"text/css\" href=\"/stylesheets/#{sheet}.css\" />"
    end
  end
  
  def javascript_include_tag(*args); end
  
  def render(options = {})
    "{٪ include #{options[:partial][/[a-z_]*$/]}.html ٪}"
  end
  
  def get_binding
    binding
  end
end
{% endhighlight %}

You can see how we re-interpret these method calls in a way that's meaningful to Jekyll. (Note that since `binding` is a private method we have to publicize it with the `get_binding` wrapper.)

### Yields ###
The second tricky bit is dealing with your standard Rails layout's multiple `yield` calls, the consequence of using `content_for` blocks in your views. We have to anticipate this and set up ERB to act accordingly. Where do we even capture arguments to yield? Turns out the correct place to do this on `get_binding`, our wrapper to the private `binding` method. Now the `yield` we're interested in---the one where we want Jekyll content to go---is the one called without any arguments. So we set the block to output the `content` Liquid tag when it sees `yield` called with an empty argument set. Other `yield` calls---to dump `content_for` material, which could never be prepared by Jekyll anyways---are simply ignored.

### And we're done ###
Your complete layout package will probably include several partials---each with its own fake context class in `stubs.rb`---as well as asset files. To build your layout from its source in the cloud, just

{% highlight console %}
$ rake layout:build
{% endhighlight %}

Check out our developer hub's [`Rakefile`](http://github.com/brighterplanet/brighterplanet.github.com/blob/master/Rakefile) and [`stubs.rb`](http://github.com/brighterplanet/brighterplanet.github.com/blob/master/lib/stubs.rb) for all the details.

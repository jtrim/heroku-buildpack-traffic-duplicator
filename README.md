# Traffic Duplicator

#### A Heroku buildpack for duplicating traffic from a source Heroku application to N destinations.

### Why?

Sometimes a good ol' fashioned load test does the trick. It helps you track down where the bottlenecks are, then you're on your merry way. But what if the performance characteristics of your application are mostly driven by specific, hard-to-repeat user interaction?

You could spend the time contriving lots of different user scenarios to come close to repeating what you're seeing on production, but what those interaction patterns change? Lots of things can affect the way a user interacts with an application - a new feature, removing a feature, or a UI redesign just to name a few - so then you have to rebuild those aforementioned user scenarios. Why not just use the _actual_ traffic from your production app?

That's what this buildpack helps you do.

### Quick Start

This example makes a few assumptions:

- that we have a standard Ruby on Rails application
- that this app is already running on Heroku
- and that we're starting out with this in our `Procfile`: `web: bundle exec rails s -p $PORT`

---

Firstly, this buildpack depends on a couple of libraries: [`em-proxy`](https://github.com/igrigorik/em-proxy) and [`gor`](https://github.com/buger/gor). While the `gor` binary is bundled with this buildpack, you'll need to ensure the `em-proxy` binary is installed and available in the shell environment that runs your application. With a Ruby application, this is as easy as adding `em-proxy` to your `Gemfile`:

`Gemfile`

```ruby
source 'https://rubygems.org'
gem 'em-proxy'
```

Next, you'll need set the buildpack of your application to [`heroku-buildpack-multi`](https://github.com/ddollar/heroku-buildpack-multi) (skip this step if you're already using this buildpack):

```bash
heroku buildpacks:set https://github.com/ddollar/heroku-buildpack-multi.git
```

Now, edit your `.buildpacks` file. In this case, we want to add `heroku-buildpack-traffic-duplicator` **before** the buildpack that your application would normally run on. In this case, since we're running a Rails app, we want to add the Ruby buildpack as the last line of the file:

`.buildpacks`

```
https://github.com/jtrim/heroku-buildpack-traffic-duplicator.git
https://github.com/heroku/heroku-buildpack-ruby.git
```

(see [the Heroku buildpack guide](https://devcenter.heroku.com/articles/buildpacks) for more information on how buildpacks work)

Next, we'll need to set some options for [Gor](https://github.com/buger/gor) to do its thing. This buildpack looks for a file called `gor.options` within the `config` directory of your application. This is how we tell `gor` where to send the traffic it's duplicating:

`config/gor.options`

```bash
--output-http 'https://my-downstream-app.herokuapp.com'
--output-http-header 'Host: my-downstream-app.herokuapp.com'
```

Gor has [lots of other options](https://github.com/buger/gor#command-line-reference) to control how it treats traffic it is asked to forward, including regex-based filtering against http headers and URL strings.

Lastly, you'll need to add the wrapper script to your `Procfile`:

`Procfile`

```yaml
web: start-traffic-duplicator bundle exec rails s -p $PORT
```

Now, deploy your application. Requests will be duplicated from the Heroku dyno(s) running your production application to the downstream application(s) you've specified.

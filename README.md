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

### How does it work?

During a deploy, Heroku runs `bin/detect`, `bin/compile`, and `bin/release` for every buildpack present. The `bin/compile` script in this library copies `bin/gor` and `bin/start-traffic-duplicator` to your application's `bin` directory.

Now, given the example application above, Heroku would normally run `bundle exec rails s -p $PORT` to start the Rails application, where `$PORT` is an environment variable set by Heroku that corresponds to the network port the application will serve traffic on. When we change the Procfile entry to `start-traffic-duplicator bundle exec rails s -p $PORT`, Heroku instead starts the wrapper script, supplying the `bundle exec ...` bit as arguments.

`start-traffic-duplicator`'s job is to start three processes (forgive the slight pseudocode):

- your application, via `bundle exec rails s -p $PORT`, which is reconfigured to listen on `$PORT+1`
- `gor`, via `gor --input-http :$PORT+2` and any additional options you've specified in `config/gor.options`
- `em-proxy`, via `bundle exec em-proxy -l $PORT -r 127.0.0.1:$PORT+1 -d 127.0.0.1:$PORT+2`

In other words, `em-proxy` takes your application's place by binding to `$PORT`. Upon receiving a request, `em-proxy` relays that request to your application on `$PORT+1` and duplicates the traffic to `gor` on `$PORT+2`, discarding the response from `gor`.

### Caveats

- This trio of processes currently depends on the loopback interface local to the Heroku dyno. This implies a couple of things:
  - There is going to be a very very very slight performance hit associated with using the loopback interface for gor and your application. For 99% of you who'd like to use this library, **that hit will be negligible**. However, if you're squeezing every last bit of performance out of Heroku's dynos, and your application's performance hangs in a delicate balance, you may want to take that into consideration
  - Heroku exposes `$PORT`, and this buildpack uses `$PORT+1` and `$PORT+2` with reckless abandon. In other words, if you've built yourself a complex ecosystem of processes that also listen on the loopback interface, you may end up with conflicts
- While `em-proxy` has the ability to relay to unix sockets, the straightforward option is to use the loopback interface. If you want this functionality, I'm happy to accept pull requests!

### Contributing

- Fork it
- Create your feature branch (git checkout -b my-new-feature)
- Commit your changes (git commit -am 'Added some feature')
- Push to the branch (git push origin my-new-feature)
- Create new Pull Request

### License

`em-proxy`: The MIT License - Copyright (c) 2010 Ilya Grigorik   
`gor`: [Apache License Version 2.0](https://github.com/buger/gor/blob/master/LICENSE.txt)


Everything else: The MIT License - Copyright (c) 2015 Jesse Trimble

# Alaska

At Mavenlink we utilize the rails3 asset pipeline (known as sprockets) to compile our coffeescript into javascript. We began charting the time spent waiting for our application to load (and compile coffeescript) over the past 6 months and determined that if we let the trend continue, we would reach "peak coffeescript production"... meaning our developers would literally be spending more time waiting for their coffeescript to compile per page load, than spent looking at the results!

# Peak Coffeescript

To combat this trend, we developed `alaska` ... the persistent execjs runtime used for coffeescript compilation. The mechanism used is fundamentally different than the default execjs runtime. The differences are outlined below.

## ExecJS::ExternalRuntime

In the default execjs runtime, coffeescript files are converted to javascript files roughly as follows:

1. The sprockets system determines which file needs to be updated
2. An ExecJS::ExternalRuntime::Context is created with the javascript source of the coffeescript compiler as an instance variable.
3. For each file sprockets determines needs to be updated, a new temporary file is created that contains the contexts coffeescript compilation module AND the original coffeescript source code
4. The node interperater is then executed with this tempory file as the argument, and the standard output of that process is delivered back to the sprockets system
5. Sprockets continues with this process for each remaining file, invoking the /usr/bin/node process for each file to be compiled.

To break this down into laymens terms, it means that for every gallon of oil (compiled coffeescript) we want to pull out of the ground, we have to send a truck, with its out drill setup out to the location. It should be immediatly appearant that this approach will begin to be slower especially as more and more coffeescript is being produced.

## ExecJS::Alaska

In contrast to the default execjs runtime, the alaska runtime constructs a persistent pipeline to the nodejs interepreter, greatly reducing roundtrip time of coffeescript compliation.

1. The sprockets system determines which file needs to be updated (it should be noted that this gem does not alter anything with the sprockets module)
2. An ExecJS::Alaska::Context is created with the javascript source of the coffeescript compiler, a seperate nodejs daemon is spawned in the background, and a http server is started on a local UNIX socket for communication.
3. The initial coffeescript compilation module is then loaded _once_ in the nodejs runtime
4. For each file to be compiled, a http request is made with the request body set to the coffeescript, the response body is the compiled javascript, which is delivered back to sprockets

With this caching of the coffeescript compilation module, and the persistent nodejs compliation server process, we can reduce the roundtrip time for each coffeescript compilation down to several milliseconds (on average in mavenlinks primary rails application the roundtrip time is 16ms)


# DRILL BABY DRILL

In a rails initializer file (e.g. `config/initializers/execjs.rb`) declare the `ExecJS.runtime` to be an instance of `Alaska`

    require 'alaska'

    if Rails.env == "development" || Rails.env == "test" || ENV["RAILS_GROUPS"] == "assets"
      # use alaska.js pipelining only when precompiling assets
      ExecJS.runtime = Alaska.new(:debug => true)
    end

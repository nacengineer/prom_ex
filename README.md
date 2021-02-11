<!--START-->
<p align="center">
  <img align="center" width="40%" src="guides/images/logo.svg" alt="PromEx Logo">
  <img align="center" width="40%" src="guides/images/logo_text.png" alt="PromEx Logo">
</p>

<p align="center">
  Prometheus metrics and Grafana dashboards for all of your favorite Elixir libraries
</p>

<p align="center">
  <a href="https://hex.pm/packages/prom_ex">
    <img alt="Hex.pm" src="https://img.shields.io/hexpm/v/prom_ex?style=for-the-badge">
  </a>

  <a href="https://github.com/akoutmos/prom_ex/actions">
    <img alt="GitHub Workflow Status (branch)" src="https://img.shields.io/github/workflow/status/akoutmos/prom_ex/PromEx%20CI/master?label=Build%20Status&style=for-the-badge">
  </a>

  <a href="https://coveralls.io/github/akoutmos/prom_ex?branch=master">
    <img alt="Coveralls github branch" src="https://img.shields.io/coveralls/github/akoutmos/prom_ex/master?style=for-the-badge">
  </a>
</p>

<br>
<!--END-->

# Contents

- [Installation](#installation)
- [Design Philosophy](#design-philosophy)
- [Available Plugins](#available-plugins)
- [Grafana Dashboards](#grafana-dashboards)
- [Setting Up Metrics](#setting-up-metrics)
- [Performance Concerns](#performance-concerns)
- [Attribution](#attribution)

## Installation

This library is still under active development with changing API contracts and forked dependencies...use at your own
risk for now :).

[Available in Hex](https://hex.pm/packages/prom_ex), the package can be installed by adding `prom_ex`
to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:prom_ex, "~> 0.1.14-beta"}
  ]
end
```

Documentation can be found at [https://hexdocs.pm/prom_ex](https://hexdocs.pm/prom_ex).

### Design Philosophy

With the widespread adoption of the Telemetry library and the other libraries in the [BEAM Telemetry GitHub
Org](https://github.com/beam-telemetry), we have reached a point where we have a consistent means of surfacing
application and library metrics. This allows us to have a great level of insight into our applications and dependencies
given that they all leverage the same fundamental tooling. The goal of this project is to provide a "Plug-in" style
library where you can easily add new plug-ins to surface metrics so that Prometheus can scrape them. Ideally, this
project acts as the "Metrics" pillar in your application (in reference to [The Three Pillars of
Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html)).

To this end, while PromEx does provide a certain level of configurability (like the polling rate, starting behaviour for
manual metrics and all the options that the plugins receive), the goal is not to make an infinitely configurable tool.
For example, you are not able to edit the names/descriptions of Prometheus metrics via plugin options or even the tags
that are attached to the data points.

Instead, if there things that you don't agree with or that are incompatible with your usage of a certain 1st party
plugin and want to edit how the PromEx plugins react to Telemetry events, it is recommended that you fork the plugin in
question and edit it to your specific use case. If you think that the community can benefit for your changes, do not
hesitate to make a PR and I'll be sure to review it. This is not to say that event configurability will never come to
PromEx, but I want to make sure that the public facing API is clean and straightforward and not bogged down with
configuration. That and the Grafana dashboards would then have to become templatized to accommodate all this
configurability.

PromEx provides a few utilities to you in order to accomplish this goal:

- The `PromEx.Plug` module that can be used in your Phoenix or Plug application to expose the collected metrics
- A Mix task to upload the provided complimentary Grafana dashboards
- A Mix task to create a PromEx metrics capture module
- A behaviour that defines the contract for PromEx plug-ins
- A behaviour that defines the functionality of a PromEx metrics capture module
- Grafana dashboards tailored to each specific Plugin so that metrics work out of the box with dashboards

### Available Plugins

| Plugin                           | Status      | Description                                            |
| -------------------------------- | ----------- | ------------------------------------------------------ |
| `PromEx.Plugins.Application`     | Beta        | Collect metrics on your application dependencies       |
| `PromEx.Plugins.Beam`            | Beta        | Collect metrics regarding the BEAM virtual machine     |
| `PromEx.Plugins.Phoenix`         | Beta        | Collect request metrics emitted by Phoenix             |
| `PromEx.Plugins.Ecto`            | Beta        | Collect query metrics emitted by Ecto                  |
| `PromEx.Plugins.Oban`            | Beta        | Collect queue processing metrics emitted by Oban       |
| `PromEx.Plugins.PhoenixLiveView` | Beta        | Collect metrics emitted by Phoenix LiveView            |
| `PromEx.Plugins.Broadway`        | Coming soon | Collect message processing metrics emitted by Broadway |
| `PromEx.Plugins.Absinthe`        | Coming soon | Collect GraphQL metrics emitted by Absinthe            |
| `PromEx.Plugins.Finch`           | Coming soon | Collect HTTP request metrics emitted by Finch          |
| `PromEx.Plugins.Redix`           | Coming soon | Collect Redis request metrics emitted by Redix         |
| More to come...                  |             |                                                        |

### Grafana Dashboards

<img align="center" width="100%" src="guides/images/dashboards_preview.png" alt="PromEx Dashboards">

PromEx comes with a custom tailored Grafana Dashboard per Plugin. [Click here](https://hexdocs.pm/prom_ex/grafana-dashboards.html)
to check out sample screenshots of each Plugin specific Grafana Dashbaord.

### Setting Up Metrics

The goal of PromEx is to have metrics set up be as simple and streamlined as possible. In that spirit, all
that you need to do to start leveraging PromEx along with the built-in plugins is to run the following mix
task:

```
$ mix prom_ex.gen.config --datasource YOUR_PROMETHEUS_DATASOURCE_ID
```

Then add the generated module to your `application.ex` file supervision tree:

```elixir
defmodule MyCoolApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      ...

      MyCoolApp.PromEx
    ]

    opts = [strategy: :one_for_one, name: MyCoolApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

With that in place, all that you need to do is then add the PromEx plug somewhere in your
`endpoint.ex` file (I would suggest putting it before your `plug Plug.Telemetry` call so that
you do not pollute your logs with calls to `/metrics`):

```elixir
defmodule MyCoolAppWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_cool_app

  ...

  plug PromEx.Plug, prom_ex_module: MyCoolApp.PromEx
  # Or plug PromEx.plug, path: "/some/other/metrics/path", prom_ex_module: MyCoolApp.PromEx

  ...

  plug Plug.RequestId
  plug Plug.Telemetry, event_prefix: [:phoenix, :endpoint]

  ...

  plug MyCoolAppWeb.Router
end
```

With that in place, all you need to do is start your server and you should be able to hit your
metrics endpoint and see your application metrics:

```terminal
$ curl localhost:4000/metrics
# HELP my_cool_app_application_dependency_info Information regarding the application's dependencies.
# TYPE my_cool_app_application_dependency_info gauge
my_cool_app_application_dependency_info{modules="69",name="hex",version="0.20.5"} 1
my_cool_app_application_dependency_info{modules="1",name="connection",version="1.0.4"} 1
my_cool_app_application_dependency_info{modules="4",name="telemetry_poller",version="0.5.1"} 1
...
...
```

Be sure to check out the module docs for each plugin that you choose to use to ensure that you are familiar
with all of the options that they provide.

### Performance Concerns

You may think to yourself that with all these metrics being collected and scraped, that the performance of your
application may be negatively impacted. Luckily PromEx is built upon the solid foundation established by the `Telemetry`,
`TelemetryMetrics`, and the `TelemetryMetricsPrometheus` projects. These libraries were designed to be as lightweight
and performant as possible. From some basic stress tests that I have run, I have been unable to observe any meaningful
performance reduction (thank you OTP and particularly ETS ;)). Below are the results from a recent stress test using
ApacheBench:

#### With PromEx metrics collection

```terminal
$ ./benchmarks/ab-graph.sh -u http://localhost:4000 -n 1000 -c 50 -k
Server Software:        Cowboy
Server Hostname:        localhost
Server Port:            4000

Document Path:          /
Document Length:        3389 bytes

Concurrency Level:      50
Time taken for tests:   4.144 seconds
Complete requests:      1000
Failed requests:        0
Keep-Alive requests:    1000
Total transferred:      4060000 bytes
HTML transferred:       3389000 bytes
Requests per second:    241.32 [#/sec] (mean)
Time per request:       207.191 [ms] (mean)
Time per request:       4.144 [ms] (mean, across all concurrent requests)
Transfer rate:          956.81 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.2      0       1
Processing:    39  202  24.3    203     264
Waiting:       38  202  24.3    203     264
Total:         39  202  24.2    203     264

Percentage of the requests served within a certain time (ms)
  50%    203
  66%    210
  75%    215
  80%    218
  90%    227
  95%    237
  98%    246
  99%    255
 100%    264 (longest request)
```

#### Without PromEx metrics collection

```terminal
$ ./benchmarks/ab-graph.sh -u http://localhost:4000 -n 1000 -c 50 -k
Server Software:        Cowboy
Server Hostname:        localhost
Server Port:            4000

Document Path:          /
Document Length:        3389 bytes

Concurrency Level:      50
Time taken for tests:   4.156 seconds
Complete requests:      1000
Failed requests:        0
Keep-Alive requests:    1000
Total transferred:      4060000 bytes
HTML transferred:       3389000 bytes
Requests per second:    240.59 [#/sec] (mean)
Time per request:       207.822 [ms] (mean)
Time per request:       4.156 [ms] (mean, across all concurrent requests)
Transfer rate:          953.90 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       1
Processing:    38  202  23.1    205     267
Waiting:       37  202  23.1    205     267
Total:         38  202  23.0    205     267

Percentage of the requests served within a certain time (ms)
  50%    205
  66%    211
  75%    215
  80%    219
  90%    226
  95%    232
  98%    238
  99%    246
 100%    267 (longest request)
```

#### Plotting the stress test results

In the spirit of visualizing performance characteristics, the percentile data from the ApacheBench stress tests has been
overlaid and plotted using Gnuplot (thanks to https://github.com/juanluisbaptiste/apachebench-graphs for making
gnuplotting a lot more streamlined :)). As we can see, the charts track each other more or less 1:1 except for the
slowest 5% of requests where we see a slight performance hit.

<img align="center" width="100%" src="guides/images/apache_bench_stress_test.png" alt="PromEx Stress Test">

### Attribution

It wouldn't be right to not include somewhere in this project a "thanks" to the various projects that
helped make this possible:

- The various projects available in [BEAM Telemetry](https://github.com/beam-telemetry)
- All of the Prometheus libraries that Ilya Khaprov ([@deadtrickster](https://github.com/deadtrickster)) maintains
- The logo for the project is an edited version of an SVG image from the [unDraw project](https://undraw.co/)

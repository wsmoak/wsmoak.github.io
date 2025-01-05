---
layout: post
title:  "OpenTelemetry Logging for Ruby on Rails"
date:   2025-01-04 13:00:00 -0500
tags:   otel rails
---

OpenTelemetry provides powerful observability capabilities for Rails applications. Let's explore how to configure logging.

### Basic Setup

Create a Rails app with `rails new -T rails-otel-demo`

The configuration lives in [config/initializers/open_telemetry.rb][open-telemetry-rb] and demonstrates several key concepts:

```ruby
OpenTelemetry::SDK.configure do |c|
  # Configuration goes here
end
```

(The examples I found use `opentelemetry.rb` for this filename but the official project name is OpenTelemetry so by convention it should have the underscore.)

### Resource Attributes

The OTel Ruby SDK supports many of the standard OTel environment variables.  You can find a list of them in the [spec compliance matrix](https://github.com/open-telemetry/opentelemetry-specification/blob/main/spec-compliance-matrix.md#environment-variables).

For example if you are using the `OTEL_RESOURCE_ATTRIBUTES` environment variable, it should have the following format for the value:
```sh
OTEL_RESOURCE_ATTRIBUTES="k8s.namespace.name=the-namespace,k8s.pod.uid=a2b3c4d5-e6f7"
```

### Configuration Precedence

By default, values set in the code will override values set in environment variables.  The code in this example is written to prefer the OTel standard environment variables and fall back to a value set in the code if necessary.

1. Environment variables (like `OTEL_SERVICE_NAME` and `OTEL_SERVICE_ATTRIBUTES`)
2. Programmatic configuration
3. Default values

For example:
```ruby
c.service_name = ENV.fetch('OTEL_SERVICE_NAME', 'from_config_initializer')
```

Then if you run the app with
```bash
$ OTEL_LOGS_EXPORTER=otlp OTEL_SERVICE_NAME=from_envar bundle exec rails server -p 3001
```
you will see that the service_name label in Loki is set to `from_envar` rather than `from_config_initializer`.

Similarly if you know that the Kubernetes Namespace Name is set as `K8S_NAMESPACE` in your deployed environment, you can write

```ruby
  SCR = OpenTelemetry::SemanticConventions::Resource

  SCR::K8S_NAMESPACE_NAME => resource_attrs[SCR::K8S_NAMESPACE_NAME] || ENV.fetch('K8S_NAMESPACE', 'unknown_namespace')
```

(The code for parsing the resource attributes from the environment variable is in [config/initializers/open_telemetry.rb][open-telemetry-rb].)

This can help if other parts of your CI/CD process are injecting enviroment variables other than the standard ones, and/or you are using a service such as Seekrit or Doppler to control them.

### Semantic Conventions

The code uses OpenTelemetry's semantic conventions for standardized attribute naming:

```ruby
c.resource = OpenTelemetry::SDK::Resources::Resource.create(
  OpenTelemetry::SemanticConventions::Resource::DEPLOYMENT_ENVIRONMENT => Rails.env.to_s
)
```

When running the app locally, this sets `deployment.environment` to `development`, which shows up as the value for the `deployment_environment` label in Loki.

### Types

Note that with the Ruby SDK, all the attribute keys must be strings, and all the values must be string or number (or arrays of strings or numbers).

Simply using `Rails.env` as the value of an attribute will NOT work!  It has a string representation that looks like what we need, but it is NOT a String.

```ruby
$ bundle exec rails console

rails-otel-demo(dev)> Rails.env
=> "development"

rails-otel-demo(dev)> Rails.env.class
=> ActiveSupport::EnvironmentInquirer
```

### Grafana docker-otel-lgtm

To see the logs show up in Loki using Grafana locally, try out Grafana's docker-otel-lgtm project:

```bash
git clone https://github.com/grafana/docker-otel-lgtm.git
cd docker-otel-lgtm
./run-lgtm.sh
```

Then visit http://localhost:3000

Note that the default port for the Rails app is also 3000, so start the Rails app with -p 3001 to avoid a conflict.

### Inflections

Don't forget to un-comment the code in `config/initializers/inflections.rb` and add one for OTel.  Otherwise, if you have any names with `_otel` it will get capitalized as Otel which is incorrect.

```ruby
  ActiveSupport::Inflector.inflections(:en) do |inflect|
    inflect.acronym "RESTful"
    inflect.acronym "OTel"
  end
```

### References

- https://github.com/wsmoak/rails-otel-demo/tree/1.0.0
- OpenTelemetry Ruby in GitHub https://github.com/open-telemetry/opentelemetry-ruby
- [semantic_conventions/resource.rb](https://github.com/open-telemetry/opentelemetry-ruby/blob/main/semantic_conventions/lib/opentelemetry/semantic_conventions/resource.rb) for standard attribute names
- The official OpenTelemetry documentation for Ruby https://opentelemetry.io/docs/languages/ruby/
- https://opentelemetry.io/docs/languages/ruby/getting-started/
- [Steven Harman's post in #otel-ruby in the CNCF Slack from January 2003 showing how to configure resource attributes](https://cloud-native.slack.com/archives/C01NWKKMKMY/p1674566998568639?thread_ts=1674560943.812979&cid=C01NWKKMKMY)
- [Kayla Reopelle's announcement about logging support in the Ruby SDK in #otel-ruby in the CNCF Slack from December 2024](https://cloud-native.slack.com/archives/C01NWKKMKMY/p1733516156143249)
- If you can't see the Slack messages, [get an invitation to the CNCF Slack](https://slack.cncf.io)
- https://opentelemetry.io/docs/languages/sdk-configuration/general/

### AI

I used Anthropic's Claude 3.5 Sonnet for help with some of the code and to draft this blog post.  It really is awesome.  Keep notes in the README.md file of a project while you are learning something new, and then ask it (using the VS Code Github Copilot plugin) "@workspace look at the open_telemetry.rb, logger.rb, and README.md files and write a blog post explaining what I've learned".

[cc-by-nc]: https://creativecommons.org/licenses/by-nc/4.0/deed.en
[site-url]: https://wsmoak.net
[open-telemetry-rb]: https://github.com/wsmoak/rails-otel-demo/blob/1.0.0/config/initializers/open_telemetry.rb

Copyright 2025 Wendy Smoak - This post first appeared on [wsmoak.net][site-url] and is [CC BY-NC][cc-by-nc] licensed.

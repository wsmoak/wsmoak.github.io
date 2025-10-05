---
layout: post
title:  "Sending Structured Logs to OpenTelemetry with dry-logger"
date:   2025-10-05 14:30:00
tags: ruby rails opentelemetry logging dry-logger
---

If you have an existing Rails application using [dry-logger](https://dry-rb.org/gems/dry-logger/) and you're migrating to OpenTelemetry for observability, you'll want a smooth transition path. Rather than switching all your logging overnight, dry-logger's multiple backend support allows you to send logs to both your existing destination and OpenTelemetry simultaneously. Once you're confident everything works, you can switch to OpenTelemetry exclusively.

## Setting Up

First, add the OpenTelemetry Ruby SDK to your project and configure it.  See my earlier post on [OpenTelemetry Logging for Ruby on Rails](https://wsmoak.net/2025/01/04/rails-otel-logging.html) for more information.

## Creating the OpenTelemetry Backend

Here is an example of a custom backend that uses the OTel Ruby SDK for logging.

```ruby
module DryLogger
  class OpenTelemetryBackend

    def info(message = nil, **payload)
      log(:info, message, **payload)
    end

    # ... repeat for debug, warn, error, fatal, and unknown.
    # Or this duplication can be removed by using `method_missing`.

    private

    def otel_logger
      @otel_logger ||= OpenTelemetry.logger_provider.logger(
        name: "your_app_name", # scope.name
        version: "your_app_version" # scope.version
      )
    end

    def log(severity, message, **payload)
      severity_text = severity.to_s.upcase
      json_payload = payload.to_json

      payload.deep_stringify_keys!
      payload = flatten_hash(payload)
      payload.transform_values!(&:to_s)

      if message.present?
        otel_logger.on_emit(
          severity_text: severity_text,
          body: message,
          attributes: payload
        )
      else
        otel_logger.on_emit(
          severity_text: severity_text,
          body: json_payload,
          attributes: payload
        )
      end
    end

    def flatten_hash(hash, separator = ".")
      hash.each_with_object({}) do |(key, value), result|
        if value.is_a?(Hash)
          flatten_hash(value, separator).each do |nested_key, nested_value|
            result["#{key}#{separator}#{nested_key}"] = nested_value
          end
        else
          result[key] = value
        end
      end
    end
  end
end
```

## How It Works

Since the message is optional with dry-logger, the backend handles both cases.  If the message is there, it becomes the body, and the (flattened and string-ified) payload goes into the attributes.  If the message is not there, then it adds the whole json-formatted payload as the body in addition to the attributes.

For the payload, depending on your situation, you may want to be explicit about which keys are added as attributes for OTel logging instead of adding everything.

### Logging with a message

```ruby
logger.info("Customer retrieved", customer_id: 123, metadata: { source: "api" })
```

In this case:
- The message `"Customer retrieved"` becomes the log body
- The payload is flattened and added as attributes: `customer_id: "123"`, `metadata.source: "api"`

### Logging without a message (payload only)

```ruby
logger.info(customer_id: 123, metadata: { source: "api" })
```

Here:
- The JSON-serialized payload becomes the log body: `"{\"customer_id\":123,\"metadata\":{\"source\":\"api\"}}"`
- The flattened payload is still included as attributes for easy searching

This dual approach ensures you always have readable log messages and searchable structured attributes in your OpenTelemetry backend.

## Key Features

**Nested Hash Flattening**: Complex nested structures are automatically flattened with dot notation. For example:

```ruby
{ user: { id: 123, name: "John" }, metadata: { source: "api", version: 2 } }
```

Becomes:
- `user.id: "123"`
- `user.name: "John"`
- `metadata.source: "api"`
- `metadata.version: "2"`

**Type Conversion**: All attribute keys and values are converted to strings for OpenTelemetry compatibility.

**Flexible Logging**: Supports both traditional message-based logging and modern structured logging patterns.

## Usage

There are two ways to use the OpenTelemetry backend with dry-logger:

### Option 1: OpenTelemetry Only

Create a helper module in `app/lib/application_logger.rb`:

```ruby
module ApplicationLogger
  def self.build(id)
    Dry.Logger(id) do |dispatcher|
      dispatcher.add_backend(DryLogger::OpenTelemetryBackend.new)
    end
  end
end
```

Then use it in your application:

```ruby
logger = ApplicationLogger.build(:your_app)
logger.info("Processing order", order_id: 12345, amount: 99.99)
```

This approach sends logs exclusively to OpenTelemetry without any STDOUT output, which is ideal for production environments where you want centralized logging.

### Option 2: OpenTelemetry + Default Stream

Add the OpenTelemetry backend to an existing dry-logger instance:

```ruby
logger = Dry::Logger(:your_app).add_backend(DryLogger::OpenTelemetryBackend.new)
logger.info("Processing order", order_id: 12345, amount: 99.99)
```

This sends logs to both STDOUT and OpenTelemetry, helpful during development when you want to see logs locally while also populating your telemetry backend.

## Real-World Example

Here's how you might use it in a Rails controller:

```ruby
class CustomersController < ApplicationController
  def index
    logger = ApplicationLogger.build(:customers)

    logger.info("Fetching customer list",
                user_id: current_user.id,
                filters: params[:filters],
                metadata: { source: request.remote_ip })

    @customers = Customer.all

    logger.info("Customer list retrieved",
                count: @customers.size,
                duration_ms: Time.current - start_time)
  end
end
```

In your OpenTelemetry backend, you'll see structured logs with all the context you need for debugging and monitoring, with searchable attributes like `user_id`, `filters`, `metadata.source`, `count`, and `duration_ms`.

## Testing Your Backend

Here's a simple RSpec example to verify your backend works correctly:

```ruby
RSpec.describe DryLogger::OpenTelemetryBackend do
  subject(:backend) { described_class.new }

  it "sends logs with flattened attributes to OpenTelemetry" do
    message = "Test with nested payload"
    payload = { user: { id: 123, name: "John" } }
    otel_logger = instance_double(OpenTelemetry::SDK::Logs::Logger)

    allow(backend).to receive(:otel_logger).and_return(otel_logger)

    expect(otel_logger).to receive(:on_emit).with(
      severity_text: "INFO",
      body: message,
      attributes: {
        "user.id" => "123",
        "user.name" => "John"
      }
    )

    backend.info(message, **payload)
  end
end
```

## Conclusion

By implementing a custom OpenTelemetry backend for dry-logger, you get the best of both worlds: clean, maintainable logging code in your application and rich, structured telemetry data in your observability platform. The flattened attributes make it easy to search and filter logs any OpenTelemetry-compatible backend.

For a more complete example, check out the [`with-dry-logger` branch of the `rails-otel-demo` repository](https://github.com/wsmoak/rails-otel-demo/tree/with-dry-logger-backend) on GitHub.

### AI

I used Anthropic's Claude for help with some of the code and to draft this blog post.

[cc-by-nc]: https://creativecommons.org/licenses/by-nc/4.0/deed.en
[site-url]: https://wsmoak.net

Copyright 2025 Wendy Smoak - This post first appeared on [wsmoak.net][site-url] and is [CC BY-NC][cc-by-nc] licensed.
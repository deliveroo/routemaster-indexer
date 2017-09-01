## Materialist

> _adjective_ `philosophy`: relating to the theory that nothing exists except matter and its movements and modifications.

A "materializer" is a ruby class that is responsible for receiving an event and
materializing the remote resource (described by the event) in database.

This library is a set of utilities that provide both the wiring and the DSL to
painlessly do so.

### Configuration

First you need an "event handler":

```ruby
handler = Materialist::EventHandler.new({ ...options })
```

Where options could be:

- `topics` (only when using in `.subscribe`): An array of topics to be used.
If not provided nothing would be materialized.
- `queue`: name of the queue to be used by sidekiq worker

Then there are two ways to configure materialist in routemaster:

1. **If you DON'T need resources to be cached in redis:** use `handler` as siphon:

```ruby
handler = Materialist::EventHandler.new
siphon_events = {
  zones:               handler,
  rider_domain_riders: handler
}

app = Routemaster::Drain::Caching.new(siphon_events: siphon_events)
# ...

map '/events' do
  run app
end
```

2. **You DO need resources cached in redis:** In this case you need to use `handler` to subscribe to the caching pipeline:

```ruby
TOPICS = %w(
  zones
  rider_domain_riders
)

handler = Materialist::EventHandler.new({ topics: TOPICS })
app = Routemaster::Drain::Caching.new # or ::Basic.new
app.subscribe(handler, prefix: true)
# ...

map '/events' do
  run app
end
```

### DSL

Next you would need to define a materializer for each of the topic. The name of
the materializer class should match the topic name (in singular)

These materializers would live in a first-class directory (`/materializers`) in your rails app.

```ruby
require 'materialist/materializer'

class ZoneMaterializer
  include Materialist::Materializer

  use_model :zone

  materialize :id, as: :orderweb_id
  materialize :code
  materialize :name

  link :city do
    materialize :tz_name, as: :timezone

    link :country do
      materialize :name, as: :country_name
      materialize :iso_alpha2_code, as: :country_iso_alpha2_code
    end
  end
end
```

Here is what each part of the DSL mean:

#### `use_model <model_name>`
describes the name of the active record model to be used.

#### `materialize <key>, as: <column> (default: key)`
describes mapping a resource key to database column.

#### `link <key>`
describes materializing from a relation of the resource. This can be nested to any depth as shown above.

When inside the block of a `link` any other part of DSL can be used and will be evaluated in the context of the relation resource.

#### `after_upsert <method>`
describes the name of the instance method to be invoked after a record was materialized.

```ruby
class ZoneMaterializer
  include Materialist::Materializer

  after_upsert :my_method

  def my_method(record)
  end
end
```
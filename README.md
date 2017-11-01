# Karafka Sidekiq Backend

[![Build Status](https://travis-ci.org/karafka/karafka-sidekiq-backend.png)](https://travis-ci.org/karafka/karafka-sidekiq-backend)
[![Join the chat at https://gitter.im/karafka/karafka](https://badges.gitter.im/karafka/karafka.svg)](https://gitter.im/karafka/karafka?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[Karafka Sidekiq Backend](https://github.com/karafka/karafka-sidekiq-backend) provides support for consuming (processing) received Kafka messages inside of Sidekiq workers.

## Installations

Add this to your gemfile:

```ruby
gem 'karafka-sidekiq-backend'
```

and create a file called ```application_worker.rb``` inside of your ```app/workers``` directory, that looks like that:

```ruby
class ApplicationWorker < Karafka::BaseWorker
end
```

and you are ready to go. Karafka Sidekiq Backend integrates with Karafka automatically

**Note**: You can name your application worker base class with any name you want. The only thing that is required is a direct inheritance from the ```Karafka::BaseWorker``` class.

## Usage

If you want to process messages with Sidekiq backend, you need to tell this to Karafka.

To do so, you can either configure that in a configuration block:

```ruby
class App < Karafka::App
  setup do |config|
    config.backend = :sidekiq
    # Other config options...
  end
end
```

or on a per topic level:

```ruby
App.routes.draw do
  consumer_group :videos_consumer do
    topic :binary_video_details do
      controller Videos::DetailsController
      worker Workers::DetailsWorker
      interchanger Interchangers::Binary
    end
  end
end
```

You don't need to do anything beyond that. Karafka will know, that you want to run your controllers ```#perform``` method in a background job.

## Configuration

There are two options you can set inside of the ```topic``` block:

| Option       | Value type | Description                                                                                                       |
|--------------|------------|-------------------------------------------------------------------------------------------------------------------|
| worker       | Class      | Name of a worker class that we want to use to schedule perform code                                               |
| interchanger | Class      | Name of a parser class that we want to use to parse incoming data                                                 |


### Workers

Karafka by default will build a worker that will correspond to each of your controllers (so you will have a pair - controller and a worker). All of them will inherit from ```ApplicationWorker``` and will share all its settings.

To run Sidekiq you should have sidekiq.yml file in *config* folder. The example of ```sidekiq.yml``` file will be generated to config/sidekiq.yml.example once you run ```bundle exec karafka install```.

However, if you want to use a raw Sidekiq worker (without any Karafka additional magic), or you want to use SidekiqPro (or any other queuing engine that has the same API as Sidekiq), you can assign your own custom worker:

```ruby
topic :incoming_messages do
  controller MessagesController
  worker MyCustomWorker
end
```

Note that even then, you need to specify a controller that will schedule a background task.

Custom workers need to provide a ```#perform_async``` method. It needs to accept two arguments:

 - ```topic_id``` - first argument is a current topic id from which a given message comes
 - ```params``` - all the params that came from Kafka + additional metadata. This data format might be changed if you use custom interchangers. Otherwise it will be an instance of Karafka::Params::Params.

Keep in mind, that params might be in two states: parsed or unparsed when passed to #perform_async. This means, that if you use custom interchangers and/or custom workers, you might want to look into Karafka's sources to see exactly how it works.

### Interchangers

Custom interchangers target issues with non-standard (binary, etc.) data that we want to store when we do ```#perform_async```. This data might be corrupted when fetched in a worker (see [this](https://github.com/karafka/karafka/issues/30) issue). With custom interchangers, you can encode/compress data before it is being passed to scheduling and decode/decompress it when it gets into the worker.

**Warning**: if you decide to use slow interchangers, they might significantly slow down Karafka.

```ruby
class Base64Interchanger
  class << self
    def load(params)
      Base64.encode64(Marshal.dump(params))
    end

    def parse(params)
      Marshal.load(Base64.decode64(params))
    end
  end
end

topic :binary_video_details do
  controller Videos::DetailsController
  interchanger Base64Interchanger
end
```

## References

* [Karafka framework](https://github.com/karafka/karafka)
* [Karafka Sidekiq Backend Travis CI](https://travis-ci.org/karafka/karafka-sidekiq-backend)
* [Karafka Sidekiq Backend Coditsu](https://app.coditsu.io/karafka/repositories/karafka-sidekiq-backend)

## Note on Patches/Pull Requests

Fork the project.
Make your feature addition or bug fix.
Add tests for it. This is important so we don't break it in a future versions unintentionally.
Commit, do not mess with Rakefile, version, or history. (if you want to have your own version, that is fine but bump version in a commit by itself I can ignore when I pull). Send me a pull request. Bonus points for topic branches.

[![coditsu](https://coditsu.io/assets/quality_bar.svg)](https://app.coditsu.io/karafka/repositories/karafka-sidekiq-backend)

Each pull request must pass our quality requirements. To check if everything is as it should be, we use [Coditsu](https://coditsu.io) that combinse multiple linters and code analyzers for both code and documentation.

Unfortunately, it does not yet support independent forks, however you should be fine by looking at what we require.

Please run:

```bash
bundle exec rake
```

to check if everything is in order. After that you can submit a pull request.
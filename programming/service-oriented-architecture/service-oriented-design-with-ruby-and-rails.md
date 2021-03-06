## Implementing and Consuming Your First Service

Service: A system that responds to HTTP requests. Ex: Twitter API, Amazon S3. *Using a broader scope of definition, a service can refer to a sytem that provides functionality through a shared interface.* This would include RDBMSes, Memcached servers, message queues (RabbitMQ), other types of data stores.

Simple service: User management system. This could be used across multiple applications within an organization as a single sign-on point.

#### Tools

- *Sinatra.* Sinatra is perfectly suited for creating small web services.
- *ActiveRecord.* ORM/simple interface for mapping Ruby objects to MySQL or PostgreSQL.
- *JSON.*
- *Typhoeus.* HTTP library specifically designed for high-speed parallel access to services.
- *RSpec.*

    /user-service
      /config.ru
      /config
        database.yml
      /db
        /migrate
      /models
      /spec
      Rakefile

> Rakefile

    require 'rubygems'
    require 'active_record'
    require 'yaml'

    desc 'Load the environment'
    task :environment do
      env = ENV['SINATRA_ENV'] || 'development'
      databases = YAML.load_filee('config/database.yml')
      ActiveRecord::Base.establish_connection(databases[env])
    end

    namespace :db do
      desc "Migrate the database"
      task(:migrate => :environment) do
        ActiveRecord::Base.logger = Logger.new(STDOUT)
        ActiveRecord::Migration.verbose = true
        ActiveRecord::Migrator.migrate("db/migrate")
      end
    end

> Spec on `GET /user`

    require File.dirname(__FILE__) + '/../service'
    require 'spec'
    require 'spec/interop/test'
    require 'rack/test'

    set :environment, :test
    Test::Unit::TestCase.send :include, Rack::Test::Methods

    def app
      Sinatra::Application
    end

    describe 'service' do
      before(:each) do
        User.delete_all
      end

*Only the public interface of the service is being tested.* Sinatra provides a convenient way to write tests against HTTP service entry points. These are the most important tests for the service because they represent what consumers see. *Tests can be written for the models and code behind the service, but the consumers of the service really only care about its HTTP interface.* Testing only at this level also makes the tests less brittle because they aren't tied to the underlying implementation.

However, the tests still require a user account to test against. Two possible options to deal with test data--service can either load a set of fixtures when started in a test environment, or ensure that `user create` works before the other tests are ran.

#### Implementing GET User

> `service.rb`: Contains the entire service.

    require 'rubygems'
    require 'activerecord'
    require 'sinatra'
    require 'models/user'

    env_index = ARGV.index("-e")
    env_arg = ARGV[env_index + 1] if env_index
    env= env_arg || ENV["SINATRA_ENV"] || "development"
    databases = YAML.load_file("config/database.yml")
    ActiveRecord::Base.establish_connection(databases[env])

    get '/api/v1/users/:name' do
      user = User.find_by_name(params[:name])
      if user
        user.to_json
      else
        error 404, { error: 'user not found'}.to_json
      end
    end

Starting the service: `ruby service.rb -p 3000`


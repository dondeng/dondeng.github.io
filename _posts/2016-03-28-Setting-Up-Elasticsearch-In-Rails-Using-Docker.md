---
layout: post
title:  "Setting up elastic search in Rails using Docker"
image: es_docker.png
date:   2016-03-28 12:35:35
categories: docker elasticsearch rails
---

I recently worked at moving our application search solution from using sphinx to elastic search. Our Sphinx setup was not 
optimal (and maybe we did not set it up correctly). In production with sphinx, we had to maintain a separate dummy app on 
our sphinx server just to run maintenance on our sphinx instance. We had long decided to separate our sphinx server from 
our application servers. I have become an avid fan of Docker and containerization especially in a micro-service architecurre like ours. Basically my opinion is 'if you are not using Docker, you should at least look into it'. Setting up Docker in development involved the following process.

{{ more }}

## Docker Setup

#### Docker setup in Mac OS X

Installing Docker on Mac not scope of this article but instructions can be found [here.][1] Installation using those instructions are straight forward and worked like a charm with no hiccups. There is no point of re-writing them here.

#### Docker setup in Ubuntu

Likewise, in Ubuntu, Instructions to install Docker on Ubuntu were straight forward as documented in the Docker [documentation][2]

## Elasticsearch setup

#### Mac OS X setup

Pull the latest elastic search image using:

    $docker pull elasticsearch
    
Run the image by mounting a volume to persist the elastic search indexing data. This is so that we can discard our elastic search container at any time without neccessarily losing our search index data since the data will reside in the host machine and not the container.

    $ docker run -d -p 9200:9200 -v "$PWD/esdata":/usr/share/elasticsearch/data elasticsearch
    
This should work fine. However, I ran into a documented problem regarding mounting docker volumes in Mac OS X. It turns out that there is a permissions problem with Mac OS X and mounting docker volumes when using docker machine on Mac. The error you may get will be similar to the one below:

{% highlight ruby %}

[2016-03-23 14:26:06,882][WARN ][bootstrap] unable to install syscall filter: seccomp unavailable: your kernel is buggy and you should upgrade
Exception in thread "main" java.lang.IllegalStateException: Unable to access 'path.data' (/usr/share/elasticsearch/data/elasticsearch)
Likely root cause: java.nio.file.AccessDeniedException: /usr/share/elasticsearch/data/elasticsearch
    at sun.nio.fs.UnixException.translateToIOException(UnixException.java:84)
    at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:102)
    at sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:107)
    at sun.nio.fs.UnixFileSystemProvider.createDirectory(UnixFileSystemProvider.java:384)
    at java.nio.file.Files.createDirectory(Files.java:674)
    at java.nio.file.Files.createAndCheckIsDirectory(Files.java:781)
    at java.nio.file.Files.createDirectories(Files.java:767)
    at org.elasticsearch.bootstrap.Security.ensureDirectoryExists(Security.java:337)
    at org.elasticsearch.bootstrap.Security.addPath(Security.java:314)
    at org.elasticsearch.bootstrap.Security.addFilePermissions(Security.java:259)
    at org.elasticsearch.bootstrap.Security.createPermissions(Security.java:212)
    at org.elasticsearch.bootstrap.Security.configure(Security.java:118)
    at org.elasticsearch.bootstrap.Bootstrap.setupSecurity(Bootstrap.java:196)
    at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:167)
    at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:285)
    at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:35)
Refer to the log for complete error details

{% endhighlight %}

There is a GitHub issue that explores the problem and solution which can be found [here][3]. The solution involves activating and mounting the shared folder in the virtual machine as NFS. The workaround documented in the issue above, involves doing the following:

Get the docker-machine-nfs script from [here][4]
Then run the command: 
    $ docker-machine-nfs dev-nfs     
When done, open the file /etc/exports and replace -mapall=$uid:$gid with -maproot=0
or just use
{% highlight bash %}
docker-machine-nfs default --shared-folder=/Users --nfs-config="-alldirs -maproot=0”  [updated setting]
{% endhighlight %}
Then restart nfsd
{% highlight bash %}
$ sudo nfsd restart
{% endhighlight %}
And finally run
{% highlight bash %}
$ eval "$(docker-machine env dev-nfs)"
{% endhighlight %}
You should then be able to start you container with:
{% highlight bash %}
docker run -d -p 9200:9200  --name k2_search -v "$PWD/esdata":/usr/share/elasticsearch/data elasticsearch
{% endhighlight %}
And verify with
{% highlight bash %}
$ docker ps
{% endhighlight %}
{% highlight bash %}
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                              NAMES
980a09759648        elasticsearch       "/docker-entrypoint.s"   About an hour ago   Up About an hour    0.0.0.0:9200->9200/tcp, 9300/tcp   k2_search
{% endhighlight %}
Remember Mac OS X has a light weight Linux VM machine to run docker so the container is reachable only via the IP of the VM. You can find out the IP of the VM using:
{% highlight bash %}
$docker-machine ip default
{% endhighlight %}
 (In this case my Virtual Machine is named default)

Lets say the ip is 192.168.99.100 , you should be able to visit your browser at port 9200 and get the following output:
{% highlight json %}
{
  "name" : "Longneck",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.2.1",
    "build_hash" : "d045fc29d1932bce18b2e65ab8b297fbf6cd41a1",
    "build_timestamp" : "2016-03-09T09:38:54Z",
    "build_snapshot" : false,
    "lucene_version" : "5.4.1"
  },
  "tagline" : "You Know, for Search"
}
{% endhighlight %}
Once Elasticsearch is setup, we can move to the Rails application.

## Rails setup

#### Setup
There is a Ruby elastic search library that we will be using ‘elasticsearch’ which is a wrapper for two separate libraries (elasticsearch-transport and elasticsearch-api)

- [elasticsearch-ruby][5]
- [elasticsearch-rails][6]

The elasticshearch-model gem, builds on top of the elastic search library. The key is to extend any model that you want to search via elastic search with Elasticsearch functionality. To do this we make use of ActiveSupport::Concern instrumentation so as to pull out Elastcisearch code from our model code and make it ‘dryer’. For example let us say we have an Account model which we want to expose to Elasticsearch.

In the models/concerns directory, create a file called account_indexer.rb and add the following:
{% highlight ruby %}
module AccountIndexer
  extend ActiveSupport::Concern

  included do
    include Elasticsearch::Model

    settings index: { number_of_shards: 1} do
      mappings dynamic: 'false' do
        indexes :number, analyzer: 'english', index_options: 'offsets'
        indexes :first_name, analyzer: 'english', index_options: 'offsets'
        indexes :last_name, analyzer: 'english', index_options: 'offsets'
      end
    end

    def as_indexed_json(options={})
      as_json(root: false, only: [:number, :first_name, :last_name])
    end

    def self.serach(query)
      __elasticsearch__.search(query)
    end
  end
end
{% endhighlight %}
Then in the Account model  code you would have
{% highlight ruby %}
class Account < ActiveRecord::Base
     include AccountIndexer
end
{% endhighlight %}
The code
{% highlight ruby %}
settings index: { number_of_shards: 1} do
   mappings dynamic: 'false' do
     indexes :number, analyzer: 'english', index_options: 'offsets'
     indexes :first_name, analyzer: 'english', index_options: 'offsets'
     indexes :last_name, analyzer: 'english', index_options: 'offsets'
   end
 end
{% endhighlight %}
Is used to configure the index with mappings. In this case, we are interested in indexing 3 columns of the Account model - number, first_name, last_name. You can view the mappings hash with the command:
{% highlight ruby %}
Account.mappings.to_hash
{:account=> {
    :dynamic=>"false",
   :properties=> {
     :number=> {:analyzer=>"english", :index_options=>"offsets", :type=>"string"},
     :first_name=> {:analyzer=>"english", :index_options=>"offsets", :type=>"string"},
     :last_name=> {:analyzer=>"english", :index_options=>"offsets", :type=>"string"}
    }
  }
}
{% endhighlight %}
####  Indexing
The defined settings and mappings are used to create an index with the desired configuration
Commands to create and refresh indexes are provided by the name spaced functions:
{% highlight ruby %}
Account.__elasticsearch__.create_index! force: true
Account.__elasticsearch__.refresh_index!
{% endhighlight %}
You can write rake tasks that perform these actions like so:
{% highlight ruby %}
desc 'bulk index the account model'
task bulk_index_accounts: :environment do
  puts 'creating accounts index...'
  Account.__elasticsearch__.create_index!
end
{% endhighlight %}
Before running the commands above, Elasticsearch needs to be bootstrapped in some kind of initialiser. Create an elastcisearch.yml file in your config directory with the following:


{% highlight ruby %}
---
development:
  host: 'http://localhost:9200'
  transport_options:
    request:
      timeout: 5

test:
  host: 'http://localhost:9200'
  transport_options:
    request:
      timeout: 5

production:
  host: 'http://localhost:9200'
  transport_options:
    request:
      timeout: 5
{% endhighlight %}  
Create an initializer elastcisearch.rb in config/initializers with the following:
{% highlight ruby %}
config = {
    host: 'http://localhost:9200/',
    transport_options: {
        request: { timeout: 5 }
    },
}

if File.exists?('config/elasticsearch.yml')
  config.merge!(YAML.load_file("#{Rails.root}/config/elasticsearch.yml")[Rails.env].deep_symbolize_keys)
end

Elasticsearch::Model.client = Elasticsearch::Client.new(config)
{% endhighlight %}
This will create an Elasticsearch client that will be used by ALL models in the application.    
After creating the index, you could do the initial import of the data using the command.
{% highlight ruby %}
Account.import
{% endhighlight %}
However, recall that this call will be making remote api calls to the Elasticsearch server. At scale with thousands of records this is not optimal. Thankfully Elasticsearch provides a bulk indexing functionality to mitigate this. You could create a module to easily do bulk indexing of existing records like the one below:
{% highlight ruby %}
module Search
  module BulkIndexer
    def self.import(klass)
      klass.constantize.find_in_batches do |batch_objects|
        bulk_index(klass, batch_objects)
      end
    end

    def self.prepare_records(records)
      records.map do |record|
        {index: {_id: record.id, data: record.as_indexed_json}}
      end
    end

    def self.bulk_index(klass, records)
      klass.constantize.__elasticsearch__.client.bulk({
          index: klass.constantize.__elasticsearch__.index_name,
          type: klass.constantize.__elasticsearch__.document_type,
          body: prepare_records(records)
      })
    end
  end
end
{% endhighlight %}
With this for example, a call to bulk index the Account model would be:
{% highlight ruby %}
Search::BulkIndexer.import('Account')
{% endhighlight %}
#### Searching
A simple search now takes the form 
{% highlight ruby %}
Account.search('query_string')
{% endhighlight %}
We are using just the basic search of elasticsearch as can be seen in the function 
{% highlight ruby %}
def self.serach(query)
  __elasticsearch__.search(query)
end
{% endhighlight %}
You can customize the search by sending in more options using the Elasticsearch DSL but we will stick with the plain vanilla search and you can look at the search DSL documentation. There are many options of retrieving the search results including the related records eg.
{% highlight ruby %}
records = Account.search('query_string').records
{% endhighlight %}
#### Updating indices
As the application is used, invariably, the indices need to be updated when records are created/edited/deleted. The gem has callbacks that can be invoked automatically by including 
{% highlight ruby %}
include Elasticsearch::Model::Callbacks
{% endhighlight %}
in the model (or in our case in the concern we are including). However, this will result in additional overhead when performing crud operations because the indexing operation involves doing an API call. It is prudent to pass this to a back ground job using Sidekiq or Resque. Since we use Resque, a background job worker to update indices could look something like this:

{% highlight ruby %}
 module Search
    class Indexer
      @queue = :indexer

      def async_index_operation(operation, record_id, klass)
        begin
          Resque.enqueue(KopoKopo::Search::Indexer, operation, record_id, klass)
        rescue => ex
          indexing_logger.error  "E [#{Time.now.utc.iso8601}] Error: Error in Enqueueing #{klass} #{operation} operation: #{ex.message}"
        end
      end

      def self.perform(operation, record_id, klass)
        begin
          @es_client = Elasticsearch::Model.client
          index_object(operation, record_id, klass) if @es_client
        ensure
        end
      end

      def self.index_object(operation, record_id, klass)

        case operation.to_s
          when /index/
            record = klass.constantize.find(record_id)
            @es_client.index(index: ActiveSupport::Inflector.pluralize(klass.downcase),
                             type: klass.downcase,
                             id: record_id,
                             body: record.as_indexed_json) if record
          when /delete/
            @es_client.delete(index: ActiveSupport::Inflector.pluralize(klass.downcase),
                              type: klass.downcase,
                              id: record_id)
          else raise ArgumentError, "Unknown operation '#{operation}'"
        end
      end

      def indexing_logger
        @indexing_logger = Logger.new( "#{Rails.root}/log/indexer-job.log", 'monthly')
      end
    end
  end
{% endhighlight %}
Add callbacks to the Account concern that will be invoked on saving an Account object or deleting an Account object

{% highlight ruby %}
after_save {Search::Indexer.new.async_index_operation('index', self.id, self.class.to_s)}
after_destroy {Search::Indexer.new.async_index_operation('delete', self.id, self.class.to_s)}
{% endhighlight %}

  [1]:  https://docs.docker.com/engine/installation/mac/
  [2]: https://docs.docker.com/engine/installation/linux/ubuntulinux/
  [3]: https://github.com/boot2docker/boot2docker/issues/581
  [4]: https://github.com/adlogix/docker-machine-nfs
  [5]: https://github.com/elastic/elasticsearch-ruby
  [6]: https://github.com/elastic/elasticsearch-rails


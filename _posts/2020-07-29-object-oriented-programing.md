---
layout: post
title: Using SOLID principles to refactor a Ruby script
tags: [Ruby, OOP]
---



I enjoy automating tasks that take time and effort to do manually. By automating
these workflows, I'm able to help out my teammates to free up time to focus on
other things that are important. I recently automated a task to retrieve emails
from a database and then add the emails to a database and then upload them to
Box, a cloud storage service.

So I created a Ruby script to automate the process which I did fairly quickly.
Although I wrote the script in a fair amount of time the quality of the code
wasn't good, it was awful. I wasn't going to let that piece of software in that
state so I decided to be guided by Object Oriented Design principles and
refactor that piece of code that didn't let me have a good night sleep.

{% highlight rb %}
require 'json'
require 'date'
require 'dotenv/load'
require 'fileutils'
require 'csv'
require 'logger'
require './my_sql'
require './box_api'
require 'pry'

begin
  year = ARGV[0]
  month = ARGV[1]
  day = ARGV[2]

  search_timestamp = Time.new(year, month, day).utc

  db_client = MySQL.new(search_timestamp)

  emails = db_client.get_emails_from_db
  return 'No new emails found' if emails.entries.empty?

  box = BoxApi.new(ENV['BOX_USER_ID'])

  date = DateTime.now.strftime("%m-%d-%Y").to_s
  file_name = "access-emails-#{date}"

  CSV.open("./tmp/#{file_name}.csv", "wb") do |csv|
    emails.entries.each do |entrie|
      csv << [entrie.values[0], entrie.values[1]]
    end
  end

  box.upload_file_to_box("./tmp/#{file_name}.csv", file_name, ENV['BOX_FOLDER_ID'])

  FileUtils.rm("./tmp/#{file_name}.csv")

  puts "successfully uploaded CSV file to Box"
rescue StandardError => e
  logger = Logger.new(STDOUT)
  logger.level = ENV.fetch('LOG_LEVEL', Logger::INFO)
  logger.datetime_format = '%Y-%m-%d %H:%M:%S '
  logger.error("msg: #{e}, trace: #{e.backtrace.join("\n")}")
end
{% endhighlight %}

When I turn around and look at the above code I see a lot of dependencies,
different concerns being mixed, and a lack of clarity of whats going on.
Now, let's think about the different tasks this software is doing.

* Configuration
* File Backup
* File Upload

Now when discussing configuration, we can use
[OpenStruct](https://ruby-doc.org/stdlib-2.5.1/libdoc/ostruct/rdoc/OpenStruct.html)
which is a data structure similar to a hash. This way we can add all the
configuration and have it in one place. Adding clarity and cohesion to what the
configuration is and what it intends to achieve.

{% highlight rb %}
  require 'ostruct'

  config = OpenStruct.new(
    year: ARGV[0],
    month: ARGV[1],
    day: ARGV[2],
    box_user_id: ENV['BOX_USER_ID'],
    box_folder_id: ENV['BOX_FOLDER_ID'],
    local_path: ENV['LOCAL_PATH']
  )
{% endhighlight %}


This is so much better already, I can now get the year from `config.year`.
In this way improving readability and increasing cohesion in the configuration.


Now the other concern is the file backup. We can improve by creating a class
with follows the [single-responsibility principle](Single-responsibility
principle) of backing up the file.

{% highlight rb %}
  class BackupFile
    def initialize(rows:, date: DateTime.now.strftime("%m-%d-%Y").to_s, directory: "./tmp")
      @rows = rows
      @date = date
      @directory = directory
    end

    def save
      CSV.open(full_path, "wb") do |csv|
        rows.each do |entry|
          csv << [entry.values[0], entry.values[1]]
        end
      end
    end

    def full_path
      File.join(@directory, filename)
    end

    def delete
      FileUtils.rm(full_path)
    end

    private

    attr_reader :rows, :date

    def filename
      "access-emails-#{date}"
    end
  end
{% endhighlight %}


Notice how we use [Dependency
Injection](https://sandimetz.com/blog/2009/03/21/solid-design-principles)
passing `DateTime` as an argument into the `BackupFile` class.

Then we have the other concern of Upload. Now imagine you wanted to use
`Dropbox` or any other cloud storage service? We can create a class that takes
any client as a parameter and we can use dependency injection to pass the cloud
storage's client as an argument into the class.

{% highlight rb %}
  class Uploader
    def initialize(client:, path:, remote_folder:)
      @client = client
      @path = path
      @remote_folder = remote_folder
    end

    def upload
      client.upload(path, file_name, remote_folder)
    end

    private

    attr_reader :client, :path, :remote_folder

    def file_name
      File.basename(path)
    end
  end


  class BoxAdapter
    def initialize(client:, box_user_id:)
      @client = client.new(box_user_id)
    end

    def upload(path, file_name, remote_folder)
      client.upload_file_to_box(path, file_name, remote_folder)
    end

    private

    attr_reader :client
  end
{% endhighlight %}


Notice here how I used the [Adapter Design
Pattern](https://sourcemaking.com/design_patterns/adapter) To match the `BoxApi`
interface to the `Uploader` class. If we wanted to use Dropbox, we would just
need to create a `DropboxAdapter` or `OtherStorageServiceAdapater`. We can now
also use dependency injection to pass these clients and not be tied to one
specific clod storage service, like this:
`Uploader.new(BoxAdapter.new(args),file.full_path, config.box_folder_id)`
which is much better.

Now the runner script is much cleaner and better. 

{% highlight rb %}
  search_timestamp = Time.new(config.year, config.month, config.day).utc
  db_client = MySQL.new(search_timestamp)
  emails = db_client.get_emails_from_db
  return 'No new emails found' if emails.entries.empty?

  file = BackupFile.new({ rows: emails.entries, directory: config.local_path })
  file.save

  uploader = Uploader.new(BoxAdapter(BoxApi.new,config.box_user_id),
                          file.full_path,
                          config.box_folder_id)

  uploader.upload

  file.delete
{% endhighlight %}

We have touched on different Object Oriented Design principles that have
increase the quality of the code bringing the software to a state where
dependencies are managed and more adaptable to future changes.

Shout out to [Christian Bruckmayer](https://twitter.com/bruckmayer) for his
mentorship and feedback. Fun fact I wrote this blog while listening to [Que No
Quede Huella](https://www.youtube.com/watch?v=U3oLCYVlXwI) it reminds me of back home!















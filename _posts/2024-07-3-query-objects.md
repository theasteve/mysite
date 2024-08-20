---
layout: post
title: Refactoring the model layer by extracting queries from models
tags: [Ruby, Rails]
---

I recently started reading "A Layered Design for Ruby on Rails Applications" by Vladimir Dementyev. I have always encountered some pain points when working with Rails applications when the model and controller layers grow in scope. One specific problem is adding complex querying logic to the model. At first, although it does not look like a dangerous pursuit, adding such a burden on the model starts to crack into the code quality. Thankfully, the book talks about the use of Query Objects and how we can use them to extract complex queries from our models.

I want to also emphasize that query objects is specificlly use for complex queries that might better suited to be extracted into its own class. Not every query needs to be a query object. 

For loominex.io, a maintenance management system, I have the following model:

{% highlight rb %}
class WorkOrder < ApplicationRecord
  belongs_to :workspace
  belongs_to :equipment, optional: true
  has_many :tasks, dependent: :destroy

  has_many_attached :files

  scope :search, -> (query) {
    where('name ILIKE ?', "%#{query}%")
  }

  scope :open, -> { where.not(status: 'Completed').where('due_date >= ?', Date.current) }

  scope :delayed, -> { where(status: ['Not Started', 'In Progress']).where('due_date < ?', Date.current) }

  scope :completed, -> { where(status: 'Completed') }

  scope :urgent, -> { where(priority: 'Urgent').where.not(status: 'Completed') }
end
{% endhighlight %}

As you can see from the example above, our model is increasing in scope. Increasingly, I'm adding more responsibility to the model layer. Although these queries are not as complex, with time and requirements increasing, the WorkOrder model can get out of hand. This is where the bugs start showing up.

Thankfully, query objects help in extracting some of these queries from the models. It would help us make the WorkOrder model a bit leaner, improve the health of the codebase by removing the querying from the model, and help us integrate a new convention that would make it easier to test the app and decouple the application. Let's look into it:

{% highlight rb %}
class ApplicationQuery
  class << self 
    def query_model
      name.sub(/::[^\:]+$/, "").safe_constantize
    end

    def resolve(...) = new.resolve(...)
  end

  private attr_reader :relation

  def initialize(relation = self.class.query_model.all)
    @relation = relation
  end

  def resolve(...)
    relation
  end
end
{% endhighlight %}

I can now add my query objects under the model directory in the `work_order` directory.

├── work_order
│ ├── open_status_query.rb
│ └── urgent_status_query.rb

ruby
Copy code

{% highlight rb %}
class WorkOrder
  class OpenStatusQuery < ApplicationQuery
    def resolve(due_date = Date.current)
      relation.where.not(status: 'Completed').where('due_date >= ?', due_date)
    end
  end
end

class WorkOrder
  class UrgentStatusQuery < ApplicationQuery
    def resolve
      relation.where(priority: 'Urgent').where.not(status: 'Completed')
    end
  end
end
{% endhighlight %}

With this, I can now call:

{% highlight rb %}
irb(main):001:0> WorkOrder::UrgentStatusQuery.resolve
  WorkOrder Load (1.0ms)  SELECT "work_orders".* FROM "work_orders" WHERE "work_orders"."priority" = $1 AND "work_orders"."status" != $2  [["priority", "Urgent"], ["status", "Completed"]]
=>
[#<WorkOrder:0x000000010d536048
  id: 13,
  equipment_id: nil,
  workspace_id: 2,
  ...
]
{% endhighlight %}

{% highlight rb %}
irb(main):001:0> WorkOrder::OpenStatusQuery.resolve
  WorkOrder Load (3.5ms)  SELECT "work_orders".* FROM "work_orders" WHERE "work_orders"."status" != $1 AND (due_date >= '2024-07-03')  [["status", "Completed"]]
=>
[#<WorkOrder:0x0000000104cdd2f0
  id: 31,
  equipment_id: 15,
  workspace_id: 6,
  name: "sample test",
  description: nil,
  priority: ni
  ...
]
{% endhighlight %}

We can now conclude that this refactored version makes our codebase healthier, makes it easier to test, removes complex queries from the model layer, and improves our code quality.

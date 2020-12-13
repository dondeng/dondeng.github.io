---
title: "Partial Indexes with PostgreSQL and Rails"
image: postgres_aws_3.png
date: 2014-07-17 12:35:35
categories:
  - rails
  - postgresql
---

So what happens when you run into a situation that you really need to be paranoid about making duplicate records into the db?
When dealing with financial systems, this is more often the case (You never want to have a scenario that you end up double spending).
Rails model validations more often takes care of this, but it is technically possible that something slips through and God forbid it i
s that one million dollar transaction.

{{ more }}

Having a validation such as:

{% highlight ruby %}
validates :name, :presence => true, :uniqueness => true
{% endhighlight %}

should do the trick. Upon trying to save a record with a duplicate name, the record would not save, and .save would return false.
However, there are some exceptional conditions that can result in the application level validations being by passed. This could happen
with race conditions such as multiple application servers hitting a database. Having suffered this situation and if you get as paranoid
as I am, then you would probably also bring down the validation to the database level by making use of unique indexes.

To spice things up, the records for which we want to avoid duplicates have uniqueness being defined by multiple columns hence the necessity
of a compound index. As an example, say you have a model called Payment with the following attributes: id, amount, order_id, payment_date,
description, status. order_id is a foreign key pointing to an Order item. Since it is possible to make multiple payments per order, there
is a one to many relationships between orders and payments. All payments transition from a status of ‘Scheduled’ to ‘Paid’ after they are confirmed.
The migration file for payments would look like this:

{% highlight ruby %}
class CreatePayments < ActiveRecord::Migration
def change
create_table :payments do |t|
t.float :amount
t.integer :order_id
t.date :payment_date
t.string :description
t.string :status
t.timestamps
end
end
end
{% endhighlight %}

In our world that is filled with paranoia we would want to add a unique index to ensure that duplicates are avoided at the db level.
To do this we would create a compound index.

{% highlight ruby %}
class CreateUniqueIndexForPayments < ActiveRecord::Migration
def change
add_index :payments, [:order_id, :amount, :payment_date, :status] name: 'unique_index_to_avoid_duplicate_payments', unique: true
end
end
{% endhighlight %}

Let’s analyze what this index will prevent. You would not be able to create a payment of the same amount being applied to the same order
on the same day. Unless of course the first payment’s status flips from ‘Scheduled’ to ‘Paid’ before the second payment is made.
Yes this is a simplistic example and it prevents the possibility that two legitimate consecutive payments of the same amount to can
be made against the same order within a short period of time. Though theoretically possible, that scenario would be practically very rare,
and in many cases might be an inconvenience worth dealing with if the upside is completely guaranteeing that a double payment does not
get registered in error.

How can this index be improved further? Enter ‘Partial Indexes’. The payments table will be undoubtedly be a very huge table. However,
the uniqueness we are concerned about above, only applies to ‘Scheduled’ payments. The payments table is bount to have hundreds or
thousands of ‘Paid’ status payments whereas the payments with a ‘Scheduled’ status will always be a very small subset of the payments table.
Using a partial index that takes into account only the status ‘Scheduled’ would significantly increase performance and efficiency by
reducing the index’s size thus taking up less storage. The index is also easier to maintain and faster to scan. To know about partial
indexes you can read the original white paper by Michael Stonebraker located [here](http://db.cs.berkeley.edu/papers/ERL-M89-17.pdf).
Modifying the migration to cater for a partial index on the status field yield

{% highlight ruby %}
class CreateUniqueIndexForPayments < ActiveRecord::Migration
def change
add_index :payments, [:order_id, :amount, :payment_date, :status] name: 'unique_index_to_avoid_duplicate_payments', where: "status = 'Scheduled'", unique: true
end
end
{% endhighlight %}

Here is a catch though; I was trying this in rails 3.2 and it happens that partial index migrations are not supported. In Rails 4.x there
is in built suppot for the above migration. To get around this for rails 3.2, we used the gem pg_power which is billed as an ActiveRecord extension
to get more out of PostgreSQL.
In psql, our table would look like this:-

{% highlight ruby %}
dennis-home-# \d payments
Table "public.payments"
Column | Type | Modifiers
--------------+-----------------------------+-----------------------------------------------------
id | integer | not null default nextval('payments_id_seq'::regclass)
amount | double precision |
order_id | integer |
payment_date | date |
description | character varying(255) |
status | character varying(255) |
created_at | timestamp without time zone |
updated_at | timestamp without time zone |
Indexes:
"payments_pkey" PRIMARY KEY, btree (id)
"unique_index_to_avoid_duplicate_payments" btree (order_id, amount, payment_date, status) WHERE status::text = 'Scheduled'::text
{% endhighlight %}

With this we now are able to avoid in-inadvertent duplicate payments which might otherwise be catastrophic.


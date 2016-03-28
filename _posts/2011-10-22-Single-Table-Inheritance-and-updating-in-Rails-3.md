---
layout: post
title:  "Single table inheritance and updating in Rails 3"
image: postgres_loves_ruby.png
date:   2011-10-22 12:35:35
categories: rails 
---


For my first blog post, I may as well start by talking about a recent oddity that I encountered working on our rails application (at Kopo Kopo, Inc). 
It probably may not be an oddity for some rails aficionados but well, it was a head scratcher so I decided to share.

So basically here is the setup. Say you have a model called Person and two other models that inherit from Person called Student and Teacher respectively

{{ more }}

{% highlight ruby %}
class Person < ActiveRecord::Base
end

class Teacher < Person
end

class Student < Person
end
{% endhighlight %}

Let’s say you want to use only one form (view) to be able to create either a Student or Teacher and the object to be created 
would have to depend on the user selecting an option in a select box to determine what model will actually be created. 
So in essence the creation form would look somewhat like the following:-

{% highlight ruby %}
<%= form_for(@Person, :action => 'create') do |f| %>
  <label> Chose Type </label>
  <%=  f.select('type', { 'Teacher'=>'Teacher', 'Student'=>'Student'})%><br/>
  <label>First Name: </label>
  <%= f.textfield :first_name, size=>30, :autocomplete=>"off" %><br/>
  <label>Last Name</label>
  <%= f.textfield :last_name, size=>30, :autocomplete=>"off" %><br/>
  <!-- Fields specific to Teacher -->
  <!-- Fields specific to Student -->
<% end %>
{% endhighlight %}

Note that the object that is used to call the form_for is a ‘Person’ object so the forms fields are actually bound to a Person object. 
The same would go for the editing view

{% highlight ruby %}
<%= form_for(@Person, :action => 'update') do |f| %>
  <label> Chose Type </label>
  <%=  f.select('type', { 'Teacher'=>'Teacher', 'Student'=>'Student'})%><br/>
  <label>First Name: </label>
  <%= f.textfield :first_name, size=>30, :autocomplete=>"off" %><br/>
  <label>Last Name</label>
  <%= f.textfield :last_name, sizez=>30, :autocomplete=>"off" %><br/>
  <!-- Fields specific to Teacher -->
  <!-- Fields specific to Student -->
<% end %>
{% endhighlight %}

You would expect the Persons controller to have:-

{% highlight ruby %}
class PersonsController < ApplicationController
  def create
    if params[:person][:type] == 'Teacher'
      @person = Teacher.new(params[:person].merge(:type=>'Teacher'))
    elsif params[:person][:type] == 'Student'
      @person = Student.new(params[:person].merge(:type=>'Student'))
    end

    if @person.save
      flash[:notice] = "Added Successfully"
    end
  end

  def update
    @person = Persons.find(params[:id])
    if @person
      @person.update_attributes(params[:person])
    end
  end
end
{% endhighlight %}

The interesting/tricky part comes up in the update method. It so happens that since the objects were created as either a ‘Teacher’ or ‘Student’ 
entity but the edit form is bound to a ‘Person’ object and the update_attributes is also called on the parameters of a ‘Person’ object 
(the superclass), the update does not work and the fields are not updated. For the update to work, you would have to call update_attributes 
on the particular object being updated. Remember this case is probably arising because we want to use the same form to update both a ‘Teacher’ 
and ‘Student’ object. We thus have to change the update method like so:-

{% highlight ruby %}
def update
  @person = Persons.find(params[:id])
  if @person
    if @person.instance_of?(Teacher)
      @person.update_attributes(params[:teacher])
    elsif @person.instance_of?(Student)
      @person.update_attributes(params[:student])
    end
  end
end
{% endhighlight %}
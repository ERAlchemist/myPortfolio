---
layout: post
title: LABrador
excerpt_separator: "@@@"
feature-img: "img/sample_feature_img.png"
thumbnail-path: "img/lab.png"
short-description: A RubyOnRails web appllication to manage the request and retrieval of wine samples at a large scale production winery

---
<img src="/img/logo_big2.png" style="margin: auto"><br>
**Summary**<br>
    This was built as my cornerstone project of the Bloc Full-Stack Web Development program.  I chose to build an application to be used at my day job where I work in Quality Control for a large scale production winery.  I wanted to build something to help solve one of the many logistical challenges faced by a facility that produces millions of gallons of wine a year. The problem I chose to tackle was related to sampling.  As anyone familiar with wine production will tell you, each lot of wine requires regular testing and tasting thoughout the production process.  With hundreds of tanks and thousands of barrels to sample, it can be daunting task to keep track of all the samples that are needed. I wanted to build a basic 'to-do' like app to help keep track of sample requests and allow the user to mark off each sample as it is collected and close out requests when complete.<br><br>
**User Stories**<br>
As a User I want to be able to....<br>
    * Create sample requests and specify details such as title and description.<br>
    * Assign a due date to the request to inidicate when it needs to be completed.<br>
    * Add samples to the request and specify the Tank and LotID of each sample<br>
    * Upload a large list of samples by uploading a CSV file containing the Tanks and Lot IDs<br>
    * Add reccuring sample requests that would repeat on a set interval<br>
    * Mark off samples as complete as they are collected<br>
    * Indicate when a tank is empty and cannot be sampled<br>
    * Delete erroneous or unneccessary samples from the list<br>
    * See a calendar view of what requests are due on which days<br>
    * Close out a request once it is complete<br><br>
**Problem**<br>
So one of the biggest problems I needed to solve on this project was to figure out how to create a recurring sample request (e.g. a request that would repeat on a set interval.)  I relealized all it really needed to do was create a duplicate request with the same parameters and samples after the previous request was closed and increase the `:due_date` by one interval length.  So for this i turned to the help of the [ameoba gem](https://github.com/amoeba-rb/amoeba) which allows for easily copying Active Record objects including their children. This gem was simple and easy to use.  I just needed to place the amoeba method call in the Request model and indicate which fields to set to null (`:time_completed` and `:complete`) and inidicated to clone the associated `:samples` <br>
{% highlight ruby %}
    class Request < ApplicationRecord
        has_many :samples

        amoeba do
            enable
            nullify [:time_completed, :complete]
            clone :samples
        end
{% endhighlight %}<br>
The amoeba method call would also need to be added to the Sample model as well.  The `:is_empty`, `:time_completed`, and `:complete` fields would be nullified.
{% highlight ruby %}
class Sample < ApplicationRecord
    belongs_to :request, optional: true
    amoeba do
        enable
        nullify [:is_empty, :time_completed, :complete]
      end
{% endhighlight %}<br>
In order to keep track of the recurrence, two fields were added to the Request model a boolean `:is_recurring` and and integer `:recurrence_interval`
{% highlight ruby %}
    class AddIsRecurringToRequests < ActiveRecord::Migration[5.1]
        def change
            add_column :requests, :is_recurring, :boolean
        end
    end
{% endhighlight %}<br>  
{% highlight ruby %}
    class AddRecurrrenceIntervalToRequests < ActiveRecord::Migration[5.1]
        def change
            add_column :requests, :recurrence_interval, :integer
        end
    end
{% endhighlight %}<br>
Since the duplication would occur when the request is closed out, this would have to be indicated in the `:close_out` method of the `requests_controller` <br>
{% highlight ruby %}
    def close_out
        @request.update_attribute(:time_completed,  Time.now.in_time_zone)
        @request.update_attribute(:complete,  true)
        if @request.is_recurring
            new_request = @request.amoeba_dup
            new_request.due_date = @request.due_date + @request.recurrence_interval.days
            new_request.save!
        end
        redirect_to @request, notice: "Request Completed!"
    end
{% endhighlight %}
Finally, the view needed to allow for the user to specify values for the `is_recurring` and `recurrence_interval` fields.
{% highlight erb %}
<div class="field" id="recurring" >
	<label uk-tooltip="title: If set to true a new request will be generated with the same title, description, and samples with a due date one interval length after current due date after current request closes"> 
		Is Recurring?
	</label>
	<%= f.select :is_recurring, [true,false], :selected => false %>
</div><br>
<div class="field" id="interval" style="display:none">
  <%= f.label :recurrence_interval %>
  <%= f.select :recurrence_interval, [1,2,3,4,5,6,7] %><label> day(s)</label>
</div><br>
<div class="actions" id="new-request">
  <%= f.submit %>
</div>
{% endhighlight %}<br>
<img src="/img/schedule_view.png">
<img src="/img/side_nav.png">
<img src="/img/RequestList.png">
<img src="/img/import.png">
<img src="/img/import_success.png">
**Conclusion**<br>
The LABrador app was the first web application completely conceived, designed, built, and deployed myself. It really pushed my understanding of Front-End and Backend development.  It was a great feeling to take an idea and take it through step by step and turn it into a working application.  It was an extra bonus seeing people use it and getting their feedback on what they liked and didn't like about it.  This allowed me to refine it into an even better final product. <br>

The code repository for the app can be found [on Github](https://github.com/eralchemist/labrador).<br>
A working demo can be viewed [here](http://labrador-demo.herokuapp.com).


--- 
layout: post
title: "Auto Blogging with Rake & Rails"
category: rails
---

I have a long running blog that I struggle to keep current. The blog centers on mountain bike racing scene here in Northern California, so the content needs to stay timely as there are races every weekend all year long. Unfortunately, I don't have a lot of free time to devote to it. Long story short, I wanted to automatically create a post every Thursday morning to let people know what races were being held that weekend. You can see an example of what I wanted at <a href="http://norcalmtnbikeracing.blogspot.com/2011/03/norcal-mtb-racing-this-weekend-3-mar-8.html" target="_blank">NorCal MTB Racing blog</a>.

The basic idea was to create a <span class="code-inline">cron</span> job that pulls a list of races (with a link for details) from my rails application's database, plot the events on map, add some text and then post it to my blog on Blogger. Most of this is pretty standard rails stuff, but I did need some help for the map and posting.

For the maps, I used <a href="http://code.google.com/apis/maps/documentation/staticmaps/" target="_blank">Google Static Maps</a>. This allows you to embed a Google Maps image on your webpage just like a normal image. All you need to do it feed it the right parameters. For example, this HTML code:
{% highlight ruby %}
<img alt="NorCal MTB Race Event Map" src="http://maps.google.com/maps/api/staticmap?size=600x200&amp;maptype=roadmap&amp;sensor=false&amp;markers=color:blue|label:1|38.7067,-122.903&amp;markers=color:blue|label:2|37.7756,-122.438&amp;markers=color:blue|label:4|38.028,-121.885" />
{% endhighlight %}
Produces this map:

<img alt="NorCal MTB Race Event Map" src="http://maps.google.com/maps/api/staticmap?size=600x200&amp;maptype=roadmap&amp;sensor=false&amp;markers=color:blue|label:1|38.7067,-122.903&amp;markers=color:blue|label:2|37.7756,-122.438&amp;markers=color:blue|label:4|38.028,-121.885" />
  
To post to Blogger, I used the <span class="code-inline">blogger gem</span> (<a href="http://blogger.rubyforge.org/" target="_blank">installation instructions</a>). There are other ruby gems out there that interface with Blogger, but this was by far the most straight forward. To use the gem, you need username, password and blog ID. I found my blog ID through an <a href="http://www.google.com/support/forum/p/blogger/thread?tid=44fc0d69e2a4f283&hl=en" target="_blank">answer in the blogger support forum</a>:
<blockquote>
  <p>Got to your dashboard and click Settings for the blog you want the ID of. In the address bar you should then see something like this:</p>
  <p>http://www.blogger.com/blog-options-basic.g?blogID=9101099271220909428</p>
  <p>The long number at the end of the URL is your blog's ID number.</p>
</blockquote>

I decided that this would be best implemented as a rake task, so I could full access to all the rails application environment (especially the database connection). Here is the complete rake task:

{% highlight ruby %}
namespace :blogger do
  desc "post weekend update to blogger"
  task :weekend_update  => :environment do
    require 'blogger'
    
    # preferences for blogger login
    username = 'sample@gmail.com' # change this to your username
    password = 'xxx129304xxxx' # change this to your password
    blog_id = '123456' # change this to your blogid

    # my entry preferences
    days_ahead = 5 # how many days of events ?
    blog_entry = "For those of you making your weekend plans, 
                  here is quick rundown of this weekend\'s MTB racing schedule 
                  for NorCal:\n\n"
    
    # google static map preferences 
    base_url = "http://maps.google.com/maps/api/staticmap?"
    map_params = ["size=400x400", "maptype=roadmap", "sensor=false"]

    # find all events in the mtbcalendar.com db for this weekend
    @events = Event.active.find_tagged_with(
        "norcal", 
        :conditions => ["start_date >= CURRENT_DATE AND start_date < ?", 
                        days_ahead.days.from_now], 
        :order => "start_date asc")

    # loop through events adding place markers to the google static map URL
    x = 0
    @events.each do |marker|
      x += 1
      map_params << "markers=color:blue|label:#{x}|#{marker.lat},#{marker.lng}"
      blog_entry += "#{x}. [#{marker.name}](http://mtbcalendar.com/events/#{marker.id})
                    (#{marker.start_date.to_s(:compact)})\n"
    end # marker loop

    # add the google static map to post body
    map_pic_url = base_url + map_params.join('&')
    blog_entry += "\n![NorCal MTB Race Event Map](" + map_pic_url + ")"

    # create title for the post
    blog_title = "NorCal MTB Racing This Weekend: 
                  #{Date.today.to_s(:short)} - 
                  #{days_ahead.days.from_now.to_date.to_s(:short)}, 
                  #{Date.today.year}"

    # login into blogger and post this draft
    account     = Blogger::Account.new(username,password)
    new_post    = Blogger::Post.new(:title      => blog_title, 
                                    :content    => blog_entry,
                                    :formatter  => :rdiscount,
                                    :draft      => true)
    account.post(blog_id,new_post)

  end # task    
end # namespace
{% endhighlight %}

Although I did this with a rake task, you could also make it just a regular ruby script. However, then you would need to handle all of the database login and queries yourself. 
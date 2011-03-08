def create_new_file(path, title)
    post = <<-HTML
--- 
layout: post
title: "TITLE"
category: example
---

HTML
    post.gsub!('TITLE', title).gsub!('DATE', Time.new.to_s)
    File.open(path, 'w') do |file|
      file.puts post
    end
end


desc 'create a draft with $rake draft title="title" [slug="slug"] future=2'
task :draft do
  require "rubygems"
  require "chronic"
  
  target_dir = "_drafts"
  title = ENV["title"] || "New Title"
  slug = (ENV["slug"] ? ENV["slug"].gsub(' ','-').downcase : nil) || title.gsub(' ','-').downcase
  filename = "#{Time.new.strftime('%Y-%m-%d')}-#{slug}.markdown"
  future = ENV["future"].to_i || 1
  stamp = Chronic.parse("in #{future} days").strftime('%Y-%m-%d')
  filename = "#{stamp}-#{slug}.markdown"
  path = File.join(target_dir, filename)
  create_new_file(path, title)
  puts "new draft generated in #{path}"
  system "open -a textmate #{path}"
end

desc 'create a new post with $rake post title="title" [slug="slug"]'
task :post do
  target_dir = "_posts"
  title = ENV["title"] || "New Title"
  slug = (ENV["slug"] ? ENV["slug"].gsub(' ','-').downcase : nil) || title.gsub(' ','-').downcase
  filename = "#{Time.new.strftime('%Y-%m-%d')}-#{slug}.markdown"
  path = File.join(target_dir, filename)
  create_new_file(path, title)
  puts "new post generated in #{path}"
  system "open -a textmate #{path}"
end

desc 'ping aggregators'
task :ping do
  `curl "http://www.google.com/webmasters/tools/ping?sitemap=http%3A%2F%2Fken.pepple.info%2Fsitemap.xml"`
end

desc 'generate tag pages'
task :tags do
    puts 'Generating tags...'
    require 'rubygems'
    require 'jekyll'
    include Jekyll::Filters

    options = Jekyll.configuration({})
    site = Jekyll::Site.new(options)
    site.read_posts('')

    html =<<-HTML
---
layout: page
title: Tags
---

    HTML

  site.categories.sort.each do |category, posts|
    html << <<-HTML
    <h4 class="tags">#{category}</h4>
    HTML

    html << '<ul class="posts">'
    posts.each do |post|
      post_data = post.to_liquid
      html << <<-HTML
        <li>
          <a href="#{post.url}">#{post_data['title']}</a> (#{date_to_string post.date})
        </li>
      HTML
    end
    html << '</ul>'
  end

  File.open('tags.html', 'w+') do |file|
    file.puts html
  end

  puts 'Done.'
end
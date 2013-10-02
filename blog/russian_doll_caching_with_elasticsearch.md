Russian Doll Caching with ElasticSearch
=======================================

Here at Active, we eat our own dog food, which means our primary data
store for information about our events on Active.com comes through the same API that
you all use. We're also religiously focused on improving the load times
of our applications, especially of Active.com itself. For those of you
who don't know, Active.com is a ruby application, built with Rails. Like
most frameworks, Rails can lose a lot of time to compiling it's view
templates with new information, and we're no expception. On our event
details pages, we spend an inordinate amount of time doing so, then
cache it at Memcached for ~15 mins, which helps the overall response
times (and during high traffic events). However when people request an
event that isn't already stored at this cache that full recalculation
has to occur, which hurts our 98th percentile numbers, and more
importantly hurts our users who just come to check out a new event.

This is all made more complicated by the fact that each of our event
assets have child assets representing sub events, pricing information,
etc. Rails 4 solved this problem by implementing Russian Doll Caching (a
nested form of generational caching explained well [here](http://blog.remarkablelabs.com/2012/12/russian-doll-caching-cache-digests-rails-4-countdown-to-2013)). Obviously this would be our first choice, however we would rather avoid linking directly to our database, as we prefer to continue eating our own dog food.

We did come up with a solution, a way to implement the Russian Doll
cache on our site and expose it to you all too! The \_version field in an
ElasticSearch document provides a definitive version for a document.
Thus we are now passing that through our api in a new field called
"document_version" you'll find in your api calls to Activity Search v2.
All children of a given asset will now either be indexed with new guids
or their versions will be incremented when they are changed. Roll the
cache digests gem in, and you can now construct a functional Russian
Doll Caching scheme backed by ElasticSearch.

    class EventController < ApplicationController
      def show
        @event = ACTV.event params[:id]
      end
    end

    <% cache("event/#{@event.id}/@{event.document_version}") do %>
      <!-- Display Event Information -->
      <% @event.components.each do |component| %>
        <% cache("component/#{component.id}/#{component.document_version}") do %>
          <!-- Display Individual Component Information -->
        <% end %>
      <% end %>
    <% end %>

So now if the event itself is changed, it will bump the document version
without changing the children. So just the section displaying that event
information will have to be recompiled, while the others are drawn from
Memcached. If the components (in this case the subevents and their
pricing) are altered, that individual subevent will fall out due to
either it's guid or it's document version changing, and the parent
document will fall out due to us reindexing it (which bumps it's
document_version). Thus only that one subcomponent will be recompiled
and the general information will be recompiled while everything else is
drawn from Memcached. Additionally the inclusion of the cache digests
gem will ensure that your cache keys have the digests of the views
appended to the end of them.

Hopefully this will help you if you're using our Activity Search v2 API
or simply if you're using ElasticSearch as a datastore in your
application!

<!--
title: Instagram Post Scheduler: Part 1
description: Writing a custom solution to post Images to Instagram on a set schedule
slug: instagram-post-scheduler-part-1
image: https://library.wamphlett.net/photos/blog/photography/post-scheduler.jpg
published: 2023-11-26
-->
# Writing an Instagram Post Scheduler: Part 1
For those who know me, it's no secret that I've been pretty resistant to the world of social media. Despite my efforts to steer clear, more and more people have been asking where they can see my photos. After being asking again during my recent trip to Japan, I finally caved in and created an Instagram account.

## Instagram vs. My Editing Schedule
Photography is a hobby and usually something I only do when I am on holiday. When I get back, I don't tend to edit everything in one go. I usually will spend an afternoon every now and then looking back through my photos and selecting some to edit. After a few weeks of having Instagram, I started to feel pressured into editing photos for the sake of uploading something that week. That lead to me rushing edits and honestly, started to suck the enjoyment out of editing. I don't want my hobby to start feeling like a chore so I wanted to find a way to maintain regular uploads while not changing my editing style.

## Finding a way to schedule posts
When I have a productive editing session, I'm not keen on flooding Instagram with all my new images at once. Nor am I interested in making sporadic bulk posts every few weeks. I needed a solution that would allow me to edit at my own pace, but then gradually release those images on Instagram. Sure, I could manually post them on a weekly basis, but knowing myself, I'd likely lose track once distracted by something else. Automation was key. Though there are paid services available, I wasn't eager to add another subscription to my life. Plus, it's just like me to want to tackle the challenge myself.

## Playing with the Facebook graph API
First thing was first, I need to check if there was even an API I can use to automatically post images? The answer was yes, but they don't make it easy. You can use the [Content Publishing API](https://developers.facebook.com/docs/instagram-api/guides/content-publishing/)but there are a few catches. In order to make use of this API, you need an Instagram creator account (easy enough to set up), a Facebook account to create an app in the Meta developer dashboard (not my favourite part, as I'm not a fan of Facebook, but I reactivated an old account and locked it down), and a Facebook page linked to your Instagram account, again I just made a page and locked it all down. 

Once that was sorted, I could create a business app in the Meta developer dashboard. After obtaining the necessary access tokens, I used the create container endpoint to prepare a new post:

```shell
curl -i -X POST \
  "https://graph.facebook.com/v18.0/{instagram_id}/media
  ?image_url={public_image_url}
  &caption={post_caption}"
```

Then, with the container ID from the above step, I could use the publish container endpoint to actually post it to Instagram:

```shell
curl -i -X POST \
  "https://graph.facebook.com/v18.0/{instagram_id}/media_publish
  ?creation_id={container_id}"
```

So, there it is! If I can post using curl, I can automate it.
## Limitation with locations
Instagram lets you add a location to your posts and as part of the container creation, you can add location tags:

```shell
curl -i -X POST \
  "https://graph.facebook.com/v18.0/{instagram_id}/media
  ?image_url={public_image_url}
  &caption={post_caption}
  &location_id={location_id}"
```

Simple enough right? Well, I didn't find this out until much later but in order to get the location ID, you need to use the [page search API](https://developers.facebook.com/docs/pages-api/search-pages):

```shell
curl -i -X GET \
  "https://graph.facebook.com/pages/search?q=Facebook
  &fields=id,name,location,link
  &access_token={access-token}"
```

This is where things go down hill. In order to use this API, your app requires app review and business verification. 

> This permission or feature requires successful completion of the App Review process before your app can access live data.

> This permission or feature is only available with business verification. You may also need to sign additional contracts before your app can access data.

In order to get your app reviewed, it must be a publicly accessible app and meet a bunch of other requirements which I just don't want to do. This seems like a crazy requirement in my opinion, these IDs are already available if you just search for the location on Instagram. You can find a list of locations [here](https://www.instagram.com/explore/locations/). And if you go to any of these pages, you can see the ID right in the URL: https://www.instagram.com/explore/locations/549299662

So, I guess if I want to automatically tag my posts with a location, I am forced to manually go find the ID and save that - for now at least. 

## Building a scheduler
I built a simple server in Go with a Mongo DB database. It consists of a manager for managing entries, a publisher to interface with the Instagram API, and a scheduler for timed posts.

### Manager
The server holds a list of entries for Instagram publishing, managed by a [gin server](https://github.com/gin-gonic/gin). It handles CRUD operations for entries and schedules, and sets entry priority.

```go
type Entry struct {
	ID            primitive.ObjectID `bson:"_id" json:"id"`
	ImageURL      string             `bson:"image_url" json:"image_url"`
	LocationID    int                `bson:"location_id" json:"location_id"`
	Caption       string             `bson:"caption" json:"caption"`
	Priority      int                `bson:"priority" json:"priority"`
	PublishedTime *time.Time         `bson:"published_time" json:"published_time"`
	ShouldPublish bool               `bson:"should_publish" json:"should_publish"`
}
```

### Scheduler
The scheduler works based on a list of specific days and times for posting.

```go
type ScheduleEntry struct {
	ID        primitive.ObjectID `bson:"_id" json:"id"`
	DayOfWeek time.Weekday       `bson:"weekday" json:"weekday"`
	TimeOfDay time.Time          `bson:"time" json:"time"`
}

type Schedule struct {
	Entries []ScheduleEntry
}
```

Using the config, the scheduler calculates the next time a publish should happen

```go
func (s *Scheduler) calculateNextTime() time.Time {
	currentTime := time.Now()
	var minTime time.Time
	found := false

	for _, entry := range s.schedule.Entries {
		// we have to calculate the next two weeks of times
		for _, t := range []time.Time{currentTime, currentTime.Add(time.Hour * 24 * 7)} {
			nextTime := s.calculateNextTimeForEntry(entry, t)
			if nextTime.After(currentTime) && (!found || nextTime.Before(minTime)) {
				minTime = nextTime
				found = true
			}
		}
	}

	return minTime
}

func (s *Scheduler) calculateNextTimeForEntry(entry Entry, currentTime time.Time) time.Time {
	// first, find the next day of the week that matches the entry
	daysUntilNext := int(entry.DayOfWeek - currentTime.Weekday())
	if daysUntilNext < 0 {
		daysUntilNext += 7 // wrap around to the next week
	} else if daysUntilNext == 0 && currentTime.Hour() > entry.TimeOfDay.Hour() {
		// if it's the same day but the time has already passed, go to next week
		daysUntilNext = 7
	}

	return time.Date(
		currentTime.Year(),
		currentTime.Month(),
		currentTime.Day(),
		entry.TimeOfDay.Hour(),
		entry.TimeOfDay.Minute(),
		entry.TimeOfDay.Second(),
		entry.TimeOfDay.Nanosecond(),
		currentTime.Location(),
	).AddDate(0, 0, daysUntilNext)
}
```

Once the time is calculated, the scheduler triggers the publisher, which then interacts with the Facebook API. The cycle restarts after each publish or schedule change.
### Publisher
The publisher is responsible for getting the next post from the manager and calling the Facebook API using the endpoints I discussed above.

```go
func (p *Publisher) Publish(url, content string) error {
	log("publishing to Instagram...")
	log("image url: %s", url)
	log("caption: %s", content)

	// Create a container
	containerID, err := p.createContainer(url, content)
	if err != nil {
		return err
	}

	log("successfully created media container: %s", containerID)
	if !p.fullPublish {
		log("full publishing not enabled! not publishing the container")
		return nil
	}

	// Publish the container
	publishID, err := p.publishContainer(containerID)
	if err != nil {
		return err
	}
	log("successfully published media container %s. publish id: %s", containerID, publishID)
	return nil
}
```

```go
// Response represents the API response structure
type APIResponse struct {
	ID string `json:"id"`
}

func (p *Publisher) apiUrl() string {
	return fmt.Sprintf("https://graph.facebook.com/v18.0/%s", p.instagramPageID)
}

func (p *Publisher) createContainer(imageURL, caption string) (string, error) {
	return p.callFacebookAPI(
		fmt.Sprintf("%s/media?image_url=%s&caption=%s&access_token=%s",
			p.apiUrl(),
			url.QueryEscape(imageURL),
			url.QueryEscape(caption),
			p.longLivedPageToken))
}

func (p *Publisher) publishContainer(containerID string) (string, error) {
	return p.callFacebookAPI(
		fmt.Sprintf("%s/media_publish?creation_id=%s&access_token=%s",
			p.apiUrl(),
			containerID,
			p.longLivedPageToken))
}

func (p *Publisher) callFacebookAPI(url string) (string, error) {
  // code to curl the Facebook API
}
```
## What next?
So thats it, it works but lets face it, managing a schedule via an API is going to get tiresome quickly so we need some sort of UI...
<!--
title: This site
slug: my-site
published: true
-->
# My Website
I've been wanting to flesh out my site for a long time and I finally got round to it!

For a long time, my site was just a single page with a few words on it. At the time of making the site, I was doing a lot of [Flutter](https://flutter.dev/multi-platform/web) mobile work and the Flutter team were getting ready to launch their web implementation. Because I loved working with Flutter, I decided to try and make the site on their beta branch and as with any Flutter development, it was a breeze - seriously, if you haven't checked Flutter out, do it! It make front end work enjoyable again!

## Time to add a blog

The intention has always been to have a place where I can write about the stuff I work on, but its just one of them things which has always fallen to the bottom of the priority list. While [building my arcade](../ultracade/README.md), I really started to get frustrated that I didn't have somewhere to put all my updates so I set out to start making a blog.

I needed something really simple, anything complicated or tedious to manage would just end up being another dead project that gets neglected. Markdown couldn't be any less simple to manage and requires little to no management so I started designing a solution with Markdown as my base. 

The [blog repository](https://github.com/wamphlett/blog) couldn't be any simpler, its just a bunch of Markdown files grouped by a topic. Each topic has its own overview page and a then a collection of articles. It was important to me that if you went to the repository, you could navigate the blog happily without ever needing to visit the website. In my opinion, Github handles this really well, the topic overview pages are just the `README.md` files and then you can navigate to any article within the topic.

## Serving the markdown to a website
Now that the blog was sorted, I needed a way to get that Markdown and put it on my website so that I could style it a bit and "make it look prettier". I ended up writing [a small Go server](https://github.com/wamphlett/blog-server) which clones the blog repository, looks at all the files and indexes them. 

In order to index the files with some sort of logic, I needed to add metadata to each of the entries. As I said previously, I wanted the blog to be completely readable in the repository so I didnt want to litter the files with metadata or have a specific naming convention for all the files. I settled on using regular HTML comments at the top of each file. The server then uses this information when its indexing all the entries to know what page to serve, and when.

```html
<!--
title: My Website
slug: my-website
published: true
-->
```

Next, I needed a way to replace all of the relative links with the URI of intended page. Using a regex to find all the links, I can then work out the path to the linked file. From there I can use the index to get the URI of the page and bingo, links are now sorted.

```go
func (r *Reader) replaceRelativeLinks(s, path string) string {
	reg := regexp.MustCompile(`(\[[\w\d\s\-!?]*\]\()(\.[\/\.\w\d\-]*)\)`)
	for _, match := range reg.FindAllStringSubmatch(s, -1) {
		linkedFilePath := filepath.Clean(filepath.Join(filepath.Dir(path), match[2]))
		if p := r.index.GetURIForFile(linkedFilePath); p != "" {
			s = strings.ReplaceAll(s, match[0], fmt.Sprintf("%s%s)", match[1], p))
		}
	}
	return s
}
```

I had a similar issue with all the relative links to static content. I handled this by making sure all static content is in a single directory and then any links to that directory are replaced with the blog server URL and the server just serves that directory as a file server.

```go
// serve static files
s.router.PathPrefix(fmt.Sprintf("/%s/", assetDir))
  .Handler(neuter(http.FileServer(http.Dir(contentDir))))
```

Finally, I needed a way to turn my Markdown into HTML. I decided to do this on the server so that I have more control and I ended up using [Goldmark](https://github.com/yuin/goldmark) which completely does everything I need out of the box.

## Updating the website
So I now have a blog and an API to return the entries, I needed a site to display them. As much as I would have loved to use Flutter for this, its just not the best tool for a conventional website due to the fact everything is one large canvas! I have a fair amount of [React JS](https://reactjs.org/) experience so I decided to put together a real simple site using that instead.

I've grown quite fond of the [Flutter home page](https://warrenamphlett.co.uk/) and wasn't ready to say goodbye yet so I decided to keep it. My [Caddy server](https://caddyserver.com/v2) is just set up to pass the URIs to the relevant containers.

```
warrenamphlett.co.uk {
  handle / {
    reverse_proxy wamphlett:80
  }

  handle /main.dart.js {
    reverse_proxy wamphlett:80
  }

  handle /assets/* {
    reverse_proxy wamphlett:80
  }

  # fall back to serving everything from the blog
  handle {
    reverse_proxy wamphlett-blog:80
  }
}
```

And thats it... Hopefully it serves me well for the foreseeable future!
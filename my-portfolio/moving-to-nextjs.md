<!--
title: Moving to NextJS
description: Replacing my Flutter website with a new NextJS site
image: https://library.wamphlett.net/photos/blog/headers/nextjs.jpg
slug: moving-to-nextjs
published: 2023-11-11
-->
# Moving to NextJS
I should start this by saying that the intention was never to replace my Flutter site, only to improve it but sometimes things don't work out the way you expect them to. 

This all started after people kept asking me where I had been and if I had any nice photos, I didn't have any social media at the time so got the idea to add a timeline to my current website - just something simple with a few pictures. I set out to add the changes to my site and it didn't take long before I had it done.

<div class="images single">
  <img src="https://library.wamphlett.net/photos/blog/my-portfolio/flutter-timeline.jpg?w=1920" />
</div>

## Performance issue
Once the site was live, I started adding photos and the more I added, the more the site didn't like it. Scrolling became very stuttery and images took forever to load. Because Flutter makes one large canvas, the browser tries to load everything at once, this meant that photos outside the viewport were being prioritised and meaning sometime you were waiting a long time for the main image to load. My site doesn't need to be the most performant website on the web but that just wasn't acceptable and lets face it, its not exactly a great first impression for a so-called software engineer. 

I tried multiple ways to try and fix this but each solution failed. Paired with the stuttery scrolling which I had no hope of fixing, I came to the realisation that I was going to need to come up with a more web friendly solution.
## Hello NextJS
I've got some previous ReactJS experience and my blog was already written using it so I decided that would be the best thing to work with. I've never used a ReactJS framework before so I thought it would be a good chance to see what they can offer. I looked at a few and ultimately settled on [NextJS](https://nextjs.org/).

I still love the design of my website so I set out to make the exact same site using NextJS and it really didn't take long. NextJS's tooling makes creating components a breeze and I was able to rewrite all the Flutter components like for like. The only thing I had to change a little bit were the social icon animations but thats no big deal.

The performance is now great and the NextJS image has [priority images built right in](https://nextjs.org/docs/pages/api-reference/components/image#priority). As an added bonus, I now get to use proper web metadata again too. Unfortunately, there was no way to do this with Flutter for web at the time. 

```ts
export const metadata: Metadata = {
  metadataBase: new URL('https://warrenamphlett.co.uk'),
  title: {
    template: '%s | Warren Amphlett',
    default: 'Warren Amphlett',
  },
  description: 'Software engineer and Photographer',
  authors: { name: 'Warren Amphlett' },
  openGraph: {
    type: 'website',
    locale: 'en_GB',
    url: 'https://warrenamphlett.co.uk',
    images:
      'https://library.wamphlett.net/photos/website/2023/albania/three-of-a-kind.jpg',
    siteName: 'Warren Amphlett',
  },
};
```

## Managing the timeline
The timeline has to be easy to manage - anything thats cumbersome or time consuming means that I will never update the site. I didn't want a full CMS, thats very overkill so I decided to make an events state object which can have events registered to it. 

```ts
export enum EventType {
  Travel = 'travel',
}

export type EventData = {
  year: number;
  month: number;
  type: EventType;
  title: string;
  tagline?: string;
  intro?: string;
  small?: boolean;
  images: ImageGrid[];
};
```

Each event has an array of images that define which grid should be used to display them.

```ts
// All of the available grids
export enum Grids {
  Row = 'row',
  Double = 'double',
  DoubleInverted = 'doubleInverted',
  Offset = 'offset',
  OffsetInverted = 'offsetInverted',
  TriWide = 'triWide',
  TriWideInverted = 'triWideInverted',
  TriSquare = 'triSquare',
  TriSquareInverted = 'triSquareInverted',
}

// The image details
export type GalleryImage = {
  url: string;
  title?: string;
  description?: string;
};

// Grid data to define how the images should be displayed
export type GridData = {
  grid: Grids;
  aspectRatio?: number;
  skinny: boolean;
  images: GalleryImage[];
};
```

On app start up, I register all of the events with the state object and I can use these later to dynamically build the timeline.

```ts
...
register(Japan);
register(Albania);
...
```

I do something similar with the images for the timeline, on app start up, I register all of the images I intend to use on the site.

```ts
export default buildImages(libraryUrl('2016/mountfuji'), {
  aboveTheClouds: {
    title: 'Above the clouds',
    description:
      'watching a thunderstrom roll in from above is quite something',
    url: 'above-the-clouds.jpg',
  },
  dilierium: {
    title: 'Dilierium',
    description: 'no sleep and a lack of oxygen will do that to a man',
    url: 'dilierium.jpg',
  },
  ...
```

And thats it, when I want to add more to my timeline, I just register the images, create a new event in the [event directory](https://github.com/wamphlett/wamphlett/tree/nextjs/src/app/events) and decide which grids I want to use to display the images. 




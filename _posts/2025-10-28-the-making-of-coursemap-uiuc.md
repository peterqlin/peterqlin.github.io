---
layout: post
title:  "The Making Of CourseMap UIUC"
date:   2025-10-11
categories: projects
---

About a month ago, I spontaneously decided to walk over to Temple Hoyne Buell Hall—the architecture building. It wasn't my first time there—I've studied in the atrium several times in the past—but that day I decided to check out what was going on in the lecture hall. I ended up walking into an ancient Greek architecture class, which was kinda cool.

After that little excursion, an idea popped into my head.

Basically, I'd been wanting to make a database project, since I'd never done the whole create and populate tables from scratch thing before. My idea was to scrape all the courses currently being offered during the Fall 2025 semester, chuck the data into a relational database, and create a real-time map visualization of what classes were going on around you—a "course map" of sorts.

The first step was figuring out where to get the data from; I've seen a bunch of "UIUC course this or that" projects before, so I was sure there was a way to get that data. I ended up stumbling across this [github repo](https://github.com/shanktt/UIUC-Course-Explorer-Scraper/) that had basically all I needed to get started.

Since I wanted to visualize in real-time which classes were happening, I would need to know more than just what classes were being offered—I needed all the sections of those classes in order to figure out their start and end times. Turns out that the scraper from the repo used the `/schedule` and `/catalog` directories interchangeably when requesting data from the course explorer, which I didn't notice at first. Only when I saw a slight inconsistency in the URL when switching between tabs did I realize the difference. I needed to scrape data from `/schedule` because that's where all the course section information was.

Once the scraping URL was figured out, I got to automating it. The first thing to tackle was the course explorer "directory structure". I use quotes because the main "directory" is really just an XML file that contains links to other XML files, but I'll just treat it like a folder. Here's how it's organized:
{% highlight markdown %}
schedule/2025/fall
├─ AAS
├─ ...
├─ CS
│  ├─ 100 (CS Orientation)
│  │  └─ AL1
│  ├─ 101 (Intro Computing)
│  │  ├─ AL1
│  │  ├─ ...
│  │  └─ AYS
│  ├─ ...
│  └─ 598 (Special Topics)
│     ├─ AO2
│     ├─ ...
│     └─ YLL
├─ ...
└─ YDSH
{% endhighlight %}

To access every section for every class, we need to do a tree walk through all majors (AAS-YDSH), collecting the sections (AL1, AL2, etc) for each of that major's classes (100, 200, etc).

I saved a local copy of every XML file that I scraped so I wouldn't have to repeat the process if something went wrong during scraping, effectively checkpointing my scraping progress based on saved files. This ended up coming in handy when one of my requests was denied, causing my script to skip a class or two. I was able to instantly restart scraping from the skipped classes thanks to the checkpoint.

To prevent this post from dragging out too long, I'll rapidfire the last couple points I wanted to make.
- Database table structure: I decided on three tables: classes, sections, and buildings. Each section references an external class ID and an external building ID.
- Data cleaning: While populating the database, I ran into format errors due to missing section end times, room numbers, and other attributes. For simplicity, I filtered out everything that did not fit my schema. I also filtered out online classes for obvious reasons.
- Visualizing it: My frontend relied on leaflet.js to render course locations, which required each building's longitude and latitude data. The data I scraped from the course explorer did not contain coordinate information, so I used Geoapify's Address Autocomplete API to figure out the exact locations for each building I scraped. Again, for simplicity, I deleted any sections with buildings that the API could not locate.

After adding some UI components and making the app somewhat presentable, I briefly hosted it with Vercel, Supabase, and Fly.io, and was able to use CourseMap out in the world. Here's a screen recording I took outside Grainger library.

![coursemap demo](../../../../assets/coursemap_demo.gif)

You can find the source code [here](https://github.com/peterqlin/coursemap).


---
layout: post
title:  "Past Library Experience"
date:   2023-10-23 14:19:17 -0700
categories: jekyll update
---

*An outline of work I've done for different University of California Libraries*

## UC Santa Barbara Library, DreamLab

I worked for this research department. My primary duties included desk support, assisting with workshops, and working on code projects. 

### Data Carpentry

I assisted with multiple data carpentry workshops and made contributions to one. Example of contributions [twarc workshop](https://ucsbcarpentry.github.io/2022-05-26-ucsb-twarc/): 
In particular, I made edits to [episode 3](https://github.com/gnunnelley/gh-pages-copy/blob/gh-pages/_episodes/03-3-tweet-anatomy.md). I wrote about the anatomy of JSON files and how the twitter API organized their data. 

### Airphotos project 

The [Airphotos repository](https://github.com/jonjab/AirPhotos) contains notebooks designed to geolocate aerial photography imagery. The imagery data sets are sourced from UC Santa Barbara’s library. I ran, tested, and added functions to the Airphotos repository. I worked with the OpenCV library to automatically crop images and COLMAP (an open source software for reconstruction of image collection). The Topography Comparison uses point data, from imagery sets processed by COLMAP, and compares it against USGS topographical data to find the best match. 

[AirPhotos]({{ site.url }}_pdfs/AirPhotos.pdf)

## UC Merced Library, GIS Center

 I worked remotely for this research center. My primary duties involved cleaning data and creating dashboards. I also worked with Esri softwares for mapping and other purposes. I complied notebook of Python, R, and SQL code that I wrote during my time in this position.

[DataClean]({{ site.url }}_pdfs/datacleanUCM.pdf)

<!---
trouble shooting issues with embedding pdfs 
either a CSS problem or url/pathway problem 
jekyll compiles/renders the .md post relative to the post directory when {{ site.url }} not specified in the path 
embedded pdf files do not load otherwise 

<"/_pdfs/datacleanUCM.pdf" type="application/pdf" width="100%" height="500">
<iframe src="{{ site.url }} _pdfs/datacleanUCM.pdf" type="application/pdf" width="100%" height="500">

https://stackoverflow.com/questions/46105960/inserting-my-pdf-in-my-blogs
xhttps://discourse.gohugo.io/t/embed-pdf-file-into-a-page-or-post-papermod-theme/36440/6
PDF.js workaround? 

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
-->
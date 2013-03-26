---
layout: post
title: "Cache Busting Clientside"
date: 2013-03-26 14:25
comments: true
categories: []
author: Adam Willoughby-Knox
---

One of the problems we had to overcome at [Julep](http://www.julep.com) was **cache-busting**. This refers to when a client's web browser caches a copy of `styles.css`, and doesn't know we've made changes to that file and that it needs to download a new version.

<!-- more -->
This issue might seem minor at first, a few misaligned blocks, a block of text not styled quite right. The ~~easy~~ wrong answer: *Just tell them to refresh the browser, Shift + Refresh!*

But that isn't very nice, and when a user gets a page that is destroyed by out of date javascript and are posting on facebook on your behalf that you need to *Shift + Refresh*, well that is something you should be finding a solution to.

[Yslow](http://developer.yahoo.com/blogs/ydn/posts/2007/05/high_performanc_2/) and [Google Insights](https://developers.google.com/speed/docs/best-practices/caching#LeverageBrowserCaching) confirm that we should be using *far future expirations* on these files so that they are aggressively cached.
The solution they propose to this cache-busting issue is to use the [ETag](http://en.wikipedia.org/wiki/HTTP_ETag) property in the headers. It sounds like the perfect solution, but yahoo warns that they are [strongly tied to the origin server](http://developer.yahoo.com/performance/rules.html#etags). The second solution is the [Last-Modified header](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html) which relies on a timestamp handshake between server and client. 

We opted for a solution that provides granularity and control, we modify the file name based on the SHA stamp of the last git commit on that file. Every commit has a SHA stamp, and we can guarantee that user's are forced new assets when we release code.

## Here is how we solved the issue:
1. First we keep a running list of the files that need to be managed
1. Then we append the first 8 characters of the SHA stamp for the last git commit to touch that file
1. In magento we also have to update xml files where these assets are initially requested. This part is still a manual step.

{% codeblock SHA-Stamping Files lang:bash %}
#!/bin/bash
# Create sha stamped assets from this list of files

FILES="
../skin/frontend/julep/default/css/styles.css
../skin/frontend/julep/default/css/plugins.css
../skin/frontend/julep/default/js/julep.js
../skin/frontend/julep/default/js/plugins.min.js
../js/general/campaign_configuration.js
../js/general/julep.js
../js/general/quiz_handler.js
../skin/frontend/julep/default/css/quiz.css
"

for f in $FILES
do
	sha=$(git log --pretty=oneline -1 $f | cut -c1-8)
	
	extension="${f##*.}"
	filepath="${f%.*}"

	cp ${f} ${filepath}_${sha}.${extension}
	
	echo "Processed: ${f}";
done
{% endcodeblock %}

Note: One weird side-effect is for files like `plugins.min.js` get renamed `plugins.min_12345678.js` if that bugs you, be sure to update the bash code above. 
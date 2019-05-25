---
layout: post
title: "Exploring An Application's Past & Future"
date: 2019-05-06
categories: blog
---

Years ago, when I started building web applications I would simplistically imagine them as a 2D graph of assets. It didn't take long before I started running into race conditions, time zone discrepancies and caching issues, which made me become aware of the crucial dimension of time in which an application evolves. As I started becoming familiarized with security issues, I realized that vulnerabilities often occur at the boundaries of an application's assets, where references to past/future assets (subdomains, links, libraries, 3rd party services etc) are often left dangling, outdated or become actual vulnerabilities themselves. What follows are some findings I stumbled upon while keeping this perspective in mind.

<!--more-->

Digging up the past
-------------------

I found that [The Wayback Machine](http://web.archive.org/) from Archive.org is a great tool for quickly surveying the public assets of a domain, allowing anyone to browse historical snapshots.

![](https://lh3.googleusercontent.com/CPFx99xbknwUIrSKl_o0Hely6cqddzlLLk2dwQO4NCtrsBfZ4eLx31o6f7AxQMsVRmkLJL5M0gvOfsJOta1QbYrd0-9S1n0fr7IVMLOQd-B-tphmPkf8kLXQjIxo6I76aT11dl3I "The Wayback Machine sitemap viewer")

The viewer groups the site by years and shows all assets in radial-tree graph

The center circle is the domain, while the successive rings dig recursively into the directory structure. Naturally, not only the content is archived, but also things like JavaScript code, parameters and comments. On the visual sitemap explorer the juicy stuff can be found on the outer-most rings, which reveal actual files and a range of parameters.
The image above reveals a `?pagename=` parameter, which already hints to [local file inclusion vulnerability](https://labs.detectify.com/2012/11/16/local-file-inclusions-in-perlcgi/).

[Waybackurls](https://github.com/tomnomnom/waybackurls) is a tool for quickly fetching URLs from the Wayback Machine, which can give a feel of how the application is built.

![](/assets/posts/1/waybackurls.png)

Although mostly used for fuzzing, I like [wfuzz](https://wfuzz.readthedocs.io/en/latest/) because of it's nicely structured output. This scan quickly reveals which URLs are still available, 404s and redirects, already hinting at some potential attacks

![](/assets/posts/1/wfuzzed.png)

For more a more in-depth perspective, [wayback_machine_downloader](https://github.com/hartator/wayback-machine-downloader) can be used to fetch a copy of all the historical snapshots of a website.

An interesting find: emails & reset password links.

![](https://lh6.googleusercontent.com/JgD5dEoWoFO4hd9EmJ7nl4_XZ-3tMChuFSlp0AbrlR3dStlxgMEL6g80U0Znu_D2glxWCOx36K0-m56jep6e-Uw2l4BJbDZ3_GOsOJdLwby67BZJvU8xTYrfsr8ZJskCXydVqhJL)

...and activation links.

![](https://lh6.googleusercontent.com/vHHSTSHS9tgasYO4zCC0JnB5yfy4lu50gbtCwi6hFpJtvMzl6sOiQOCxjEVD2pHAh60CyNWZh3qPWmV8KYzfi6duelWkgD6WS_DMCIyc-8Rq7y6v2txixBogjD0gBiOs6BIuUCoR)

[Password reset](https://hackerone.com/reports/43807) and activation links get indexed by crawlers or get [leaked to 3rd parties](https://hackerone.com/reports/303322) via the Referrer header and, to make matters worse, expiration policies vary from framework to framework or has [bugs](https://hackerone.com/reports/118948), so they can occasionally be reused. Since the image above also reveals user emails, this paves the way for a [targeted phishing campaign](https://blog.detectify.com/2016/10/20/how-to-identify-a-phishing-email/).

The Wayback Machine also stores error pages and redirects; 40* and 50* pages can provide insight into the configuration of the application's stack while 30* redirects, conveniently highlighted, open the way to potential [subdomain takeovers](https://labs.detectify.com/2014/10/21/hostile-subdomain-takeover-using-herokugithubdesk-more/).

![](https://lh3.googleusercontent.com/jypQv0LcPCfkZq7VJRlzYE1RyBL7ATlJ8EU3LsCL8WJbBy02sRCBVZRosMsMPixA8De8pbfV6NfW1mexDCNoeEBlj3VNakTTf9tqByJKGUCeSmIY3LvCw0q28Mt85aJx2sY6V591 "Current error page vs. Error page from Wayback Machine")

Comparing different versions of error pages can reveal when an application's infrastructure has changed.

Modern web frameworks [concatenate, minify and fingerprint assets](https://guides.rubyonrails.org/asset_pipeline.html#what-is-fingerprinting-and-why-should-i-care-questionmark) for performance and caching purposes. This can sometimes make it hard to spot changes made to client-side code. Since it's easy to get a list of all previously generated assets from The Wayback Machine, it becomes easy to spot different versions of the same file based on their file size.

![](https://lh5.googleusercontent.com/voD4mMrSi3GPOat_nT0oiwc0YsGg5vB-wlEY6rp5hbM4HuLOVox3MMmHopLnB_0aiCfEGUcn9A5UzBKJaxxe5fhdEI1M7LfUNh1dchfyzApPUAIAyIwLZtFHjbtPzOVn05J37qLL "Listing all historical JS assets, sorted by size")

Even though their names are obscure, similar sizes point to the same files.

Learning from others mistakes
-----------------------------

"The future is already here --- it's just not very evenly distributed" claims William Gibson, the sci-fi author who also coined the term "cyberspace". While the quote can be dismissed as abstract, the reality in software development is that incomplete features often get released to production - both intentional and unintentionally. Some of these features quickly get reverted. But these reversals are not always atomic operations. Artefacts get left behind, i.e. [comments](https://cwe.mitre.org/data/definitions/615.html), which can be a treasure-trove of development credentials, sensitive links, library versions, [S3 buckets](https://labs.detectify.com/2018/08/02/bypassing-exploiting-bucket-upload-policies-signed-urls/).

In order to safely deploy features according to a marketing timeline feature flags are used, which provide the opportunity to test the features incrementally to selected audiences, in case anything breaks. Feature flags is why many unreleased features from popular products become [leaked](https://www.bbc.com/news/technology-47630849) to the public long before their official launch. Giants like Facebook & Google [implement their own feature flag service](https://tech.co/news/the-dark-launch-how-googlefacebook-release-new-features-2016-04), while other websites used [3rd party services](https://rollout.io/blog/started-quickly-javascript-feature-flags/). Building on what we have explored until now, we can get insight into how a feature flag implementation may evolve.

![](https://lh5.googleusercontent.com/un6V3QsKKQ4QPxSG5mS_klUZW-lwHIYwtUVDFePqq5ceRuyj6W2QojQ8rMBnVsJNgBchijU3HTOIG0QFdpU_dPBnDCiKiYee52ZbBWWZFb4Spwg_xULy_s9GYSo4FY6v242eNgfl "Diffing 2 versions of a JavaScript file reveals how the feature flag implementation has changed")

The initial implementation is importing a local config file, while the newer implementation if fetching the configuration with an AJAX call to an external service.

The future becomes the past
---------------------------

In the context of web application security, an assumption I sometimes find myself making is that once a security issue is fixed, things tend to move forward and that class of bugs will not surface again. This, ofcourse, is false. From my previous work in web development, conscious and consistent effort has to be put into setting up proper testing and maintaining code quality. A high code coverage percentage is  the pride and joy of any seasoned developer, but that doesn't prevent security bugs from sipping through.

A recent [XSS was made possible in Google Search](https://www.youtube.com/watch?v=lG7U3fuNw3A) because the sanitizer was modified -- essentially becoming more lenient -- because of interface design issues.

![](https://lh4.googleusercontent.com/srpcLSJaEAUHE3HuZ1KgDbO-7CKSVVv4_XVYfZyAc0Mux7cMbnvjWnjs6jKwTW8HpztKpZebbvF_PtcWv77hwPlR3RykebcscEZUhAMlmDF8pVqM6FZpfdLuS0zR23SR74Ie5jO5 "XSS trigger in Google Search")

Since [Google's frontend is open source](https://github.com/google/closure-library), we can take a look at what exactly happened.

![](https://lh3.googleusercontent.com/jnlLdLGjaMOFC00AvMgVhNyLI9j4oW5V0AYi8688rlnyCyjM-onG03wC2N6bZN2aj9JhNuzKEVvi9keFSUSDNG_0ZC5rCIgAkSvfwG88UCQffeqJmWDzFsmI8Ho9trpQNrWUI6X2 "Commit which introduced the vulnerability")

The commit message for the fix notes that tests were added and updated. While automated tests vouch for a programmer's correctness at the level at which the test was introduced (unit, integration, system) and guard against violations of the application's architecture and business logic, they do not always vouch for the overall security of the application.

![](https://lh4.googleusercontent.com/leX0ncR409vk4gvsSiASDXpD-crIkNg0_wWVtKhyMkc0vnYhHseY0B-vwuQIYs0hcwzpgm-_0UZD4-wA2q_YP1YHf14LKPmy_z7i3Xn68hWY1E6PswE-e5muOzyFbxA9Fsok9_DL "Commit which fixes the vulnerability")

We can see that the fix is actually a rollback, essentially bringing the code to a previous point in time.

To sum it up, application code moves back and forth, inherently producing side effects which can be exploited. Hacking is about finding the right time to act... and the right time to stop.

---
layout: post
title: Defer non-critical CSS
authors:
  - demianrenzulli
description: |
  Learn how to defer non-critical CSS with the goal of optimizing the Critical
  Rendering Path, and improve FCP (First Contentful Paint).
date: 2019-02-17
---

CSS files are [render-blocking resources](https://developers.google.com/web/tools/lighthouse/audits/blocking-resources): they must be loaded and processed before the browser renders the page. Web pages that contain unnecessarily large styles take longer to render, increasing the likelihood that users bounce: 53% of mobile visits bounce if the page does not load [in 3 seconds or less](https://www.thinkwithgoogle.com/intl/en-154/insights-inspiration/research-data/need-mobile-speed-how-mobile-latency-impacts-publisher-revenue/).

In this guide, you’ll learn how to defer non-critical CSS with the goal of optimizing the [Critical Rendering Path](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/), and improving [FCP (First Contentful Paint)](https://developers.google.com/web/tools/lighthouse/audits/first-contentful-paint).

## Loading CSS in a suboptimal way

The following example contains an accordion with three hidden paragraphs of text, each of which is styled with a different class:

<div class="glitch-embed-wrap" style="height: 420px; width: 100%;">
  <iframe
    allow="geolocation; microphone; camera; midi; encrypted-media"
    src="https://glitch.com/embed/#!/embed/defer-css-unoptimized?path=index.html&previewSize=100&attributionHidden=true"
    alt="defer-css-unoptimized on Glitch"
    style="height: 100%; width: 100%; border: 0;">
  </iframe>
</div>

This page requests a CSS file with eight classes, but not all of them are
necessary to render the "visible" content.

The goal of this guide is to optimize this page, so only the **critical** styles
are loaded synchronously, while the rest (like the ones applied to paragraphs),
are loaded in a non-blocking way.

## Measure

Run [Lighthouse](https://web.dev/discover-performance-opportunities-with-lighthouse/#run-lighthouse-from-chrome-devtools) on [the page](https://defer-css-unoptimized.glitch.me/) and go to the **Performance** section.

The report shows the **First Contentful Paint** metric with a value of "1s", and
the opportunity **Eliminate render-blocking resources**, pointing to the
**style.css** file:

<figure class="w-figure">
  <img src="./lighthouse-unoptimized.png"
       alt="Lighthouse report for unoptimized page, showing FCP of '1s' and 'Eliminate blocking resources' under 'Opportunities'" class="screenshot">
</figure>

{% Aside %}
The CSS we are using for this demo site is quite small. If you were requesting
larger CSS files (which are not uncommon in production scenarios), and if
Lighthouse detects that a page has, at least, 2048 bytes of CSS rules that
weren't used while rendering the **above the fold** content, you'll
also receive a suggestion called **Remove Unused CSS**.
{% endAside %}

To visualize how this CSS blocks rendering:

1. Open [the page](https://defer-css-unoptimized.glitch.me/) in Chrome.
1. Press Control+Shift+J or Cmd+Option+J (Mac), to open DevTools.
1. Go to the **Performance** tab and click on the **Reload** button.

In the resulting trace, you’ll see that the **FCP** marker is placed immediately
after the CSS finishes loading:

<figure>
  <img src="./cdt-perf-unoptimized.png"
       alt="DevTools performance trace for unoptimized page, showing FCP starting after CSS loads." class="w-screenshot">
</figure>

This means that the browser needs to wait for all CSS to load and get processed
before painting a single pixel on the screen.

## Optimize

To optimize this page, you need to know which classes are considered “critical”.
You'll use the the [Coverage Tool](https://developers.google.com/web/updates/2017/04/devtools-release-notes#coverage) for that:

1. In DevTools, open the [Command Menu](https://developers.google.com/web/tools/chrome-devtools/ui#command-menu), by pressing `Control+Shift+P` or `Command+Shift+P` (Mac).
1. Type "Coverage" and select **Show Coverage**.
1. Click the **Reload** button, to reload the page and start capturing the
   coverage.

<figure class="w-figure">
  <img class="w-screenshot"
       src="./coverage-unoptimized.png"
       alt="Coverage for CSS file, showing 55.9% unused bytes.">
</figure>


Double-click on the report, to see classes marked in two colors:

* Green (**critical**): These are the classes the browser needs to render the
  visible content (like the title, subtitle, and accordion buttons).
* Red (**non-critical**): These styles apply to content that's not immediately
  visible (like the paragraphs inside the accordions).

With this information, optimize your CSS so that the browser starts processing
critical styles immediately after page loads, while deferring non-critical CSS
for later:

* Extract the class definitions marked with green in the report obtained from
  the coverage tool, and put those classes inside a `<style>` block at the head
  of the page:

```html
<style type="text/css">
.accordion-btn {background-color: #ADD8E6;color: #444;cursor: pointer;padding: 18px;width: 100%;border: none;text-align: left;outline: none;font-size: 15px;transition: 0.4s;}.container {padding: 0 18px;display: none;background-color: white;overflow: hidden;}h1 {word-spacing: 5px;color: blue;font-weight: bold;text-align: center;}
</style>
```

* Then, load the rest of the classes asynchronously, by applying the following pattern:

```html
<link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="styles.css"></noscript>
```

This is not the standard way of loading CSS. Here's how it works:

* `link rel="preload" as="style"` requests the stylesheet asynchronously. You can learn more about `preload` in the [Preload critical assets guide](https://web.dev/preload-critical-assets).
* The `onload` attribute in the `link` allows the CSS to be processed when it finishes loading.
* "nulling" the `onload` handler once it is used helps some browsers avoid re-calling the handler upon switching the rel attribute.
* The reference to the stylesheet inside of a `noscript` element works as a fallback for browsers that don’t execute JavaScript.

The [resulting page](https://defer-css-optimized.glitch.me/) looks exactly like the previous version, even when most styles load asynchronously. Here's how the inlined styles and asynchronous request to the CSS file look like in the HTML file:

<!-- Copy and Paste Me -->
<div class="glitch-embed-wrap" style="height: 420px; width: 100%;">
  <iframe
    allow="geolocation; microphone; camera; midi; encrypted-media"
    src="https://glitch.com/embed/#!/embed/defer-css-optimized?path=index.html&previewSize=0&attributionHidden=true"
    alt="defer-css-optimized on Glitch"
    style="height: 100%; width: 100%; border: 0;">
  </iframe>
</div>

## Monitor

Use DevTools to run another **Performance** trace on the [optimized
page](https://defer-css-optimized.glitch.me/).

The **FCP** marker appears before the page requests the CSS, which means the
browser doesn’t need to wait for the CSS to load before rendering the page:

<figure class="w-figure">
  <img src="./cdt-perf-optimized.png"
       alt="DevTools performance trace for unoptimized page, showing FCP starting before CSS loads." class="w-screenshot">
</figure>

As a final step, run Lighthouse on the optimized page.

In the report you’ll see that the FCP page has been reduced by **0.2s** (a 20%
improvement!):

<figure class="w-figure">
  <img src="./lighthouse-metrics-optimized.png"
       alt="Lighthouse report, showing an FCP value of '0.8s'."
       class="w-screenshot">
</figure>

The **Eliminate render-blocking resources** suggestion is no longer under
**Opportunities**, and now belongs to the **Passed Audits** section:

<figure class="w-figure">
  <img src="./lighthouse-opportunities-optimized.png"
       alt="Lighthouse report, showing 'Eliminate blocing resources' on the 'Passed Audits' section." class="w-screenshot">
</figure>

## Next steps & references

In this guide, you used vanilla code to implement this optimization. In a real
production scenario, it’s a good practice to use functions like
[loadCSS](https://github.com/filamentgroup/loadCSS/blob/master/README.md), that
can encapsulate this behavior and work well across browsers. As a complement to
this, the [extract critical CSS guide](/extract-critical-css/) 
covers some of the most popular tools to extract critical CSS and includes 
[a codelab](/codelab-extract-and-inline-critical-css/) to see how 
they work in practice.

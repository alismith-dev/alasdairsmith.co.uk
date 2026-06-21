---
title: Becoming grid aware
draft: true
---

As mentioned in my [post about redesigning this site](/journal/site-redesign-spring-2026), one of the major goals in rebuilding was making it grid aware.

There's an overview of the grid aware functionality on [the grid aware page](/grid-aware), but in this post I want to go into more detail about how it works and how I built it.

## What does grid aware mean?

I've been enjoying learning about green software over the last few months. I feel passionate about reducing the impact of my work on the environment, but I've also found there are many interesting lessons on system design.

A grid aware system is based on the principles of green software. Since almost all software runs on grid electricity, we can use carbon intensity data from the grid to infer the emissions of the software. Then various strategies can be applied to try reduce emissions. To be grid aware, the system adapts its behaviour based on how much carbon is being emitted from the grid when run.

Grid aware _websites_ adapt functionality of the frontend based on conditions of the user's electricity grid. The focus is on the frontend as there are well established principles around backend/server based grid aware software, I'd recommend the [Green Software Foundation's course](https://learn.greensoftware.foundation/carbon-awareness) if you want to learn more.

The [Branch magazine](https://branch.climateaction.tech/) from the Green Web Foundation is prior art, see the [full write up](https://branch.climateaction.tech/issues/issue-9/designing-a-grid-aware-branch/) for more. The GWF are also responsible for the [Grid-aware websites project](https://www.thegreenwebfoundation.org/tools/grid-aware-websites/), which directly inspired me to rebuild my site. I believe I first heard about this through the [core maintainer's blog](https://fershad.com/writing/introducing-our-grid-aware-websites-project/), whose website is also grid aware (and is well worth a read!)

## Searching for a carbon emissions API

Determining the conditions of the grid is not an easy challenge, they are complex systems that are continually changing. To get data on grid carbon emissions, I needed an API that can estimate emissions on a reasonably real-time basis.

The grid-aware-websites project was built with the assumption that the Electricity Maps API could be used as a data provider, which - spoiler alert - is probably a good idea, as they're very well respected for real-time emissions data.

Unfortunately, the project initially planned to use an Electricity Maps API endpoint that is very expensive for an individual and is limited to data from a single country, as [discussed in a GitHub ticket](https://github.com/thegreenwebfoundation/grid-aware-websites/issues/21).

Because of this, I decided to look for alternative providers. I settled on [WattTime](https://watttime.org/) mostly because there's not many around! I'd heard of them some time ago thanks to their excellent explainers like [marginal](https://watttime.org/data-science/data-signals/marginal-co2/) and [average](https://watttime.org/data-science/data-signals/average-co2/) emissions.

I built a full implementation using the WattTime API and had it working, serving pages from Astro. However, I ran into some significant issues with performance. WattTime's API has a relatively complex surface area, requiring three separate calls: one for authentication, one to identify the electricity grid's region from the user's latitude/longitude, and one to determine carbon emissions for that region. Each of these calls adds latency and ultimately added up to a significant wait for page load.

Adding caching definitely helped but since the request pattern is likely to be spread geographically, most users would need at least two requests. At a few hundred milliseconds each, the latency wasn't workable, so I abandoned this approach.

Luckily, the folks at the GWF had been working with Electricity Maps to create a new API for this purpose! Their [blog post goes into all the details](https://www.thegreenwebfoundation.org/news/a-new-api-for-grid-aware-websites-and-beyond/) but the new API immediately seemed like the solution to unblock this project for me. Only one request is needed to retrieve carbon emissions data, bringing latency down to an acceptable level (barring caveats discussed below).

## Architecture

At a high level, the grid aware mode is structured around an Astro middleware that queries the Electricity Maps API before injecting the response into the rendering layer.

### Middleware

Astro middlewares process each individual request to the site, allowing shared functionality across routes. Middlewares are therefore required to be run in Astro's server-rendering mode, enabled across the site by setting `output: 'server'` in the config.

Each incoming request can come from a different grid running under different conditions, so behaviour needs to be modified per-request. This makes middleware the ideal choice, intercepting requests and injecting emissions data into responses.

First, the middleware checks if the grid aware mode has been disabled. This can either happen if a `grid-aware-disabled` query parameter is set - allowing users to disable grid aware if wanted; or if an environment variable is set - allowing me to disable it during development.

Next, to keep performance reasonable for users moving between pages on the site, the middleware checks for a cookie containing cached carbon intensity data. Since the Electricity Maps API returns hourly data, the cookie has a TTL of 1 hour.

If there's a cache miss, then the middleware combines with the [Netlify adapter](https://docs.astro.build/en/guides/integrations-guide/netlify/), where I'm currently hosting this site, to receive the request's geolocation as latitude/longitude coordinates:

```ts
export const onRequest = defineMiddleware(async (context, next) => {
  ...

  const { longitude, latitude } = context.locals.netlify.context.geo

  ...
}
```

These will be passed to the API request as we'll see shortly.

The middleware is designed to be quite defensive, and will disable the grid aware mode if errors are encountered. I want the site to be resilient to problems in the grid aware system, as ultimately it is a nice-to-have.

### Electricity Maps integration

Once we have latitude/longitude of the request, we can fire off a request to the Electricity Maps API. As above, this is a significantly simpler implementation as the [Carbon Intensity Level endpoint](https://app.electricitymaps.com/developer-hub/api/reference#carbon-intensity-level-latest) accepts both a API token and latitude/longitude, so we can receive all of the data in a single request:

```ts
const carbonIntensityLevelQueryParams = new URLSearchParams({
  lat: latitude.toString(),
  lon: longitude.toString()
})

const res = await fetch(
  `https://api.electricitymaps.com/v4/carbon-intensity-level/latest?${carbonIntensityLevelQueryParams}`,
  {
    headers: {
      'auth-token': `${apiKey}`
    }
  }
)
```

From this, we get back the `zone` - effectively the grid powering the location - and a `high`/`moderate`/`low` `level`:

```json
{
  "zone": "GB",
  "data": [
    {
      "level": "low",
      "datetime": "2026-06-13T12:00:00.000Z"
    }
  ]
}
```

The level thresholds defined in the [API docs](https://app.electricitymaps.com/developer-hub/api/reference#carbon-intensity-level-latest) and are based on difference from the average emissions for that grid over the last 10 days.

### Injection into the rendering layer

After receiving the level data, it is set on Astro [`locals`](https://docs.astro.build/en/reference/api-reference/#locals), injecting it into the rendering layer. We'll see later that the frontend can now use it to modify behaviour:

```ts
context.locals.gridAwareCarbonIntensityLevel = gridAwareCarbonIntensityLevel
```

And finally, the data is set on the cookie, caching it for 1 hour:

```ts
context.cookies.set(
  GRID_AWARE_COOKIE_NAME,
  { gridAwareCarbonIntensityLevel },
  { maxAge: GRID_AWARE_COOKIE_MAX_AGE }
)
```

### Differences with the grid-aware website project

Although I've been directly inspired by Green Web Foundation's [grid-aware-websites project](https://github.com/thegreenwebfoundation/grid-aware-websites), there are some architectural differences.

The key difference is that the grid-aware-websites project is explicitly targeted at pre-existing websites. A stated goal was avoiding the need to rewrite a whole site to add grid aware support.

My goal was to learn from the grid aware pattern, so rewriting from scratch made sense. Because of this choice, my architecture optimises for control via the primary server over edge workers. In the long term, if grid aware becomes the norm for web architectures, control from the server is more direct and easy to reason about.

However, this has a major downside: because the primary server must resolve the carbon intensity data before rendering, site latency becomes dependent on the speed of the API response. The latency is more than I would like but acceptable. I have some ideas that I'll cover later that may help with this.

## Usage in the frontend

Once `locals.gridAwareCarbonIntensityLevel` has been set, the frontend can use it to make decisions about how to change behaviour.

In all cases, I chose to only change behaviour when intensity level is considered high. Based on my testing in the UK, it seems like high intensity only really kicks in during the evening. Given that my goal is for the site to behave as normal for most of the time, setting this as the bar seems reasonable.

### CSS

The intensity level is made available to CSS by registering a custom CSS property and setting as the initial value:

```ts
CSS.registerProperty({
  name: '--grid-aware-carbon-intensity-level',
  syntax: '*',
  inherits: false,
  initialValue: gridAwareCarbonIntensityLevel ?? ''
})
```

This CSS property is then used to disable animations and view transitions using a style query:

```css
@container style(--grid-aware-carbon-intensity-level: high) {
  *,
  *::before,
  *::after {
    transition: none !important;
    animation: none !important;
  }

  @view-transition {
    navigation: none;
  }
}
```

This was inspired by the [Obs.js demo site](https://csswizardry.com/Obs.js/demo/) which does the same when detecting that the user is low on battery. The [Web Sustainability Guidelines](https://sustainablewebdesign.org/guidelines/2-12-ensure-animation-is-proportionate-and-easy-to-control/) also recommend avoiding unnecessary animations.

Separately, the "scramble in" animation in the header, built using Preact/JavaScript, also reads the intensity level and disables itself if intensity is high.

### Images

Images are the [biggest assets on the web](https://almanac.httparchive.org/en/2022/page-weight#fig-8) so avoiding loading, decoding and rendering them was one of the major goals for me.

Astro already provides a number of useful tools via the built-in [`<Image>`](https://docs.astro.build/en/guides/images/#image-) component, including automatic lazy loading. I wanted to extend this component's functionality to make it grid aware. My `<GridAwareImage>` component reads the intensity level from locals, then uses this to decide whether to render an alt-text placeholder matching the size of the true image, or an Astro `<Image>`:

```astro
{
  showPlaceholder ? (
    <div class="placeholder" style={`aspect-ratio: ${aspectRatio}`}>
      {alt}
      <a href="?grid-aware-disabled=true">Show images</a>
    </div>
  ) : (
    <Image src={src} alt={alt} />
  )
}
```

## Performance concerns

One of the major downsides of this architecture is that the API request blocks rendering of the page while in-flight. This results in a slower time-to-first-byte, my testing with [Calibre](https://calibreapp.com/) found that it was on the order of 1 second. This impacts perceived performance as is a noticeable delay between opening the site and something appearing on screen.

This is a tricky problem, as in order to render parts of the site, the components need to have grid intensity level, which is impossible until the API request completes. Providing a default value while the request is in-flight is non-optimal, since it would lead to a "flash of wrong behaviour", similar to [flash of unstyled text (FOUT)](https://fonts.google.com/knowledge/glossary/fout). Worse, since image loading is dependent on the intensity level, providing a default value could lead to an image request being sent only to become redundant soon after, defeating the purpose of the grid aware site.

To combat this and avoiding in-flight states, I'd like to investigate HTML streaming, allowing parts of the page that are not dependent on the intensity level value to render while other parts of the page have to wait. I only recently learned about a [streaming API in Astro](https://docs.astro.build/en/recipes/streaming-improve-page-performance/) so I'm hopeful this can improve performance.

## Wrapping up

I really enjoyed the rebuild and learned a lot along the way about the grid aware pattern. Although there are tradeoffs, I can see a future where it becomes more widespread across the web.

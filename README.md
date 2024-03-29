![Deploy to Pages](https://github.com/yapjonathan/designpaytransparency/workflows/Deploy%20to%20Pages/badge.svg?branch=main)

# Action-NotionSite

loconotion + Github Actions + Notion + GitHub Pages + Cloudflare Workers, Create a free website or build a blog.<br>

- Use Github Actions to run loconotion regularly to crawl Notion pages and generate pure static html pages.
- Push the generated html pages to the GitHub repository and generate a static web site with GitHub Pages.
- Finally, reverse proxy with Cloudflare Workers to implement a separate domain site.
- Runs every 20 minutes

### Usage

1. Register and create a page of your choice in Notion, click share to get the public link e.g. `https://www.notion.so/Loconotion-Example-Page-03c40xxxxxxxxxxxxxxxx9a8950ef`
2. To sign up for GitHub, click the **use this template** button on this project page to create your new project. We recommend using something like **blog** for the name of your new repository. (You can give this project a Star when you're done creating it)
3. In the GitHub project you just created, click **Settings** and then** Secrets** to create a new profile. Detailed profile of the [original project](https://github.com/leoncvlt/loconotion#advanced-usage). Make sure you use the `loconotion_site_config.toml` content in your secret config.
<details>
<summary>Example configuration file</summary>

**Name:**<br>
`SITE_CONFIG`<br>
**Value:**<br>
```
name = "notion"
page = "https://www.notion.so/Loconotion-Example-Page-03c40xxxxxxxxxxxxxxxx9a8950ef"
theme = "dark"
[site]

  [[site.meta]]
  name = "title"
  content = "Loconotion Test Site"

  [[site.meta]]
  name = "description"
  content = "A static site generated from a Notion.so page using Loconotion"

  [site.fonts]
  site = 'Nunito'
  navbar = ''
  title = 'Montserrat'
  h1 = 'Montserrat'
  h2 = 'Montserrat'
  h3 = 'Montserrat'
  body = ''
  code = ''

  [[site.inject.head.link]]
  rel="icon"
  sizes="16x16"
  type="image/png"
  href="/example/favicon-16x16.png"

  [[site.inject.body.script]]
  type="text/javascript"
  src="/example/custom-script.js"

[pages]

  [pages.d2fa06f244e64f66880bb0491f58223d]
    slug = "games-list"

    [[pages.d2fa06f244e64f66880bb0491f58223d.meta]]
    name = "description"
    content = "A fullscreen list database page, now with a pretty slug"

    [pages.d2fa06f244e64f66880bb0491f58223d.fonts]
    body = 'DM Mono'

  [pages.54dab6011e604430a21dc477cb8e4e3a]
    slug = "film-gallery"

  [pages.2604ce45890645c79f67d92833083fee]
    slug = "books-table"

  [pages.ae0a85c527824a3a855b7f4d31f4e0fc]
    slug = "random-board"
```
</details>

4. Click **Actions** in the GitHub project you just created, then click **the Deploy to Pages** toggle on the left, and then click **Run workflow** to start generating Pages for the first time.(If you don't see Auto Install, click on the .yml file and add a blank line to save it).
After generating here, make sure to check if a new branch **gh-pages** has been created under your project and see if there are folders and html files in that branch.
5. In the GitHub project you just created, click **Settings** and scroll down to **GitHub Pages** and select `gh-pages / root`.
Save it and you'll be able to start GitHub Pages.
Next, you can use `username.github.io/repository/name` to access your static site.
The username is your GitHub name, the repository is the name of your project, and the name on the first line of the configuration file corresponds to the folder under the gh-pages branch.
6. Optional. Register and add your domain on Cloudflare. Create a new Worker, use the following code to reverse proxy `username.github.io/repository/name` Then create a new route corresponding to the domain name corresponding to Workers to achieve domain access. This step online many tutorials.
<details>
<summary>workers.js</summary>

from:[booster.js](https://github.com/xiaoyang-liu-cs/booster.js)
```
const config = {
  basic: {
    upstream: 'https://en.wikipedia.org/',
    mobileRedirect: 'https://en.m.wikipedia.org/',
  },

  firewall: {
    blockedRegion: ['CN', 'KP', 'SY', 'PK', 'CU'],
    blockedIPAddress: [],
    scrapeShield: true,
  },

  routes: {
    TW: 'https://zh.wikipedia.org/',
    HK: 'https://zh.wikipedia.org/',
    FR: 'https://fr.wikipedia.org/',
  },

  optimization: {
    cacheEverything: false,
    cacheTtl: 5,
    mirage: true,
    polish: 'off',
    minify: {
      javascript: true,
      css: true,
      html: true,
    },
  },
};

async function isMobile(userAgent) {
  const agents = ['Android', 'iPhone', 'SymbianOS', 'Windows Phone', 'iPad', 'iPod'];
  return agents.any((agent) => userAgent.indexOf(agent) > 0);
}

async function fetchAndApply(request) {
  const region = request.headers.get('cf-ipcountry') || '';
  const ipAddress = request.headers.get('cf-connecting-ip') || '';
  const userAgent = request.headers.get('user-agent') || '';

  if (region !== '' && config.firewall.blockedRegion.includes(region.toUpperCase())) {
    return new Response(
      'Access denied: booster.js is not available in your region.',
      {
        status: 403,
      },
    );
  } if (ipAddress !== '' && config.firewall.blockedIPAddress.includes(ipAddress)) {
    return new Response(
      'Access denied: Your IP address is blocked by booster.js.',
      {
        status: 403,
      },
    );
  }

  const requestURL = new URL(request.url);
  let upstreamURL = null;

  if (userAgent && isMobile(userAgent) === true) {
    upstreamURL = new URL(config.basic.mobileRedirect);
  } else if (region && region.toUpperCase() in config.routes) {
    upstreamURL = new URL(config.routes[region.toUpperCase()]);
  } else {
    upstreamURL = new URL(config.basic.upstream);
  }

  requestURL.protocol = upstreamURL.protocol;
  requestURL.host = upstreamURL.host;
  requestURL.pathname = upstreamURL.pathname + requestURL.pathname;

  let newRequest;
  if (request.method === 'GET' || request.method === 'HEAD') {
    newRequest = new Request(requestURL, {
      cf: {
        cacheEverything: config.optimization.cacheEverything,
        cacheTtl: config.optimization.cacheTtl,
        mirage: config.optimization.mirage,
        polish: config.optimization.polish,
        minify: config.optimization.minify,
        scrapeShield: config.firewall.scrapeShield,
      },
      method: request.method,
      headers: request.headers,
    });
  } else {
    const requestBody = await request.text();
    newRequest = new Request(requestURL, {
      cf: {
        cacheEverything: config.optimization.cacheEverything,
        cacheTtl: config.optimization.cacheTtl,
        mirage: config.optimization.mirage,
        polish: config.optimization.polish,
        minify: config.optimization.minify,
        scrapeShield: config.firewall.scrapeShield,
      },
      method: request.method,
      headers: request.headers,
      body: requestBody,
    });
  }

  const fetchedResponse = await fetch(newRequest);

  const modifiedResponseHeaders = new Headers(fetchedResponse.headers);
  if (modifiedResponseHeaders.has('x-pjax-url')) {
    const pjaxURL = new URL(modifiedResponseHeaders.get('x-pjax-url'));
    pjaxURL.protocol = requestURL.protocol;
    pjaxURL.host = requestURL.host;
    pjaxURL.pathname = pjaxURL.path.replace(requestURL.pathname, '/');

    modifiedResponseHeaders.set(
      'x-pjax-url',
      pjaxURL.href,
    );
  }

  return new Response(
    fetchedResponse.body,
    {
      headers: modifiedResponseHeaders,
      status: fetchedResponse.status,
      statusText: fetchedResponse.statusText,
    },
  );
}

// eslint-disable-next-line no-restricted-globals
addEventListener('fetch', (event) => {
  event.respondWith(fetchAndApply(event.request));
});
```

</details>

### Tips
- The speed of access depends on how fast you can access GitHub.
- Using Cloudflare free workers need to pay attention to the number of daily visits. cloudflare has a CDN effect, which can increase the speed of access to some extent.
- This project template is just a quick build project, the core project is the key to capture Notion. If you have any questions about "crawling" and "generation", we suggest you go directly to the core project. >[loconotion](https://github.com/leoncvlt/loconotion)
- The default is not enabled to run every 20 minutes, you need to modify the `.github/workflows/pages_deploy.yml` file, remove the only two "#" signs to enable.

# Acknowledgments
- [loconotion](https://github.com/leoncvlt/loconotion)
- [booster.js](https://github.com/xiaoyang-liu-cs/booster.js)
# License
[MIT](https://github.com/artxia/Action-NotionSite/blob/main/LICENSE) © XIA
# Docker - Node + Puppeteer

[![Image information](https://images.microbadger.com/badges/image/unindented/node-puppeteer.svg)](https://microbadger.com/images/unindented/node-puppeteer) [![Version information](https://images.microbadger.com/badges/version/unindented/node-puppeteer.svg)](https://microbadger.com/images/unindented/node-puppeteer)

## Usage

Create a new project with a `package.json` that looks something like this:

```json
{
  "name": "your-project",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "chrome-launcher": "0.10.5",
    "got": "9.6.0",
    "lighthouse": "4.3.0",
    "puppeteer": "1.15.0"
  }
}
```

And an `index.js` file that looks like this:

```js
const chromeLauncher = require("chrome-launcher");
const got = require("got");
const lighthouse = require("lighthouse");
const puppeteer = require("puppeteer");

const url = "https://www.unindented.org";

(async () => {
  let chrome;
  let browser;

  try {
    chrome = await chromeLauncher.launch({
      chromePath: "google-chrome-unstable",
      chromeFlags: ["--headless", "--disable-gpu"],
      logLevel: "error"
    });

    const { port } = chrome;
    const { body } = await got(`http://localhost:${port}/json/version`);
    const { webSocketDebuggerUrl } = JSON.parse(body);

    browser = await puppeteer.connect({
      browserWSEndpoint: webSocketDebuggerUrl
    });

    const { lhr } = await lighthouse(url, { port, output: "json" });
    console.log(
      `Lighthouse scores: ${Object.values(lhr.categories)
        .map(c => c.score)
        .join(", ")}`
    );
  } catch (err) {
    console.error(err);
  } finally {
    try {
      await (browser && browser.disconnect());
    } finally {
      await (chrome && chrome.kill());
    }
  }
})();
```

Your `Dockerfile` could have the following content:

```dockerfile
FROM unindented/node-puppeteer:latest

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD true

COPY . /app/
WORKDIR /app/

RUN npm i \
  # Add user so we don't need `--no-sandbox`. Same layer as `npm install` to
  # keep re-chowned files from using up several hundred MBs more space.
  && groupadd -r pptruser \
  && useradd -r -g pptruser -G audio,video pptruser \
  && mkdir -p /home/pptruser/Downloads \
  && chown -R pptruser:pptruser /home/pptruser \
  && chown -R pptruser:pptruser /app

# Run everything after as non-privileged user.
USER pptruser

CMD ["npm", "start"]
```

Build your image:

```
docker build -t your-image .
```

And then run it:

```
docker run --rm -it --cap-add=SYS_ADMIN your-image
```

## Meta

- Code: `git clone https://github.com/unindented/docker-node-puppeteer.git`
- Home: <https://github.com/unindented/docker-node-puppeteer>

## Contributors

Daniel Perez Alvarez ([unindented@gmail.com](mailto:unindented@gmail.com))

## License

Copyright (c) 2019 Daniel Perez Alvarez ([unindented.org](https://unindented.org/)). This is free software, and may be redistributed under the terms specified in the LICENSE file.

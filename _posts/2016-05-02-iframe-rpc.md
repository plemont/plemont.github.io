---
layout: post
title:  "iframe RPC"
date:   2016-05-02 21:00:00 +0100
categories: javascript browsers
---
## Introduction

Recently, I started work on a project in which I thought I might use an
`iframe` within the application to handle some of the work.

As a result I investigated communication between a web page and contained `iframe` elements, hosted on the same domain.

Ultimately, this part of the project took a different turn, so [this code](https://github.com/plemont/iframe-rpc) is entirely unloved.

## Posting messages between windows

A typical setup I was looking at would be something like:

```html
<body>
 <iframe id='iframe' src='iframe.html'></iframe>
</body>
```

Given that both the parent page, and the `iframe` (loaded from `iframe.html`) have the same origin, it is possible to communicate between the two using [`Window.postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage).

For example, sending from `iframe` to parent:

```javascript
window.parent.postMessage('Hello!', window.location.origin);
```

and from parent to `iframe`:

```javascript
let target = document.getElementById('iframe').contentWindow;
target.postMessage('Hello!', window.location.origin);
```

This is pretty straight forward, but a little bit *fire-and-forget*:

*   Did my message get through?
*   What was the result? Was there any response?

## Something a little more workable...

It was evident this was going to be too basic for my needs.

What I really needed was the means to:

*   Call a **function** in the iframe/parent
*   Pass **arguments** to that function
*   Get a **response** from that function
*   Be alerted to **errors**, including where the called function does not **exist**
*   Set a **timeout** on the function execution

*Phew!* That's quite a different set of requirements, but if I could implement that, things would then get quite a bit more elegant. I would be able to write code such as:

```javascript
// Call the 'add' function in the parent window of this iframe
rpc.execute('parent', 'add', [1, 2])
    .then(result => {
      console.log(result);
    })
    .catch(error => {
      console.log('Error:' + error);
    });
```

That sounds quite nice. Hmmm maybe I could give that a shot?

## Implementing function execution

Fortunately, the [`Promise`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise) object seemed perfect for this scenario.

Using a `Promise` allows a message to be passed in one direction to to **call** the function, and then wait asynchronously for the **response**, at which point the promise can be resolved.

A *listener* can be set up to listen for the response, specifically for a `requestId` associated with a given call. This is torn down on success:

```javascript
let listener = window.addEventListener('message',
    event => {
      if (event.origin === window.location.origin &&
          event.data.requestId === message.requestId &&
          event.data.targetId === this.sourceId_ &&
          event.data.sourceId === message.targetId &&
          event.data.type === 'response') {
        // Remove the listener - it was just for this 'function call'
        window.removeEventListener('message', listener);
        // Remove any alerting to RPC timeout
        clearTimeout(fail);
        if (event.data.responseType === 'success') {
          resolve(event.data.response);
        } else {
          reject(event.data.response);
        }
      }
    });
```

Finally, a timer can be set, whereby if no response is received, the `Promise` can be **rejected** (causing it to fail), which allows a timeout to be implemented:

```javascript
let fail = setTimeout(() => {
  window.removeEventListener('message', listener);
  reject('No response to request for: ' + message);
}, this.timeout_);
```

## Listening for function calls

The other side of this exchange, listening for incoming messages requesting execution, is a simple event hander:

```javascript
window.addEventListener('message', event => this.messageHandler(event));
```

This simply performs some basic checks before attempting to execute the function and return the results:

```javascript
try {
  let func = this.context_[data.functionName];
  message.response = func.apply(this.context_, data.argumentList);
  message.responseType = 'success';
} catch (e) {
  message.response = e.message;
  message.responseType = 'error';
}
```

## Conclusion

This was pretty fun to write, and ultimately of absolutely zero use to me as the project took a different path. Nevertheless, it has shown me that it is relatively simple to get function execution across the page/iframe boundary, and that using a `Promise` is an ideal way to simplify this.

The code, for what it is worth, lives at [https://github.com/plemont/iframe-rpc](https://github.com/plemont/iframe-rpc)
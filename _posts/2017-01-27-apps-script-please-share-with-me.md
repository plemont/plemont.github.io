---
layout: post
title:  "Apps Script, please share with me"
date:   2017-01-27 09:00:00 +0000
categories: javascript apps-script
---
One of the great features of Google Apps is the ability to collaborate with
ease. Users of Google Docs, Sheets and Slides, as well as other applications,
will be familiar with the consistant approach to sharing through the
ever-present **Share** button:

![share](/images/share.png)

Equally, when using Apps Script, or the Drive API, sharing one's work with
others to facilitate collaboration is easy, for example:

```javascript
spreadsheet.addEditor('jane@example.com');
```

really needs no explanation.

## Please... can I join in?

Less straightforward, however, is programmatically *requesting* access to a
document.

Hang on, you might say, when would that ever be useful? Sounds a bit
contrived. Well, admittedly, it might happen less often than wanting to share a
newly-created doc with others, but there can be plenty of cases. Particularly
when faced with a long list of document URLs, from a diverse set of contacts or
customers, all of which you're wanting access to, for one reason or another.

Browsing through the [Apps Script documentation](https://developers.google.com/apps-script/)
draws a blank...

## Chrome DevTools to the rescue!

Despite no joy within Apps Script, requesting access is clearly possible as most
people will be familiar encountering the following page when attempting to open
a document without permission:

![share](/images/request.png)

Firing up DevTools and observing the requests that are made, shows two requests:

*   The first returns a **403 Forbidden** error, but also a token of some sorts.
*   The second is exactly as per the first, but with the inclusion of the token
    in the form submission. The second returns a **200 OK** response.

![share](/images/commonshare.png)

It wasn't exactly clear to me why this particular flow is being used, but it
seemed likely, that as these requests are to methods on *docs.google.com*, then
perhaps a single OAuth-authenticated request, using the *Drive* scope could
work.

This could easily be put together in Apps Script as follows:

## Apps Script implementation

```javascript
function requestShare(docId) {
  // DriveApp.createFile('', '');
  var request = {
    requestType: 'requestAccess',
    itemIds: docId,
    foreignService: 'explorer',
    shareService: 'explorer',
    authuser: 0
  };
  var url = 'https://docs.google.com/sharing/commonshare';
  var params = {
    method: 'POST',
    payload: request,
    headers: {
      Authorization: 'Bearer ' + ScriptApp.getOAuthToken()
    },
    contentType: 'application/x-www-form-urlencoded'
  };
  var response = UrlFetchApp.fetch(url, params);
  // Returned JSON expects {"status":0} on success.
  var data = JSON.parse(response.getContentText());
  if (!data.hasOwnProperty('status') || data.status !== 0) {
    throw Error('Share request unsuccessful!: ' + response.getContentText());
  }
}
```

The above snippet defines a function `requestShare`, which takes a single
argument - the document for which access is to be requested.

It is a pretty straightforward sample - just building a form-based `POST`
request, and attaching the OAuth access token. Perhaps a few things to comment
on however:

1.  **What's with the commented-out `DriveApp.createFile...`?** Well, remember
    that for the request to be permitted, the access token must have been
    granted with permissions to access Drive.

    This commented-out use of `DriveApp` tricks Apps Script into prompting for
    permissions to access Drive, thereby adding Drive to the OAuth scopes
    requested. As it is commented out it has no effect on the execution.
2.  **One of the properties is `itemIds`, can it take multiple IDs?** I've
    guessed at a number of possible formats here for supplying lists, but with
    no luck.
3.  **`shareService`? `foreignService`?** Sounds very exciting! No idea what the
    exact definition of these properties is, so in the example I've just kept
    the values as taken from DevTools. A brief experiment suggests that the
    exact values don't matter.

## That's all

Just remember, this didn't appear in the Apps Script documentation, so I
wouldn't rely on it. It worked... for now... for me...

---
layout: post
title:  "Dynamic dashboarding with the Slides API"
date:   2016-12-08 21:00:00 +0000
categories: javascript slides-api
---

Recently, the [Slides API](https://developers.google.com/slides/) was released.
I thought it might be worth playing with within
[Apps Script](https://github.com/plemont/preso/blob/master/update_preso_appsscript.js)
and [AdWords scripts](https://github.com/plemont/preso/blob/master/update_preso_adwordsapp.js).

However... it's not actually available natively in either at this time - so
using it is a little involved - more on that later. First of all I'll describe
the problem I was looking to tackle.

![preso](/images/preso-title.png)

## Dynamic visualisation using Slides

Near-real time graphs, tables and stats, shown on wall-mounted displays can look
great. I wondered whether I could use Slides to do this.

There are plenty of other visualisation tools out there, but the attraction of
the Slides API was the potential to manipulate *every* aspect of the
presentation, in a product that is widely available and generally free to use
with few restrictions.

So what was the aim:

*   To be able to **update charts, tables and text dynamically** from Apps
    Script or AdWords Scripts.
*   To be able to show such visualisations **full screen** in a compelling
    manner and have them **update automatically**.
*   To be able to **rotate automatically through such visualisations**.

This is just a basic example. In reality, you might also want to do cool things
like:

*   Change the background image depending on the weather or season
*   Insert alerts to mark team members' birthdays or special events!
*   Maybe rotate in a 'fact of the day' slide, just for fun!

As mentioned, the Slides API is not currently available in AdWords scripts or
Apps Script. It would make sense for Google to add it - Spreadsheets and
Documents are already available - but until such time, you'll need to refer to
the [setting up with the Slides API](#slides-api-setup) section, for the
necessary set up actions.

### The `Presentation` object

The starting point for this project is to retrieve a presentation, and this is
easy enough with the Slides API. An OAuth2-authenticated request can retrieve
the presentation as follows:

```
var url = 'https://slides.googleapis.com/v1/presentations/' + presentationId;
var response = UrlFetchApp.fetch(url, options);
var presentation = JSON.parse(response);
```

The [reference describes](https://developers.google.com/slides/reference/rest/v1/presentations#Presentation)
the `Presentation` object. From this documentation, you will see the structure:

![slides](/images/slides.png)

The `Presentation` object has a number of properties, but the one I'll
concentrate on, is the `slides` array, with each entry representing a slide in
the presentation.

### Charts linked to Google Sheets

Slides offers the ability to embed charts - bar chart, pie charts etc - that are
linked to a Google Sheets document.

By default, however, updating the data in the Sheets document **does not cause
the Slides visualisation to update in turn**. This behaviour is quite
reasonable: If you've prepared a presentation based on a spreadsheet, you don't
want the message to change between preparing it and delivering it.

Instead, Slides has a hover-over button that indicates that the chart can be
updated with fresher data that has become available.

![slides](/images/chart-update.png)

#### Refreshing charts through the API

Fortunately, the Slides API exposes the [`refreshSheetsChart`](https://developers.google.com/slides/reference/rest/v1/presentations/request#refreshsheetschartrequest)
request type. Having identified those `pageElement`s which are linked-charts,
refreshing the chart is as simple as the following request structure via
`batchupdate`:

    {
      "refreshSheetsChart": {
        "objectId": objectId
      }
    }

Identifying those `pageElement`s that are linked-charts is also simple: They
contain a `sheetsChart` property. In the sample application, the
`createRefreshSheetsChartsRequests` function demonstrates creating refresh
requests for all charts found in a given presentation.

### Updating text elements

![slides](/images/text.png)

Just as it is required for charts to be updateable, it will be necessary to
update text labels too. For example, a given label on the presentation may be
updated to display when the data was last updated.

The Slides API provides the [`replaceAllText`](https://developers.google.com/slides/reference/rest/v1/presentations/request#ReplaceAllTextRequest)
method, which allows all text in a presentation to be replaced. However, this is
actually not as useful here as it seems. We wish to replace text *in a given
element* - keeping track of the text that is in it is not the way to do this.
An alternative approach is required.

#### Renaming text elements

The solution, to enable repeated replacement of the text in elements of a slide,
is to do some *pre-processing* on the presentation:

*   In creating the slides, the user marks those text labels that should be
    dynamically replaced by entering the text "${label_name}".
*   At the beginning of each execution of the script, `pageElement`s are
    searched for where:
    *   The text is of the form "${label_name}".
    *   The `objectId` of the form is still just a random ID.
*   If found, these `pageElement`s `objectId`s are renamed to the form
    `<script_prefix>_<label_name>`.

![slides](/images/preprocessing.png)

This approach can be seen in `createTextAndTableRenameRequests` in the script.

#### Updating text elements

This pre-processing helps as for the text update stage, the following approach
is then taken:

*   Search for all `pageElement`s where the `objectId` has the special prefix.
*   Extract the label name from the second half of the ID
*   Look up the label in a map of text to be replaced, and if found, replace
    with the corresponding value.

In reality there are two further complications:

*   Elements in a presentation cannot have their `objectId`s changed. Instead
    the element must be duplicated with the desired new name, and the original
    then deleted.
*   Similarly, text elements cannot have their text changed. Instead, the old
    text element is deleted, and a new one inserted.

The update process can be seen in `createTextReplacementRequests`. A map is
provided for example:

```
var mapping = {
  'myHeading1': 'Latest figures - Q4' 2016',
  'myHeading2': 'Data refreshed on 01 Dec 2016'
};
```

Will replace the text in elements that originally had text *${myHeading1}* and
*${myHeading2}* respectively when the presentation was first created by hand.

### Updating tables

The approach for updating tables is essentially exactly the same, with the
following minor differences:

*   When first manually creating the table in the presentation, the *${...}*
    label should be placed in the top-left cell. Pre-processing then renames
    the table object accordingly.
*   The update process uses a structure such as:

```
var tables = {
  'testtable': {
    id: '<...Sheets ID...>',
    sheetName: 'TableData'
  }
};
```

In the above example, a table in the presentation that had been first created
with *${testtable}* in the top-left cell would, each time the script ran, be
updated with data from the *TableData* sheet in the given spreadsheet.

### Fullscreen refreshes

In the above sections, I've explained how to update charts, tables and text in a
presentation, however there remains a problem: Once in presentation mode in
Slides (likely fullscreen), irrespective of whether the Slides document is
updated, the charts, tables and text will not be redrawn.

It would appear that when in presentation mode, the slide is rendered to a
`Canvas` and - I guess - the need to update the display once in this mode (ever)
was not envisaged.

Therefore, a way is needed to:

*   Periodically refresh the page - advancing the slide if there are more than 1
*   Force the page to fullscreen

#### Advancing slides

The first item to deal with here, is advancing slides. Fortunately, Slides
allows the current slide to be selected through the URL parameter `slide=`.

The script therefore checks and renames each slide so that the names are in the
form `prefix_1_n` to `prefix_n-1_n`.

Each time the script runs, this check is performed, so if the owner of the
presentation has reordered slides, or added or deleted slides, that's no
problem.

### Chrome extension

How this helps is that a [small chrome extension](https://github.com/plemont/preso/tree/master/extension)
is able to parse the URL of the current page, and if it is a presentation, work
out what the URL of the next slide should be.

Furthermore, the extension can invoke fullscreen mode, as well as scheduling the
period fetches that are required.

With the extension installed, when navigating to the presentation view for a
Slides document, e.g.:

    ```
    https://docs.google.com/presentation/d/...id.../present
    ```

an icon appears to the right of the navigation bar:

![icon](/images/p-icon.png)

Simply clicking the icon will set the visualisation going.

### Slides API setup

As mentioned, the Slides API is not yet available within Apps Script or AdWords
scripts.

#### Using the Slides API within Apps Script

Within the script editor:

1.  Click on **Resources > Developers Console Project...**
2.  Click on the project name to open up the *Developer Console*.
3.  Search for **Slides API** and click on **Enable**.

Within the script:

Ensure that `DriveApp` is being used in the script: This will prompt Apps Script
to add the `https://www.googleapis.com/auth/drive` to the project. This scope
includes the ability to work with presentations.

The example script uses a commented DriveApp statement: This is enough to prompt
the user for permissions.

Having performed this setup, calling the Slides API via the REST interface is
as simple as passing the OAuth token in the header, as obtained from
`ScriptApp.getOAuthToken()`, as can be seen in the [sample script](https://github.com/plemont/preso/blob/master/update_preso_appsscript.js).

#### Using the Slides API within AdWords Script

Working within AdWords Scripts is a bit more involved than with Apps Script as
`ScriptApp` is not available to provide an OAuth token.

Instead, setup is as follows:

*   Create a new project in the **Developers Console**.
*   Enable the **Slides API** and the **Drive API**.
*   Create new OAuth credentials:
    *   Click on **Create credentials > OAuth client ID**.
    *   Choose *Application type:* **Other**

*   Take the *Client ID* and *Client Secret* from this process and generate a
    *Refresh token* using [this script](https://github.com/plemont/preso/blob/master/generate_refresh.js).

Making requests:

*   Use the *ClientID*, *Client Secret* and *Refresh token* with the
    [sample OAuth2 library](https://developers.google.com/adwords/scripts/docs/examples/oauth20-library)
    using the `OAuth2.withRefreshToken()` method.
*   Authenticated requests can then be made with the resulting object as can be
    seen in the [sample code](https://github.com/plemont/preso/blob/master/update_preso_adwordsapp.js).




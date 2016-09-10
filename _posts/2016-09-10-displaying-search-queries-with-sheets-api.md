---
layout: post
title:  "Displaying search queries with the Sheets API"
date:   2016-09-10 21:00:00 +0100
categories: javascript sheets-api pwa
---
![aggregate]({{ site.url }}images/grid-example.png)

## Introduction

Many people will be familar with the [Google Trends Screensaver](https://www.google.com/trends/hottrends/visualize?nrow=3&ncol=4):
It's an eye-catching way of showing trending searches.

It set me thinking about how to build something similar for displaying
information from my own feeds. A specific use case in mind was data from the
[Search Query Performance Report](https://developers.google.com/adwords/api/docs/appendix/reports/search-query-performance-report).
This data shows AdWords advertisers the queries that are resulting in their
ad impressions. This can make a great way of visualising what queries are
leading to ads being shown.

I have also been using the Sheets API at work, and wanted to find a reason to
play with it outside of work.

## Using Sheets as an aggregator

A key reason why the use case I mention is interesting in the context of Sheets
is that Sheets can be used as an aggregation platform: Advertisers will often
have many accounts. By using Sheets, I can have the Search Query Performance
Report write to separate sheets in a single document, and then retrieve **all**
the data with a single call to the Sheets API.

## Obtaining Search Query Performance Report data

The easiest way to obtain SQPR data is through [AdWords Scripts](https://developers.google.com/adwords/scripts/).
Some [simple scripts](https://github.com/plemont/griddysheets/adwords_scripts)
allow each account to write the data out to a single spreadsheet.

![aggregate]({{ site.url }}images/aggregate.png)

## Fetching data via the Sheets API

Having written data from all accounts to a single Sheets file, pulling that
data from [JavaScript is a single call](https://github.com/plemont/griddysheets/blob/master/app/scripts/sheets.js):

```javascript
gapi.client.sheets.spreadsheets.get({
  spreadsheetId: spreadsheetId,
  includeGridData: true
}).then(updateSheetData, handleFetchError);
```

The key here to ensure that all data is retrieved from all sheets is the
`includeGridData` property being set.

## Visualisation

To visualise the data, I built on the [Web Starter Kit](https://github.com/google/web-starter-kit)
to build an app which works as well on a PC or big screen as it does on a mobile
device.

![mobile]({{ site.url }}images/grid-mobile.png)

## Details

The project is [available on github](https://github.com/plemont/griddysheets).
Consult the README.md file for details on setting it up.

I have published my version at [griddysheets.firebaseapp.com](https://griddysheets.firebaseapp.com)

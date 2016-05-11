---
layout: post
title:  "File format conversion within Apps Scripts"
date:   2016-05-11 21:00:00 +0100
categories: javascript apps-scripts
---
## Introduction

Working on a recent [Apps Scripts project](https://developers.google.com/apps-script/),
I had the need to convert images from whatever format I found them in into
BMP format.

Wait... *BMP format*? Yes, seriously. Well, I didn't *have* to but it actually
turned out to be the easiest way to achieve what I needed: I wanted to
manipulate images at a pixel level, and whilst the [PNG spec](https://www.w3.org/TR/PNG/)
turned out to be really readable and not too hard to implement, BMP was still 
plain easier for getting quickly to pixel data.

So, pained as I was to use the BMP format, this was a perfect opportunity to use
the power of the [`Blob`](https://developers.google.com/apps-script/reference/base/blob)
class, to get some conversion for free.

## Format conversion with Blobs

Apps Scripts provides a class, the [`Blob`](https://developers.google.com/apps-script/reference/base/blob),
which can be used for converting data between one format and another.

In addition to this class, a range of other Apps Scripts objects implement the
the [`BlobSource` interface](https://developers.google.com/apps-script/reference/base/blob-source)
which allows them to be used as the source of a conversion from one format to
another.

In either case, conversion can be achieved by:

```javascript
var newFormatBlob = myBlobSource.getAs('<mimetype>');
```

If the object being converted can be converted to the new format, then that's
all! (If the conversion doesn't make sense, like converting a zip file to a PNG,
an error will be thrown.)

## Converting images

So, as you'll recall, I needed to convert a range of images to BMP. These images
happened to be located online. The solution I ended up using was roughly the
following:

```javascript
/**
 * @param {string} url The URL of the image.
 * @param {string} destMimeType The MIME type for the converted image.
 * @return {Blob} The converted image.
 * @throws {Error} On fetch failure or conversion failure.
 */
function convertImage(url, destMimeType) {
  var data = UrlFetchApp.fetch(url);
  var contentType = data.getHeaders()['Content-Type'];

  var image = Utilities.newBlob(data.getContent(), contentType);
  var convertedImage = image.getAs(destMimeType);
  return convertedImage;
}

function main() {
  var bmp = convertImage('http://example.com/hello.jpg', MimeType.BMP);

  // ... do something with my BMP ...
}
```

The only addition here from what I've already described is that the
[Utilities](https://developers.google.com/apps-script/reference/utilities/utilities)
class provides the means to create a `Blob` from a `Byte[]` array through the use
of [`newBlob`](https://developers.google.com/apps-script/reference/utilities/utilities#newblobdata-contenttype-name)

## Simplifying conversion

As it turns out, this process can be simplfied: [`HTTPResponse`](https://developers.google.com/apps-script/reference/url-fetch/http-response),
as returned from
[UrlFetchApp.fetch](https://developers.google.com/apps-script/reference/url-fetch/url-fetch-app#fetchurl)
*implements the `BlobSource` interface* - how epic is that? So we can instead write:

```javascript
/**
 * @param {string} url The URL of the image.
 * @param {string} destMimeType The MIME type for the converted image.
 * @return {Blob} The converted image.
 * @throws {Error} On fetch failure or conversion failure.
 */
function convertImage(url, destMimeType) {
  var data = UrlFetchApp.fetch(url);
  var convertedImage = data.getAs(destMimeType);
  return convertedImage;
}
```

as `HTTPResponse` is capable of of converting the content of the response
*directly* to an image. No need to extract the MIME type from the headers or
provide an array of bytes.

## Conclusion

The `BlobSource` interface is a really nice feature within Apps Scripts. If
there is one complaint, it is that beyond images, some of the conversions you
might expect between various document formats, don't appear to work so readily.

I hope that in the future, further conversions are added, and further
classes choose to implement `BlobSource`.

And that I can avoid working with BMPs in the future too...
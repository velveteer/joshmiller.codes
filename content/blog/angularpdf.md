+++
date = "2014-10-08T15:43:24-06:00"
draft = false
title = "Server-side PDF generation and Angular $resource"

+++

I ran into a problem recently where my Angular application needed to consume an API call that returned a PDF response. Apparently there are options to generate PDF documents client-side, such as Mozilla's [PDF.js](http://mozilla.github.io/pdf.js/), but in this case I had no control over the data. The scenario looked like this:

1. Send a HTTP request with an Accept header of `application/pdf` and expect to recieve a PDF byte-stream response.
2. Allow the user to download this PDF from their browser after making the call.

I had no access to the server in this scenario since my Angular application is stateless and relies primarily on ngResource to consume external API payloads. I simply wanted the user to click a "Download PDF" button and have the browser prompt them to download the file. I managed to accomplish this by adding a new method to the ngResource object:

{{< highlight javascript >}}
pdf: {
    method: 'GET',
    headers: {
        accept: 'application/pdf'
    },
    responseType: 'arraybuffer',
    cache: true,
    transformResponse: function (data) {
        var pdf;
        if (data) {
            pdf = new Blob([data], {
                type: 'application/pdf'
            });
        }
        return {
            response: pdf
        };
    }
}
{{< /highlight >}}

There are a few things going on here that allow me to consume a PDF byte-stream as a resource and save it into a [Blob object](https://developer.mozilla.org/en-US/docs/Web/API/Blob). Once I have a Blob object I can use a third-party library like [FileSaver.js](https://github.com/eligrey/FileSaver.js/) to present a download prompt.

The caching isn't necessary here, but it is convenient to prevent strain on the server if multiple calls are made. The `responseType` is necessary, otherwise the byte-stream is interpreted as a string and gives the user a corrupted PDF.

This was a unique scenario that involved ngResource and an API that used the same endpoint to return XML/JSON data along with a PDF document based on Accept headers. If anyone else finds themselves in a similar situation, give this a shot.


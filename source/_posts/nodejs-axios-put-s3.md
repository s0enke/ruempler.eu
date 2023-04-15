---
date: "2022-09-13 12:00"
title: "How to (PUT) upload files to S3 with Typescript and Axios"
---

## Situation

You want to upload a file to S3 with Axios and the `PUT` method.

## Challenge

S3 needs a Content-Length for `PUT` requests, otherwise it throws an 501.

<!--more-->

## Solution

Upload the file as a stream and pass the size as header.

```typescript
  await axios.put(s3_upload_url, fs.createReadStream(filename), {
    headers: {
      'Content-length': fs.statSync(filename).size,
    }
  });
```
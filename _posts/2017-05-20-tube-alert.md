---
layout: post
title: Building TubeAlert, a Serverless Progressive Web Site with push notifications
author: 
    name: David Marland
    twitter: djmarland
image: /assets/tube-alert/banner.svg
---

[TubeAlert](https://tubealert.co.uk) is a blah balha Aliquam molestie ullamcorper enim, a accumsan velit dignissim a. Nulla ac lorem ut leo maximus condimentum. Integer vitae dolor nulla. Aenean vel est volutpat, dapibus dolor ut, ultricies nunc. Aliquam erat volutpat. Pellentesque eu neque pellentesque, vulputate dolor vitae, feugiat orci. Nunc lacus ligula, fringilla sed metus quis, bibendum convallis orci. Pellentesque a sapien tristique, placerat mi sit amet, venenatis est.

[![Screenshot of the TubeAlert website](/assets/tube-alert/screenshot.png)](https://tubealert.co.uk)

## Table of contents
{:.no_toc}
This is a long read, so here's a table of contents in case you just want to skip to the parts you are interested in.

* TOC
{:toc}

## todo
- Briefly mention the goals and the previous site.
- Discuss lambda and being function based. Therefore show the workflows.
- Discuss SQL vs NoSQL. Prefer SQL but can get away with NoSQL here because it is free - How to use the Indexes
- API Gateway and S3
- Cloudflare, caching and HTTPS
- Full infrastructure diagram
- React site, using latest.json
- Offline & Push with service worker
- Progressive enhancement


## Open as app
<video class="prose__video" muted loop controls>
    <source
        src="/assets/tube-alert/open-as-app.webm"
        type="video/webm">
    <source
        src="/assets/tube-alert/open-as-app.mp4"
        type="video/mp4">
</video>


## Gotcha 1 - Lambda containers
Lambda functions run in containers. To improve start-up performance AWS may keep the container around once your function completes. Then if you run the function again your application is already bootstrapped. You can't guarantee it bit those containers may stick around for up to 24 hours.


In Nodejs this means if you set a variable in the scope of the file, it will still be set on the next invocation. Therefore the TubeAlert DI (Dependency Injection) container was written to instantiate new objects each time the function is run, rather than creating them once at the top of the file

(code) vs (code)

This is particularly important with DateTime, otherwise the application date will be frozen for every invocation until the container is terminated.

Another side effect is that memory leaks are a more pressing concern. It appeared to be the case that the longer the container live the slower the function became, until it reached the timeout and errors prevailed. Then, overnight, the container terminated and the cycle reset. 

(graph)

There was an investigation into the code to ensure no data is unnecessarily hanging around. However, libraries or processes outside of your function (such as growing log files) are out of your control.

Lambda allows you to choose and amount of memory allocated to your function. Original this was set to 128MB for all functions. Although the logs were showing the `fetch` function usage as being well below that the slowdowns and timeouts were still occurring. It seems that the underlying CPU, network and disk performance are correlated (though not declared) to your choice of memory allocation. By upping the allocation to 256MB the `fetch` function now sits at a stable ~1000ms. The timeout was also increased to 10 seconds but it now never reaches that. The lambda erros graph is now flat.

(graph)

We can take advantage of the container behaviour. For the `latest` endpoint we know that any requests with 2 minutes will be the same, as that is the rate the data is fetched from TFL. Therefore Node can store the DynamoDB result in a variable that will persist. Any subsequent calls within two minutes will reuse that data immediately, improving performance and keeping the DyanmoDB usage low.

## Push notificaiton
<video class="prose__video" muted loop controls>
    <source
        src="/assets/tube-alert/notification.webm"
        type="video/webm">
    <source
        src="/assets/tube-alert/notification.mp4"
        type="video/mp4">
</video>

## Gotcha 2 - Service Worker Cache and S3

The static assets (CSS and JS) are stored on S3, and cached for a year with HTML headers. We also want to cache them in the service worker so that the application can work offline.

However, since the static asseets are on a different domain this would encounter CORS issues. No problem, AWS allows you to set CORS headers on your bucket. *BUT* the S3 response will only include those CORS headers if the request contains and `Origin` header.

Client side JavaScript calls will include this header, such as the one made in the service worker `cacheAll` call. Linked CSS and JavaScript files included via HTML tags (`<link>`/`<script>`) will not include this header, which is usually fine as they don't need it.

What is happening though is the page is being loaded and the assets are being fetched. Those assets respond *without* CORS headers, but with the expected year long `cache-control`. The browser will then put those files into its disk cache (for a year). Next the service worker loads and activates, performing the `cacheAll` command. It attempts to go and fetch the files to be cached, send the `Origin` header. However, since the files are in the disk cache this call doesn't go to the server. The browser pulls the files from its own cache, sees that there are no CORS headers and rejects the request as insecure. Browsers do not vary their cache based on the presence of the `Origin` header.

The solution is to have the service worker call a slightly different URL by adding a query string.
```
https://static.tubealert.co.uk/34135971791.app.js?sw
```
(todo - check this actually works)

## Webpack setup
Webpack can be set up in various ways. For TubeAlert there are four webpack configs required. There is also a `webpack.base.config.js` which several of them inherit from (to save repeating the same options every time). The main shared portions of config are to setup Sass, and Babel so that ES2015 class structure and imports can be used

### `webpack.dev.config.js`
This is used during development and is activated via `yarn dev` or to provide a server `yarn server`. The server makes use of the webpack dev server to offer hot reloading during development. This is needed for convenient development of the React application

### `webpack.prod.config.js`
This is activated via `yarn prod`. It is run during the Travis build and generates the application JavaScript and CSS. This uses production mode for react and minifies the output to reduce filesize. It creates files named by hash ready for upload to the S3 static location.

### `webpack.build.config.js`
The lambda function for the webapp is running in node, which cannot understand ES2015 imports or JSX syntax natively. Therefore this webpack file builds the app.js for the lambda function. It has a different entry point to the client side version of the application as it needs to return a html response rather than activate a DOM element. This also runs ESLint and will fail the Travis build if there are any issues.

### `webpack.sw.config.js`
This simply builds the service worker. The service worker needs to know the hashed address of the static assets in order to cache them, so this webpack method will embed the manifest file (generated from `webpack.prod.config.js`) into the service worker itself. As the hash for the assets will be different when there are changes this also means the service worker will update due to the embedded manifest value now having different values.

## S3 Bucket setup
Cloudflare is setup in front of https://static.tubealert.co.uk. In order to support this the bucket name must be the same as the domain. Cloudflare DNS is setup like so...
The bucket itself is setup via the `Resources` section of the serverless.yml config. This is simply Cloudformation config.
CORS is activated on the bucket to allow the front end application (particularly the service worker) to be able to access these assets.
(code)
AWS recommend that your naming structure of files in your bucket is as spread/random as possible. This is because the files are partitioned to different servers according to filename. If all your files are similar in filename then you risk all your requests coming from the same server which can result in lower performance. A "folder" in S3 is only virtual so it is part of the filename. For this reason our static bucket will not have any folders.

We want the static files to be cached for a long time (1 year), so the filename contains the hash of the contents. If the contents change then the filename will change. It also solved the AWS sharding recommendations by putting this hash at the front.

```
2bd9189224077b82ec06.app.js
01ea68714303adfaba4f941963ae2b0f.app.css
```
Over time this could mean the S3 bucket gets full of many files and it might not be clear which ones are still in use and which ones aren't. Therefore the bucket has been setup with an expiry policy.

(code)

This deletes any files that are older than one month. The Travis job is then setup to rebuild the application every week, which will redeploy the files still in use and reset their one month clock. This gives the Travis build 4 chances before files are lost so any issues have plenty of time to alert and be fixed.

The Travis build can specify the cache-control headers the files will have when it puts them on S3. For these files it is set to 37556000 seconds (1 year - todo check)

Some files have a fixed name and therefore can't be cached for that long. Therefore there needs to be a separate Travis deploy section to upload these files from a separate `static-low-cache` folder and only specify a 10 minute cache.


### Service worker API Gateway proxy
The service worker has to be on the same domain as the application it is controlling. Therefore it can't be served from static like the other static files. It also cannot change its name as the URL location is rechecked fro updates by the browser.

Therefore, even though the file is available on S3 (https://static.tubealert.co.uk/sw.js) it is not served from there. A new API gateway route is setup to be a proxy to that file on S3. The CloudFormation sets up the appropriate IAM permissions:
(code)
And then it sets up the Resource and Method
(code)

## Deployment pipeline
Travis
`yarn test`
`yarn prod`
`yarn build`
`yarn sw`
`serverless deploy`
`s3 deploy` (long cache)
`s3 deploy` (short cache)

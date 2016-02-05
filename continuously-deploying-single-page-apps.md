# 7 Tips for Continuously Deploying Single Page Apps

Single page applications deliver fantastically rich user experiences and they open up an entirely different avenue for continuous deployment. Separating out a front end application from the server is a sound strategy for breaking up the responsibilities of the team. Maintaining a separate front end code base allows teams to iterate on features quickly and interact through formalized contracts in the form of an API.

Not everything with delivering static assets is so rosy though, there are hosting and delivery pitfalls that your team should be aware of before embarking on continuously deploying static assets. Here are some tips for effectively deploying statically hosted applications iteratively, safely, and most importantly, efficiently.

### 1. Package and Deploy Using State of the Art Tools

If your team has decided to deploy client and server code independently, there is a good chance that the server isn't written in node. That doesn't stop you from using Node and NPM to build and package your application! You're free to use [state of the art tools][wp] for packaging and development, regardless of your server side framework.

Once your build and testing process is independent of a server framework it frees up the delivery process as well. After the front end application passes integration testing the CI server can build a production release (see hint number two) and deliver it directly for distribution (see hint number five).

### 2. Minification, Compression, and Source Maps Are Not Optional

Deploying a single page application means more than uploading concatenated code to a server. It deserves all the byte saving care and attention you would give to the assets served up by a production grade web framework. That means it should be [minified][ug], [compressed][gz], and most certainly includes [source maps][sm]. Any of the popular JavaScript build tools along with a tiny bit of scripting will let you deliver perfectly optimized packages.

### 3. Optimize Code and Style Delivery

This may be slightly controversial given the recent trend toward declaring styles along side view components, but there is a trade off to bundling styles along with code. Typically a browser can download the CSS and JS files in parallel, lowering the time until [first paint after load][fpalt]. That performance boost isn't possible when all of the assets are bundled together.  Instead, all of the styles and code are smashed together in a single large file and clients end up staring at a blank screen while they wait for assets to download.

It complicates the delivery process slightly to have multiple files, but the size and performance benefits are worth the trouble.

### 4. Deliver Separate Bundles

Unless you are an ultra purist every packaged application is composed of both library modules and application code. Chances are that your application code changes much more frequently than library modules. When you serve up a giant concatenated bundle the client is forced to download everything fresh with every minor change, no matter how small it is. Application bundles routinely push a 3MB payload, which a lot of code to download again just because a few lines of application code changed.

To avoid this issue you should separate your application into at least two bundles; one for concatenated library code and another for application code. In the bright future of [HTTP/2 connection parallelism][http2] individual files may be served up in parallel and this sort of planning won't be necessary. For right now, a bit of asset bundle splitting will speed up the experience for your users on every release.

### 5. Get Friendly With a Content Distribution Network

Serve static applications from a content distribution network. This allows clients to keep pointing at the same URL while maintaining caching semantics. It also allows you to perform invalidations when you release code, despite the lack of asset fingerprinting. An invalidation updates the cached version of the application that is held at each edge server, the servers that actually serve the application to clients.

Be warned, invalidations can be slow, taking 10 minutes or more on Amazon [CloudFront][cf]. This unpredictable asynchronous behavior is part of why extra care has to be taken around versioning and releases.

### 6. Continuity Knows No Version

Don't rely on users reloading their browser. Assume that some users will be running older versions of the app, and be prepared to handle requests from deprecated features. Consider releases as a continuum of changes and decide how long your release cycle is. At a certain point it isn't practical to support every old release and the bugs they may have contained. Unless you are deploying to a kiosk with an especially infrequent update cycle, you can safely assume users will reload once a week.

### 7. Roll Features Out Gradually

Use feature flags to roll features out gradually. Ember is a stellar [example of shipping code][eff] with features available, but disabled by default. The code is live and in production, but most people aren't using it. Once it has been vetted in the wild, with staff, or with a fraction of your users you can release a new version with the feature enabled.

The same approach is often used when releasing server side code, but the stakes are higher with statically hosted single page apps. A gradual approach is crucial because rolling code back an only be as fast as your CDN's invalidation period. That means you could have a botched release in production for 10 minutes or more without being able to revoke it.

## Move Quickly With Continuously Deployed Single Page Applications

Deploying single page applications can be as simple and robust as deploying application assets bundled with server code. More so, you gain the power of native JavaScript tooling, regardless of your application framework. At its core, a server/browser relationship is a simple distributed system. By deploying single page applications separately from the server your team gains all of the flexibility, focus, and prioritization of a micro-system architecture.

[wp]: http://webpack.github.io/
[ug]: http://lisperator.net/uglifyjs/
[gz]: https://nodejs.org/api/zlib.html#zlib_zlib
[sm]: http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/
[cf]: https://aws.amazon.com/cloudfront/
[fpalt]: http://www.tuanhuynh.com/blog/2015/measuring-first-paint-time-in-chrome/
[http2]: https://http2.github.io/faq/#what-are-the-key-differences-to-http1x
[eff]: https://guides.emberjs.com/v1.10.0/configuring-ember/feature-flags/

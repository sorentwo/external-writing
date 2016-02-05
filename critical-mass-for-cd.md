# Critical Mass for Continuous Deployment

Does this situation sound familiar?

An engineering team relies on manual testing for quality assurance and
preventing regressions. The manual process is lengthy, prone to errors, and
doesn't handle edge cases, so there are frequent regressions that slip into
production. The deployment pipeline has slowed to a crawl, causing features to
pile up on top of each other. Getting anything into production keeps getting
slower.

It's not all so bad. The team has some tests and automatically runs them with
continuous integration. However, they don't trust the test coverage enough to
allow an automated system to ship into production. But why? What's missing? What
is the minimum level of test coverage necessary to trust automated tools with
continuous deployment? Let's find out.

## The Way is Continuous Deployment

> Continuous Integration refers to integrating, building, and testing code
> within the development environment. Continuous Delivery builds on this,
> dealing with the final stages required for production deployment.
>
> — [Martin Fowler][mfcd]

Leveraging automated testing as much as possible reduces the costs of building
and deploying a system. Those are financial, temporal, and communication costs
being paid by the entire company. Continuous deployment goes a long way toward
achieving these business goals:

* It takes all of the ceremony out of releases. When you release often, the
  iterations are small and focused. That means less can break between releases
  and it's easier to fix problems that appear.
* Releasing often maintains momentum and the appearance of progress.
* The earlier you get features in front of real users the faster you can get
  feedback.

So, what is the minimum amount of testing necessary to be confident in
continuous deployment? In typical engineering fashion, it depends on what your
system does, but there are some guidelines. First, the inescapable smoke test.

## Smoke Test

The bare minimum amount of testing for any integrated system is a smoke test. A
smoke test gives you the confidence needed to let an automated system push your
code out to the entire world. If the phrase "push your code out to the entire
world" takes you aback, you may want to consider expanding the scope of your
tests.

Smoke tests must exercise the entire system end-to-end. If your web app uses a
database, the smoke test must hit the database. If your front-end utilizes
JavaScript heavily,the smoke test must load and execute JavaScript. A smoke test
doesn't need to exercise any features or explore any edge cases, it just needs
to make sure all of the moving parts boot up and work together without
exploding.

Unless your application is purely an API, a smoke test should be run within the
context of a real browser environment. There are subtle differences between
simulated request/response testing and rendering real HTML. A broken link or
incorrect asset reference may not be caught in a simulated test, but within a
browser it will raise an error and fail the build.

## Happy Path Story Tests

In any application there are a few paths through the application that are
essential to the experience. Paths that weave together multiple features, like
"onboarding", or "initial signup" should have an integration test that covers
the happy path.

The happy path is the journey between features where everything works as
expected. It isn't concerned with failure scenarios, only critical features and
the intersections between them. Where smoke testing exercises the boundaries
between layers of an application, happy path testing exercises the boundaries
between features of an application.

An application needs a happy path integration test for every user story that is
critical to the business. For example, an online store would have a story test
that traced a new customer from the point where they added an item to their cart
right on through checkout. The customer may only add one item to checkout, and
their payment method would work on the first try, but you could rest assured
that the complete story was functioning as expected.

## Production-Like Environment

> For us, "end-to-end" means more than just interacting with the system from the
> outside—that might be better called "edge-to-edge" testing. We prefer to have
> the end-to-end tests exercise both the system and the process by which it's
> built and deployed.
>
> [Growing Object Oriented Software Guided By Tests][goos]

Building the final compiled, minified, and compressed production release is a
critical part of the deployment process. Any application doing something
interesting has a lot of moving parts, some of which are never utilized outside
of production mode.

Your testing story should include a production test, in which code is built and
deployed to a production-like environment. That environment may be staging, or
an ad-hoc environment that simply verifies the app can boot. The important part
is that it is running in production mode, with production code, and mimics a
production environment as closely as possible.

Things change in production. The configuration is different, dynamic loading is
disabled, CDNs get involved, and asset minification smashes JavaScript together in
unexpected ways. Those aren't the kind of problems you want to find out about
only after you've shipped to production.

## Integration Testing Enables Continuous Deployment

Note that there hasn't been any mention of unit testing, generative testing,
mutation testing, or any other in-depth developer-centric testing. All of the
components critical to building up a body of tests for continuous deployment are
focused around integration. Other automated testing styles, unit testing in
particular, are tools to help refine your code, explore edge cases, and allow
confident refactoring. Integration testing is about verifying boundaries.

Boundaries are where friction builds up between layers of a system. Classes and
modules will operate perfectly fine in a vacuum, but when you connect them to a
database, a file system, and other disparate components things break. That
places the initial burden of automated testing squarely on integration. Build up
enough momentum in your integration tests and it will sustain itself between
releases. Then and you can begin to trust in continuous deployment.

[mfcd]: http://martinfowler.com/bliki/ContinuousDelivery.html#footnote-when
[goos]: http://www.growing-object-oriented-software.com/

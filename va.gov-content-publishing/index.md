# VA.gov Content Publishing

## Memory consumption

## Performance
_February, 2021_

VA.gov is a static website, meaning that to publish new code or new content, a _build_ is issued. The result of that build is a directory of static files like HTML, which is then uploaded to our host to become va.gov. This process as a whole is generally referred to as the front-end deployment. Once done, the website's general content has no downtime and pages are served super quickly.

## What was the problem?
The process behind a front-end deployment is slow. The major bottleneck was when we fetched data out of our CMS (Drupal, in our case). That alone regularly took upwards of 20 minutes, sometimes timing out entirely. If the website got just a little bigger, that request for data would certainly time out and we wouldn't be able to publish new work.  

## What were the goals?
To fetch data from the CMS, we issued a single, giant GraphQL query. It asks for every page of every page-type and every website component (menus, etc) all at once, like a SQL query that says "give me everything from every table." __If we break this up into a bunch of small GraphQL queries, we hypothesized we'll see a performance improvement__.

## What did we do about it?
We broke apart that giant GraphQL query into a whole series of GraphQL queries, with each type of page having its own, dedicated request. Example - _"give me all pages of type X"_, _"give me all pages of type Y"_, _"give me all pages of type Z..."_ We introduced the idea in the form of a [pull request](https://github.com/department-of-veterans-affairs/vets-website/pull/15974).

## What happened?
We expected to see results that offered a short term gain, but instead we saw dramatic improvement - we measured it at __93.9% faster__ and effectively eliminated our concerns around website scalability. __At 1720 pages, we saw response times at about 49 seconds__.  Our VA Medical Center team confidently published hundreds more pages in the last week. There are also many more clear opportunities to get this process faster even as the website gets bigger.

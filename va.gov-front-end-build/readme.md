_03/2021_

# Improvements to the the VA.gov front-end build

VA.gov is a _static website_, meaning that when a user navigates to a page on the website, the files for that page are served as-is from the web server without any dynamic behavior or transformations applied. To publish new versions of the website with updated code or content, a processed referred to as a _front-end build_ is executed. The result of a front-end build is a directory of static files like HTML, which is then uploaded onto the VA.gov web server to become the new version of the website. This process as a whole — from the start of the front-end build to the new files being served from the web server — is generally referred to as a _front-end deployment_. Once done, the website's general content has no downtime and pages are served very quickly. However, the process of the front-end build itself is very complex and faced some important challenges.

1. [Front-end builds used too much memory](#Front-end-builds-used-too-much-memory)
2. [Front-end builds were too slow](#Front-end-builds-were-too-slow)

## Front-end builds used too much memory
_January, 2021_

Engineers regularly execute front-end builds from their local workstations as part of the workflow for creating and editing website templates. If the process were too memory intensive, engineers would not be able to work as productively, as they may find the process is too unstable or that their workstations simply do not contain enough memory to execute it.

### What was the problem?
By January 2021, the website had grown so substantially that the front-end build process now required nearly 7 gigabytes to execute. This was a major pain point facing teams and an imminently pressing issue, as the website was projected to grow exponentially throughout year including the first quarter. Engineers struggled greatly in their local workflows as the front-end build process crashed due to memory exceptions, and we were concerned we would start encountering the same bottlenecks in our infrastructure as well.

### What were the goals?
There are a number of sub-operations (steps) performed during the overall process of a front-end build, with each one lasting for various stretches of time and requiring various amounts of memory. I was able to identify which of these steps were the most memory-intensive by analyzing a summary of these operations outputted into my terminal after running a front-end build on my local machine with a special `verbose` command argument.

The results of the analyis were clear - the step labelled `Parse HTML files` increased the process's memory consumption by over 5 gigabytes, so this was the step of interest. My goal was simple - refactor this build step to use less memory.

### What did we do about it?
The source code behind the memory-intensive build step was brief. It was not hard to understand why it was so memory-intensive. Each build step has access to every file of the website with the contents of each file available as a plain string (text) value. However, some build steps perform advanced file manipulations that require the plain string value of `.html` files parsed into a virtual DOM tree, so that the HTML file can be manipulated in sophisticated ways. The `Parse HTML files` build step parsed a virtual DOM tree from every `.html` file, which subsequent build steps could then reliably access (an example of one of these build steps is one that automatically generates and adds element-IDs onto heading tags.) This means that the combined weight of every HTML file's virtual DOM was stored in memory at once.

The concept behind my approach was simple - since it uses too much memory to have every file's virtual DOM in memory at once, only store one at a time. Simply parse the virtual DOM of a single file, operate only on it, delete its virtual DOM and then repeat with the next file. This would also require refactoring subsequent build steps that required  each file's virtual DOM, but it would mean only one virtual DOM would be stored in memory at a time.

I opened a pull request proposing the changes to my direct team members and those on the Platform team.

### What happened?
The results were dramatic. Before my changes, peak memory use was measured at 6255.67mB - over 6 gigabytes. After my changes, memory peaked at only 819.76mB - less than 1 gigabyte. __This means that we were able to reduce memory consumption by 87%, completing eliminating the issue of memory intensiveness from the front-end build.__

- https://github.com/department-of-veterans-affairs/vets-website/pull/15601
- [PDF of pull request 15601](./files/15601.pdf)
- [PDF of files changed in pull request 15601](./files/15601_files-changed.pdf) (_Note - because of some files being moved into a new directory as well as being slightly modified, the diff portrays many more lines changed than there really were. The main concept of the changes is shown in `src/site/stages/build/plugins/modify-dom/index.js`._)

## Front-end builds were too slow
_February, 2021_

The duration of a front-end build is very important because the longer it takes, the longer a content editor has to wait in order to see their article changes (or other update to the website content) reflected on the website.

A quicker front-end build also results in quicker front-end deployments, meaning teams can respond to and recover more quickly in the event of an incident


## What was the problem?
The process behind a front-end deployment is slow. The major bottleneck was when we fetched data out of our CMS (Drupal, in our case). That alone regularly took upwards of 20 minutes, sometimes timing out entirely. If the website got just a little bigger, that request for data would certainly time out and we wouldn't be able to publish new work.

## What were the goals?
To fetch data from the CMS, we issued a single, giant GraphQL query. It asks for every page of every page-type and every website component (menus, etc) all at once, like a SQL query that says "give me everything from every table." __If we break this up into a bunch of small GraphQL queries, we hypothesized we'll see a performance improvement__.

## What did we do about it?
We broke apart that giant GraphQL query into a whole series of GraphQL queries, with each type of page having its own, dedicated request. Example - _"give me all pages of type X"_, _"give me all pages of type Y"_, _"give me all pages of type Z..."_ We introduced the idea in the form of a [pull request](https://github.com/department-of-veterans-affairs/vets-website/pull/15974).

## What happened?
We expected to see results that offered a short term gain, but instead we saw dramatic improvement - we measured it at __93.9% faster__ and effectively eliminated our concerns around website scalability. __At 1720 pages, we saw response times at about 49 seconds__.  Our VA Medical Center team confidently published hundreds more pages in the last week. There are also many more clear opportunities to get this process faster even as the website gets bigger.

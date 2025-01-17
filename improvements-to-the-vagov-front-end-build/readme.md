_03/2021_

# Improvements to the VA.gov front-end build

VA.gov is a _static website_, meaning that when a user navigates to a page on the website, the files for that page are served as-is from the web server without any dynamic behavior or transformations applied. To publish new versions of the website with updated code or content, a process referred to as a _front-end build_ is executed. The result of a front-end build is a directory of static files like HTML, which is then uploaded onto the VA.gov web server to become the new version of the website. This process as a whole — from the start of the front-end build to the new files being served from the web server — is generally referred to as a _front-end deployment_. Once done, the website's general content has no downtime and pages are served very quickly. However, the process of the front-end build itself is very complex and teams were faced with some important challenges in scaling it.

1. [Front-end builds used too much memory](#Front-end-builds-used-too-much-memory)
2. [Front-end builds were too slow](#Front-end-builds-were-too-slow)

## Front-end builds used too much memory
_January, 2021_

Engineers regularly execute front-end builds from their local workstations as part of the workflow for creating and editing website templates. If the process were too memory intensive, engineers would not be able to work as productively, as they may find the process is too unstable or that their workstations simply do not contain enough memory to execute it.

### What was the problem?
__By January 2021, the website had grown so substantially that the front-end build process now required nearly 7 gigabytes to execute.__ This was a major pain point facing teams and an imminently pressing issue, as the website was projected to grow exponentially throughout year including the first quarter. Engineers struggled in their local workflows as the front-end build process crashed due to memory exceptions, and we were concerned we would start encountering the same bottlenecks in our infrastructure as well. If the site grew any larger, engineers would either need computers with more memory or would have to start developing workarounds in order to run front-end builds on their machines.

### What were the goals?
There are a number of sub-operations (steps) performed during the process of a front-end build, with each one lasting for various stretches of time and requiring various amounts of memory. I was able to identify which of these steps were the most memory-intensive by analyzing a summary of these operations outputted into my terminal after running a front-end build on my local machine with a special `verbose` command argument.

The results of the analyis were clear - the step labelled `Parse HTML files` increased the process's memory consumption by over 5 gigabytes, so this was the step of interest. My goal was simple - refactor this build step to use less memory.

<details><summary>Output indicating memory issue</summary>

![Screenshot of terminal output indicating heavy memory consumption](./files/memory-bottleneck-output.png)

</details>

### What did we do about it?
The source code behind the memory-intensive build step was brief. It was not hard to understand why it was so memory-intensive. Each build step has access to every file of the website with the contents of each file available as a plain string (text) value. However, some build steps perform advanced file manipulations that require the plain string value of `.html` files parsed into a virtual DOM tree, so that the HTML file can be manipulated in sophisticated ways. The `Parse HTML files` build step parsed a virtual DOM tree from every `.html` file, which subsequent build steps could then reliably access (an example of one of these build steps is one that automatically generates and adds element-IDs onto heading tags.) This means that the combined weight of every HTML file's virtual DOM was stored in memory at once.

<details><summary>Code snippet of build step</summary>

The code of interest iterates through all of the files and uses `Cheerio` (an NPM package) to parse a virtual DOM from every HTML file, binding the result of each operation to the original file object. The combined result is stored in memory all at once.

![Screenshot of problematic code](./files/memory-bottleneck-code.png)
</details>

The concept behind my approach for improving it was simple - __since it uses too much memory to store every file's virtual DOM in memory at once, only store one at a time__. Parse the virtual DOM of a single file, operate only on it, delete its virtual DOM and then repeat with the next file. This would also require refactoring subsequent build steps that required each file's virtual DOM, but it would mean only one virtual DOM would be stored in memory at a time.

I opened a pull request proposing the changes to my direct team members and those on the Platform team.

<details><summary>Code snippet after changes</summary>

The updated code below shows a new pattern, where each file's virtual DOM is deleted from memory before the next file's virtual DOM is parsed.

![Screenshot of improved code](./files/memory-bottleneck-fixed.png)

</details>

### What happened?
The results were dramatic. Before my changes, peak memory use was measured at `6255.67mB` - over 6 gigabytes. After my changes, memory peaked at only `819.76mB` - less than 1 gigabyte. __This means that we were able to reduce memory consumption by 87%, completing eliminating the issue of memory intensiveness from the front-end build.__

<details><summary>Output indicating memory improvement</summary>

![Screenshot of terminal output showing an 87% drop in memory consumption](./files/memory-bottleneck-output-fixed.png)

</details>

### Links
- https://github.com/department-of-veterans-affairs/vets-website/pull/15601
- [PDF of pull request 15601](./files/15601.pdf)
- [PDF of files changed in pull request 15601](./files/15601_files-changed.pdf)
  - _Note: Because of some files being moved into a new directory as well as being slightly modified, the diff portrays many more lines changed than there really were. The main concept of the changes is shown in `src/site/stages/build/plugins/modify-dom/index.js`._

## Front-end builds were too slow
_February, 2021_

The performance of a front-end build is very important because the longer it takes, the longer a content editor has to wait in order to see their article changes (or other update to the website content) reflected on the website. As more editors were onboarded and granted access to the VA.gov content management system (CMS), there became a large emphasis on the speed of a front-end build, because many of the new editors were accustomed to the typical experience of a CMS where a page request is served using data pulled from the database at the time of the request. As a static website, the architecture of VA.gov was quite different, but we hoped to satisy the editors through a rapid publishing process powered by a fast front-end build.

### What was the problem?
To say that the front-end build was not fast enough to satisfy editors would be an understatement - __the front-end build was critically slow__. The major bottleneck occurred during a build step that fetched articles and other forms of content data from the VA.gov CMS. That build step alone regularly took upwards of 20 minutes, sometimes timing out entirely, causing the whole process to completely fail. With the current state of the front-end build, the website could no longer grow, and the roadmap required that the website grow exponentially starting as soon as possible.

### What were the goals?
The build step that fetched data from the CMS did so by issuing a single large GraphQL query structured to ask for every article of every article-type as well as every website component (menus, etc) all at once, like a SQL query that says _give me everything from every table_.

Although other teams operated on the assumption that the CMS's GraphQL API itself was the bottleneck and intended to replace it entirely with a new method of fetching data, I hypothesized that the GraphQL API was fine, and that it was how the query was written that was the issue.

<details><summary>Snapshot of the monolithic GraphQL query</summary>

The snippet below of the monolithic GraphQL query illustrates how the various types of articles converge into a single type of query called a `nodeQuery`. The actual full-length query is many thousands of lines.

![A code snippet of the GraphQL query](./files/graphql-monolithic-query.png)

</details>

__My goal became to optimize this GraphQL query by unstitching it into many smaller queries and then executing them individually in order to achieve a significant performance boost in a short timeframe.__

### What did we do about it?
Through deep analysis of the monolithic GraphQL query, I determined and implemented a strategy of unstitching the monolithic query into a collection of smaller queries. This was relatively straightforward - instead of asking for everything at once, ask for each type of article in its own request. Then, I organized the entire set of small queries into a collection.

<details><summary>Refactored GraphQL pattern</summary>

The code snippet below illustrates how the `office` article-type is wrapped into its own `nodeQuery` so that it can be executed standalone.

![Code snippet of the query for article-type "office"](./files/graphql-refactored-query.png)

The code snippet below illstrates the collection of GraphQL queries for fetching all types of articles. You can see the GraphQL query from the previous example included in the collection under the name `GetNodeOffices`.

![Code snippet showing the collection of GraphQL queries](./files/graphql-refactored-query-list.png)

</details>

Next, I added logic for executing the entire entire collection using 15 parallel HTTP requests at a time. The response data of each request is merged into a single shared data set. Once all requests have finished, the data set should match that of the original monolithic GraphQL query, which I could ensure through thorough unit testing.

After introducing the concept to a team member on the CMS program, things became even more exciting as we brainstormed ideas for going further. We observed some types of articles had many instances, so it wasn't efficient to request them all at once. After some collaborating, I added a mechanism for slicing up these large data sets, so that if there were many articles of a certain type, we could segment that data into small lists, then request all of the segments in parallel. We speculated that this method may allow us to take advantage of some interesting caching behavior we had observed with the CMS's GraphQL API, where sometimes the monolithic GraphQL query received the response data within seconds. My team member on the CMS program began calculating metrics and upgrading their infrastructure such as their database instance to handle the additional request volume.

I also added some interesting terminal output, so we could observe which queries were the most expensive or contained large data sets.

<details><summary>Output of refactored GraphQL process</summary>

The output is formatted into a Markdown table so it could easily be copied into a GitHub comment. The table contains columns for the name of the GraphQL query, response time in seconds, and the number of pages included in the response data (if applicable, as some queries returned data other than a list of pages.) The image below shows the output as visible in the Jenkins deployment job.

![Screenshot of terminal output](./files/jenkins-graphql-output.png)

</details>

I opened a pull request introducing the refactored GraphQL approach along with some preliminary results to my direct team members and those on the Platform team. The changes were also backwards-compatible, meaning that surrounding code areas continued to function the same way, so the code changes were ultimately safe to merge.

### What happened?
__We saw dramatic improvement that far exceeded our expectations.__ The upgraded GraphQL (as it became to be known) demonstrated exponentially better performance. Measuring this was difficult because there were many variables such as the caching mechanism of the GraphQL API, which was central to our strategy with the upgraded GraphQL queries. However, in our performance summary during the early stages of the GraphQL upgrade, __we concluded the performance improvement at 93.9% faster than the monolithic GraphQL query__. With the website at its then-current size of 1,720 pages, we saw response times at about __49 seconds__.

<details><summary>Performance comparison of early results</summary>

In the chart below, the vertical axis indicates response times in seconds, while the horizontal axis indicates numbers of nodes. A _node_ can be considered a block of content used to compose an article. A single article may be composed of one or many nodes.The blue and red lines represent the performance of the monolithic GraphQL query, while the yellow and green lines represent the upgraded GraphQL query.

![line chart comparing the two query strategies](./files/graphql-comparison-chart.png)

The monolithic GraphQL query could fetch 4,467 nodes in roughly 2,057 seconds. The request would consistently time out beyond 5,000 nodes. The upgraded GraphQL approach, however, not just eliminated fears of timeouts but could fetch 5,561 nodes in only 177 seconds.

</details>

In the following month, we made even deeper improvements by addressing some of the more costly queries as observed in the build output. We also experimented with parallelization to find the most optimal number of GraphQL requests that could be run simultaneously.

<details><summary>Performance comparison after further optimizations</summary>

In the chart below, the vertical axis indicates response times in seconds, while the horizontal axis indicates numbers of nodes. A _node_ can be considered a block of content used to compose an article. A single article may be composed of one or many nodes. The blue and red lines represent the performance of the monolithic GraphQL query, while the yellow and green lines represent the upgraded GraphQL query.

![line chart comparing the two query strategies plus parallelization](./files/graphql-comparison-chart-2.JPG)

These results are __without any server-side caching__, so they indicate worst case scenarios. The performance in practice should be much better.

</details>

After these deeper improvements, teams concluded an approximate __96% increase in processing time or 30X increase in throughput__.

<details><summary>Improvements to deployment times post-launch</summary>

The improvement was very evident in the Jenkins job for front-build deployments. The GraphQL upgrade is reflected in the `Build` column starting on February 15th.

![screenshot of Jenkins dashboard showing build times drop from upwards of 15 minutes to about 2 minutes](./files/jenkins-content-build.png)

</details>

Our VA Medical Center team confidently published hundreds more pages the following week and continues to do so as of writing. Although there is still much more work to do as the website grows, we concluded that the __GraphQL upgrade effectively eliminated our concerns of website scalability for the foreseeable future__.

### Links
- https://github.com/department-of-veterans-affairs/vets-website/pull/15974
- [PDF of pull request 15974](./files/15974.pdf)
- [PDF of files changed in pull request 15974](./files/15974_files-changed.pdf)

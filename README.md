# GitHub crawler

## Why can it be useful ?

With the current move to microservices, it's not rare that a team who previously had a couple of repositories, now has several dozens. 
Keeping a minimum of consistency between the repositories becomes a challenge which may cause risks : 
- have we updated all our repositories so that they use the latest Docker image ?
- have we set up the proper security config in all our repositories ?
- which versions of the library X are we using across ?
- are we using a library that we are not supposed to use anymore ?
- which team is owner of a repository ? 

These are all simple questions that sometimes take hours to answer, with always the risk of missing one repository in the analysis, making the answer inaccurate.

Github crawler aims at automating the information gathering, by crawling an organization's repositories through GitHub API. **Even if your organization has hundreds of repositories,
Github crawler will be able to report very useful information in few seconds !** 

## How does it work ?

Github crawler is a Spring Boot command line application. It is written in Java and Kotlin, the target being to move as much as possible to Kotlin.

Following a simple configuration, it will use Github API starting from a given organization level, then for each public repository, will look for patterns in specified files. 

You can easily exclude repositories from the analysis, configure the files and patterns you're interested in. If you have several types of repositories (front-end, back-end, config repositories for instance), you can have separate configuration files so that the information retrieved is relevant to each scope of analysis.

Several output types are available in [this package](./src/main/kotlin/com/societegenerale/githubcrawler/output/) :
- console is the default and will be used if no output is configured
- a simple "raw" file output
- a CI-droid ready CSV file output, to run locally and copy/paste in CI-droid bulk update UI (UI implementation is in progress)
- HTTP output, which enables you to POST the results to an endpoint like ElasticSearch, for easy analysis in Kibana



## Configuration on crawler side 

```yaml
    # the base GitHub URL to crawl
    gitHub.url: https://github.com
    
    # the name of the GitHub organization to crawl. To fetch the repositories, the crawler will hit 
    # https://${gitHub.url}/api/v3/orgs/${organizationName}/repos
    organizationName: MyOrganization
    
    #repositories matching one of the configured regexp will be excluded
    repositoriesToExclude:
      # exclude the ones that start with "financing-platform-" and end with "-run"
      - "^financing-platform-.*-run$"
      # exclude the ones that DON'T start with "financing-platform-" 
      - "^(?!financing-platform-.*$).*"
    
    # do you want the excluded repositories to be written in output ? (default is false)
    # even if they won't have any indicators attached, it can be useful to output excluded repositories, 
    # especially at beginning, to make sure you're not missing any
    publishExcludedRepositories: true
    
    # by default, we'll crawl only the repositories' default branch. But in some cases, you may want to crawl all branches
    crawlAllBranches: true
    
    # default output is console - it will be configured automatically if no output is defined
    # the crawler takes a list of output, so you can configure several
    output:
      file:
      # we'll output one repository branch per line, in a file named ${filenamePrefix}_yyyyMMdd_hhmmss.txt
       filenamePrefix: "orgaCheckupOutput"
      http:
        # we'll POST one repository branch individually to ${targetUrl}
        targetUrl: "http://someElasticSearchServer:9201/technologymap/MyOrganization"
      ciDroidReadyFile:
        # this should be an indicator defined in indicatorsToFetchByFile section. can be a comma separated list if several to output
        indicatorsToOutput: "dockerFilePath"  
     
    # list the files to crawl for, and the patterns to look for in each file         
    indicatorsToFetchByFile:
    # filename - crawler will do a GET https://${gitHub.url}/raw/${organizationName}/${repositoryName}/${branchName}/${filename}
    # use syntax with "[....]" to escape the dot in the file name (configuration can't be parsed otherwise, as "." is a meaningful character in yaml files)
      "[pom.xml]":
          # name of the indicator that will be reported for that repository in the output
          - name: spring_boot_starter_parent_version
            # name of the method to find the value in the file, pointing to one of the implementation classes of FileContentParser
            method: findDependencyVersionInXml
            # the parameters to the method, specific to each method type
            params:
              # findDependencyVersionInXml needs an artifactId as a parameter : it will find the version for that Maven artifact by doing a SAX parsing, even if the version is a ${variable} defined in <properties> section
              artifactId: spring-boot-starter-parent
          - name: spring_boot_dependencies_version
            method: findDependencyVersionInXml
            params:
              artifactId: spring-boot-dependencies
      #another file to parse..
      Jenkinsfile:
          - name: build_helper_package
            method: findFirstValueWithRegexpCapture
            params:
              # findFirstValueWithRegexpCapture needs a pattern as a parameter. The pattern needs to contain a group capture (see https://regexone.com/lesson/capturing_groups) 
              # the first match will be returned as the value for this indicator
              pattern: ".*com\\.a\\.given\\.package\\.([a-z]*)\\.BuildHelpers.*"

## Configuration on repository side 

While the global configuration is defined along with github crawler, we have the possibility to override it at the repository level. 
Repository level config is stored in a **.githubCrawler** file, at the root of the repository in the default branch

- **Exclusion** 

if a repository should be excluded, we can define it in the repository itself. if .githubCrawler contains : 

```yaml
    excluded: true  
```

Then the crawler will consider the repository as excluded, even if it doesn't match any of the exclusion pattern in the crawler config 

- **Redirecting to a specific file to parse**

Sometimes, the file we're interested in parsing is not in a standard location like the root of the repository - this is typically the case for Dockerfile.

What we can do in this case is define the file in the crawler config, and override the path in the repository config, with the redirectTo attribute : 

```yaml
    filesToParse: 
      - 
        name: Dockerfile
        redirectTo: routing/Dockerfile 
```

With above config, when the crawler tries to fetch Dockerfile at the root of the repository, it will actually try to parse *routing/Dockerfile*

- **Tagging a repo**

It may happen that you want to "tag" some repos, to be able to filter easily on them when browsing the results. This is made possible by adding below config in the .githubCrawler file : 

```yaml
    tags:
      - "someTag"
      - "someOtherTag"
```

in output, this information will be attached to all the repositories for which it has been configured. 


## Output

when running the crawler with above config and using HTTP output to push indicators in ElasticSearch, this is the kind of data you'll get 

- different values for the same indicator, fetched by findFirstValueWithRegexpCapture method : 

![](/docs/images/kibanaOutput1.png)

- different values for the same indicator, fetched by findDependencyVersionInXml method : 

![](/docs/images/kibanaOutput2.png)

(when there's no value, it means the file was not found. when the value is "not found", it means the file exists, but the value was not found in it)

- when using _crawlAllBranches: true_ property , branch name is shown : 

![](/docs/images/kibanaOutput_severalBranches.png)

Once you have this data, you can quickly do any dashboard you want ! 

## Test strategy

We follow a strict test driven strategy for the implementation. Contributions are welcome, but you'll need to submit decent tests along with your changes for them to be accepted. 
Browse the tests to get an idea of what level of test is expected.

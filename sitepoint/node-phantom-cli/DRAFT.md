
# Crawling on Command Using Node.js

It is quite common during the course of a project to find yourself needing to write custom scripts for performing a variety of actions. Such one-off scripts, which are typically triggered via the command-line ([CLI](https://en.wikipedia.org/wiki/Command-line_interface)), can be used for virtually any type of task. Having written many such scripts over the years, I have grown to appreciate the value of taking a small amount of time upfront to put in place a custom CLI [microframework](https://en.wikipedia.org/wiki/Microframework) to facilitate this process. Fortunately, [Node.js](https://nodejs.org/) and its extensive package ecosystem, [NPM](https://www.npmjs.com/), make it easy to do just that. Whether parsing a text file or running an [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load), having a convention in place makes it easy to add new functionality in an efficient and structured way.

While not typically associated with the command-line, the crawling of websites is gaining relevance for various reasons, such as automated functional testing and intelligence gathering. To that end, this tutorial demonstrates how to implement a lightweight, general-purpose CLI framework in the context of web crawling. Hopefully, this will get your creative juices flowing, whether your interest be specific to crawling or to the command-line. Technologies covered include Node.js, [PhantomJS](http://phantomjs.org/), and an assortment of NPM packages related to both crawling and the CLI. 

The source code for this tutorial can be found on [GitHub](https://github.com/wreckloose/node-phantom-cli). In order to run the examples, you will need to have both Node.js and PhantomJS installed, which is outside the scope of this article.


### Setting up a Basic Command-line Framework

At the heart of any CLI framework is the concept of converting a command, which typically includes one or more optional or required arguments, into a concrete action. Two NPM packages that are quite helpful in this regard are [commander](https://www.npmjs.com/package/commander) and [prompt](https://www.npmjs.com/package/prompt). 

Commander allows you to define what arguments may or may not be present, while prompt allows you to (appropriately enough) prompt the user for input at runtime. The end result is a [syntactically sweet](https://en.wikipedia.org/wiki/Syntactic_sugar) interface for performing a variety of actions with dynamic behaviors based on some user-supplied data.

Say, for example, we want our command to look like this:
```
$ node run.js -x hello_world
```

Our entry point, [run.js](https://github.com/wreckloose/node-phantom-cli/blob/master/run.js), defines the possible arguments like this:
```
program
  .version('1.0.0')
  .option('-x --action-to-perform [string]', 'The type of action to perform.')
  .option('-u --url [string]', 'Optional url used by certain actions')
  .parse(process.argv);
```

and defines the various user input cases like this:
```
var performAction = require('./actions/' + program.actionToPerform)

switch (program.actionToPerform) {
  case 'hello_world':
    prompt.get([{
      name: 'url',                                      // What the property name should be in the result object
      description: 'Enter a URL',                       // The prompt message shown to the user
      required: true,                                   // Whether or not the user is required to enter a value
      conform: function (value) {                       // Validates the user input
        return isValidUrl(value);                       // In this case, the user must enter a valid url
      }
    }], function (err, result) {                        // Perform some action following successful input
      performAction(phantomInstance, result.url);
    });
    break;
}
```

At this point, we've defined a basic path through which we can specify an action to perform, and have added a prompt to accept a url. We just need to add a module to handle the logic that is specific to this action. We can do this by adding a file named [hello_world.js](https://github.com/wreckloose/node-phantom-cli/blob/master/actions/hello_world.js) to the [actions](https://github.com/wreckloose/node-phantom-cli/tree/master/actions) directory:

```
'use strict';

/**
 * @param Horseman phantomInstance
 * @param string url
 */
module.exports = function (phantomInstance, url) {

  if (!url || typeof url !== 'string') {
    throw 'You must specify a url to ping';
  } else {
    console.log('Pinging url: ', url);
  }

  phantomInstance
    .open(url)
    .status()
    .then(function (statusCode) {
      if (Number(statusCode) >= 400) {
        throw 'Page failed with status: ' + statusCode;
      } else {
        console.log('Hello world. Status code returned: ', statusCode);
      }
    })
    .catch(function (err) {
      console.log('Error: ', err);
    })
    .close(); // Always close the Horseman instance, or you might end up with orphaned phantom processes
};
```

As you can see, the module expects to be supplied with an instance of a PhantomJS object and a url. We will get into the specifics of defining a PhantomJS instance momentarily, but for now, it's enough to see that we've laid the groundwork for triggering a particular action. Now that the convention is in place, we can more efficiently add new actions, and do so in a sane manner.

### Crawling with PhantomJS Using Horseman

[Horseman](https://github.com/johntitus/node-horseman) is a Node.js package that provides a powerful interface for creating and interacting with PhantomJS processes. A comprehensive explanation of Horseman and its features would warrant its own article, but suffice it to say that it allows you to easily simulate just about any behavior that a human user might exhibit, including mouse and keyboard interactions, as well as cookie handling. Horseman provides a wide range of configuration options, including things like automatically injecting [jQuery](https://jquery.com/) and ignoring SSL certificate warnings.

Each time we trigger an action through our CLI framework, our entry script (run.js) instantiates an instance of Horseman and passes it along to the specified action module. In pseudo-code it looks something like this:
```
var phantomInstance = new Horseman({
  phantomPath: '/usr/local/bin/phantomjs',
  loadImages: true,
  injectJquery: true,
  webSecurity: true,
  ignoreSSLErrors: true
});

performAction(phantomInstance, ...);
```

Now when we run our command, the Horseman instance and input url get passed to the [hello_world](https://github.com/wreckloose/node-phantom-cli/blob/master/actions/hello_world.js) module, causing PhantomJS to request the url, capture its status code, and print the status to the console. We've just run our first bona fide "crawl" using Horseman. Giddyup!



### Chaining Horseman Methods for Complex Interactions

So far we've looked at a very simple usage of Horseman, but the package can do much more. We can chain Horseman's methods to perform a sequence of actions, such as triggering mouse clicks, entering text, and submitting forms. 

In order to demonstrate a few of these features, let's define an action that simulates a user navigating through GitHub and creating a new repository, which will be triggered like so:

```
$ node run.js -x create_repo
```

Following the convention of the CLI framework we've already put in place, we need to add a new module to the actions directory named [create_repo.js](https://github.com/wreckloose/node-phantom-cli/blob/master/actions/create_repo.js). As with our "hello world" example, the create_repo module exports a single function containing all of the logic for that action.

```
module.exports = function (phantomInstance, username, password, repository) {

  if (!username || !password || !repository) {
    throw 'You must specify login credentials and a repository name';
  }

  ...
}
```

Notice that with this action we are passing more parameters to the exported function than we did previously. The parameters include username, password, and repository. We will pass these values from run.js once the user has successfully completed the prompt challenge. 


Before any of that can happen though, we have to add logic to run.js to trigger the prompt and capture the data. We do this by adding a case to our main switch statement:

```
switch (program.actionToPerform) {

    case 'create_repo':
      prompt.get([{
          name: 'repository',
          description: 'Enter repository name',
          required: true
      }, {
          name: 'username',
          description: 'Enter GitHub username',
          required: true
        }, {
          name: 'password',
          description: 'Enter GitHub password',
          hidden: true,
          required: true
      }], function (err, result) {
        performAction(phantomInstance, result.username, result.password, result.repository);
      });
      break;

    ...
```

Now that we've added this hook to run.js, when the user enters the relevant data, it gets passed to the action, which allows us to proceed with the crawl.

As for the "create repo" crawl logic itself, we use Horseman's array of methods to open the GitHub login url, enter the username and password, and submit the form:

```
phantomInstance
    .open('https://github.com/login')
    .type('input[name="login"]', username)
    .type('input[name="password"]', password)
    .click('input[name="commit"]')

```

We continue the chain by waiting for the form submission page to load:

```
    .waitForNextPage()
```

after which we use jQuery to determine if the login was successful:

```
    .evaluate(function () {
      $ = window.$ || window.jQuery;
      var fullHtml = $('body').html();
      return !fullHtml.match(/Incorrect username or password/);
    })
    .then(function (isLoggedIn) {
      if (!isLoggedIn) {
        throw 'Login failed';
      }
    })
```

An error is thrown if the login fails. Otherwise, we continue chaining methods to navigate to our profile page:

```
    .click('a:contains("Your profile")')
    .waitForNextPage()
```

Once we're on our profile page, we navigate to our repositories tab:

```
    .click('nav[role="navigation"] a:nth-child(2)')
    .waitForSelector('a.new-repo')
```

While on our repositories tab, we check to see if a repository with the specified name already exists. If it does, then we throw an error. If not, then we continue with our sequence:

```
    // Gather the names of the user's existing repositories
    .evaluate(function () {
      $ = window.$ || window.jQuery;

      var possibleRepositories = [];
      $('.repo-list-item h3 a').each(function (i, el) {
        possibleRepositories.push($(el).text().replace(/^\s+/, ''));
      });

      return possibleRepositories;
    })

    // Determine if the specified repository already exists
    .then(function (possibleRepositories) {
      if (possibleRepositories.indexOf(repository) > -1) {
        throw 'Repository already exists: ' + repository;
      }
    })
```

Assuming no errors have been thrown, we proceed by programmatically clicking the "new repository" button and waiting for the next page: 

```
    .click('a.new-repo')
    .waitForNextPage()
```

after which we enter the specified repository name and submit the form:
```
    .type('input#repository_name', repository)
    .click('button:contains("Create repository")')
```

Once we reach the resulting page, then we know that the repository has been created:
```
    .waitForNextPage()
    .then(function () {
      console.log('Success! You should now have a new repository at: ', 'https://github.com/' + username + '/' + repository);
    })
```

As with any Horseman crawl sequence, it is crucial that we close the Horseman instance at the end:

```
    .close();
```

If we fail to close the Horseman instance, then we can end up with orphaned PhantomJS processes persisting on our machine.


### Crawling to Gather Data

At this point, we have assembled a static sequence of actions to accomplish a particular goal: programmatically creating a new repository on GitHub. In order to do this, we have chained a series of discrete Horseman methods together. This approach can be useful for specific structural and behavioral patterns that are known beforehand, however, you may find that you have to implement more flexible scripting, when the path of execution has the potential to vary widely based on context-specific logic. Perhaps the site you're crawling has complicated logic forks, with many possible outcomes. Or perhaps you're just wanting to extract data from the DOM, which uses the same technique.

In such cases, you can use Horseman's evaluate() method, which allows you to execute free-form interactions in the browser, injecting any desired JavaScript, either inline or as an external script.

This section demonstrates an example of extracting basic data (links, in this case) from a page, but note that you could execute ANY JavaScript within the evaluate() method, and exfiltrate any type or amount of data.

As with our last example, we have to first add a new module to the actions directory:

```
module.exports = function (phantomInstance, url) {

  if (!url || typeof url !== 'string') {
    throw 'You must specify a url to gather links';
  }

  phantomInstance
    .open(url)

    // Interact with the page. This code is run in the browser.
    .evaluate(function () {
      $ = window.$ || window.jQuery;
      
      // Return a single result object with properties for 
      // whatever intelligence you want to derive from the page
      var result = {
        links: []
      };

      if ($) {
        $('a').each(function (i, el) {
          var href = $(el).attr('href');
          if (href) {
            if (!href.match(/^(#|javascript|mailto)/) && result.links.indexOf(href) === -1) {
              result.links.push(href);
            }
          }
        });
      }
      // jQuery should be present, but if it's not, then collect the links using pure javascript
      else {
        var links = document.getElementsByTagName('a');
        for (var i = 0; i < links.length; i++) {
          var href = links[i].href;
          if (href) {
            if (!href.match(/^(#|javascript|mailto)/) && result.links.indexOf(href) === -1) {
              result.links.push(href);
            }
          }
        }
      }

      return result;
    })
    .then(function (result) {
      console.log('Success! Here are the derived links: \n', result.links);
    })

    .catch(function (err) {
      console.log('Error getting links: ', err);
    })

    // Always close the Horseman instance, or you might end up with orphaned phantom processes
    .close();
```

And also, add a hook for the new action in run.js:

```
switch (program.actionToPerform) {

    ...

    case 'get_links':
      prompt.get([{
          name: 'url',
          description: 'Enter URL to gather links from',
          required: true,
          conform: function (value) {
            return validUrl.isWebUri(value);
          }
      }], function (err, result) {
        performAction(phantomInstance, result.url);
      });
      break;
```

Now that this code is in place, we can run a crawl to extract links from any given page by running the following command:
```
$ node run.js -x get_links
```

This action demonstrates extracting data from a page, and does not utilize any browser actions that are built-in by Horseman. It directly executes whatever JavaScript you put in the evaluate() method, and does so as if it's natively running in a browser environment. 

One last thing should be noted in this section, which was alluded to earlier. And that is: not only can you execute custom JavaScript in the browser using the evaluate() method, but you can also inject external scripts into the runtime environment prior to running your evaluation logic. This can be done like so:

```
  phantomInstance
    .open(url)
    .injectJs('scripts/CustomLogic.js')
    .evaluate(function() {
      var x = CustomLogic.getX(); // Assumes variable 'CustomLogic' was loaded by scripts/custom_logic.js
      console.log('Retrieved x using CustomLogic: ', x);
    })
```

By extending the logic above, you could perform virtually any action on any website.

### Conclusion

This tutorial has attempted to demonstrate both a custom CLI microframework and some basic logic for crawling in Node.js, using the Horseman package to leverage PhantomJS. While using a CLI framework would likely benefit many projects, the use of crawling is typically limited to very specific problem domains. One common area is in quality assurance (QA), where crawling can be used for functional and user interface testing. Another is security, where, for example, you might want to crawl your website periodically to detect if it has been defaced or otherwise compromised.

Whatever the case may be for your project, make sure to be cognizant and conscientious when you crawl. Get permission when you can, be polite to the maximum extent that you can, and never [DDoS](https://en.wikipedia.org/wiki/Denial-of-service_attack) anybody, whether accidentally or intentionally. If you suspect that you're generating a lot of automated traffic, then you probably are, and you should likely re-evaluate your goals, implementation, or level of permission.


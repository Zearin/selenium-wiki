# Design Patterns and Development Strategies

Over time, projects tend to accumulate large numbers of tests. As the total number of tests increases, it becomes harder to make changes to the codebase --- a single "simple" change may cause numerous tests to fail, even though the application still works properly. Sometimes these problems are unavoidable, but when they do occur you want to be up and running again as quickly as possible. The following design patterns and strategies have been used before with WebDriver to help making tests easier to write and maintain. They may help you too.

  * DomainDrivenDesign: Express your tests in the language of the end-user of the app.
  * PageObjects: A simple abstraction of the UI of your web app.
  * LoadableComponent: Modeling PageObjects as components.
  * BotStyleTests: Using a command-based approach to automating tests, rather than the object-based approach that PageObjects encourage
  * AcceptanceTests: Use coarse-grained UI tests to help structure development work.
  * RegressionTests: Collect the actions of multiple AcceptanceTests into one place for ease of maintenance.

If your favourite pattern has been left out, please leave a comment on this page, and we'll try and include it. Also, please remember that these are just suggestions: in order to use WebDriver you don't need to follow these patterns and strategies, and it's likely that not all of them will be useful to you. Just use the ones that you like!
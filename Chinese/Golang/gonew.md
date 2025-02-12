## Introduction to gonew

Recently, Go official blog introduced a new tool called gonew. The tool supports based on go project template clone and create a Go project belongs to you. gonew tool introduced to greatly simplify the creation of Go projects, at the same time, due to the support of custom project templates, but also to improve the standardisation of Go projects.


gonew tool has just been put into the Go tools project code repository, is still in the experimental stage, the follow-up may add new features, but the current core features (core functionality) will continue to be retained.

This article will provide a brief description of the current features of gonew for your reference.

1. Origins
Why did the Go team introduce the gonew tool? According to Russ Cox, the Go team often receives requests from Go users to use some kind of “go new” functionality, i.e. to create a new Go module with some kind of basic project template, so Russ Cox wrote a small tool that implements this kind of core functionality in private Russ Cox wrote a little tool privately that implements this core feature: rsc.io/tmp/gonew. The logic of the tool is simple: download a template module, change its module path, and put it in a new directory locally.


After Russ promoted the tool within Google, a number of teams within Google customised some of the templates, with the ServiceWeaver team being particularly responsive. This all culminated in Russ deciding to introduce golang.org/x/tools/cmd/gonew.

Let’s take a look at what gonew actually looks like and what it can do!

2. Installing and using onew
2.1 Installing gonew

It is fairly easy to install onew locally by running the following commands (if GOPATH is set, the tool will be installed under GOPATH/bin):

$go install golang.org/x/tools/cmd/gonew@latest
go: downloading golang.org/x/tools v0.12.0
go: downloading golang.org/x/mod v0.12.0
Execute the gonew:

$gonew
usage: gonew srcmod[@version] [dstmod [dir]]
See https://pkg.go.dev/golang.org/x/tools/cmd/gonew.
2.2 Creating new projects with gonew

Here are two typical scenarios for creating a new project with gonew:

Creating a “module with the same name” project based on a template

Let’s take the golang.org/x/example/helloserver template as an example, and we’ll create a new project based on that template via gonew.

$gonew golang.org/x/example/helloserver
gonew: initialized golang.org/x/example/helloserver in ./helloserver
Explore the project:

$ cd helloserver/
$ ls
LICENSE  go.mod  server.go
$ git status
fatal: Not a git repository (or any of the parent directories): .git
$ cat go.mod
module golang.org/x/example/helloserver

go 1.19
We find that gonew simply downloads the helloserver template project locally (which obviously won’t contain the git repository directory (.git) of the original template project), and the name of the go module remains unchanged.

Many people will ask: in what scenarios would such a use of gonew be used, and Russ Cox gives the scenarios:

$ gonew book.com/mybook-examples
This usage applies to creating a sample code project locally for a book author.

Creating a new module project based on a template

Here we create our own github.com/bigwhite/myhelloserver module project based on the helloserver project template:

$ gonew golang.org/x/example/helloserver github.com/bigwhite/myhelloserver
gonew: initialized github.com/bigwhite/myhelloserver in ./myhelloserver
Again, explore the newly created project:

$ cd myhelloserver/
$ ls
LICENSE  go.mod  server.go
$ cat go.mod
module github.com/bigwhite/myhelloserver

go 1.19
We see: unlike the first usage, this time the module path in go.mod is changed to the one we expect.

This usage should be the most common scenario for gonew.

According to the command description of gonew, it also supports creating a new project based on a specific version of a template project and specifying a path to store the new project locally, which we won’t demonstrate here.

3. gonew’s project template

The project template mentioned in gonew is not mysterious, it is a go module, this module has some scaffolding code, and its existence is to be reused.

Google provides some template examples, such as: go team’s hello, helloserver, and outyet’s template.

Before the appearance of gonew, many organisations probably did just that, and would define some Go scaffolding projects as templates for everyone to refer to when creating new go projects.There are also many open source tools in the Go community that are similar to gonew, and in the gonew discussion threads, many people have shown their own gonew-like projects.


With gonew, the creation of an organisation-level Go project template library will become an important means of improving the efficiency of Go project initialisation within an organisation, and in doing so, the degree of organisation-level Go project standardisation will be significantly improved.

4. gonew and Go project standard layout
When I first saw the title of the Go blog post about gonew, I thought that the Go team had finally announced a standard layout for Go projects, and that the gonew tool would be used to create new projects with the standard layout. But after I read further, I found that this is not the case.


Gonew does not specify a standard layout for Go projects. gonew is open, as long as it is a legitimate go module project can be used as a template when creating a new project. This reflects the flexibility and extensibility of the gonew tool, as for the future go team will define a series of “standard layout” template is still unknown.

5. Summary
The gonew tool simplifies the complexity of initial Go project creation and, based on a number of Go best-practice compliant project templates, Go beginners can get Go projects with Go best-practice directory layouts in minutes. Companies and organisations can also define their own Go templates to meet their internal needs, improving the efficiency of creating new Go projects and standardising the layout of Go projects. The increased standardisation of new Go project layouts will also be more friendly to the CI/CD pipeline within the organisation.


Onew has been well received by the community, who have commented on how useful it has been for getting projects off the ground and have suggested extensions, and Russ Cox has said that we can’t rule out the possibility of upgrading onew to go new in a later version of Go.

https://www.sobyte.net/post/2023-08/gonew/

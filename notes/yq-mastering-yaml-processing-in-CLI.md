
https://dev.to/martinheinz/yq-mastering-yaml-processing-in-command-line-o0g
by Martin Heinz

yq: Mastering YAML Processing in Command Line
#devops
#yaml
#tutorial

Nowadays, YAML is used for configuring almost everything (for better or worse), so whether you're DevOps engineer working with Kubernetes or Ansible, or developer configuring logging in Python or CI/CD with GitHub Actions - you have to deal with YAML files at least from time-to-time. Therefore, being able to efficiently query and manipulate YAML is essential skill for all of us - engineers. The best way to learn that is by mastering YAML processing tool such as yq, which can make you way more efficient at many of your daily tasks, from simple lookups to complex manipulations. So, let's go through and learn all that yq has to offer - including traversing, selecting, sorting, reducing and much more!
Setting Up

Before we begin using yq, we first need to install it. When you google yq though, you will find two projects/repositories. First of them, at https://github.com/kislyuk/yq is wrapper around jq - the JSON processor. If you're already familiar with jq you might want to grab this one and use the syntax you already know. In this article though, we will use the other - a bit more popular project - from https://github.com/mikefarah/yq. This version does not 100% match the jq syntax, but its advantage is that it's dependency free (does not depend on jq), for more context on the differences, see following GitHub issue.

To install it, head over to docs and choose installation method suitable for your system, just make sure you install version 4, as that's what we will work with here. On top of that, you might want to setup shell completion, information about that is available at https://mikefarah.gitbook.io/yq/commands/shell-completion.

Now that we have it installed, we also need some YAML file or document to test the commands we will be running. For that we will use the following files, which have all the common things you could find in YAML - attributes (plain and nested), arrays and various value types (string, integers and booleans):

# log.yaml
timestamp: 1620197250
user: root
log:
  level: fatal
  message: |
    Application exited with code 2
---
# user.yaml
user:
  name: John
  surname: Smith
  gender: male
  active: true
  addresses:
    - street: "Oxford Street"
      city: London
      country: England
    - street: "Ludwigstrasse"
      city: Munich
      country: Germany
  orders:
    - 4356436
    - 4345753
    - 2345234

With this out of the way, let's learn the basics!
The Basics

All the commands we will be running will start with the same base of yq eval, followed by quoted expression and your YAML file. One exception to this would be pretty-printing of YAML files, in which case you would omit the expression - for example yq eval some.yaml - in case you're familiar with jq, then this is equivalent to cat some.json | jq ..

Optionally we can also tack on some extra flags, some of the more useful ones would be -C to force colored output, -I n to set indentation of output to n spaces, or -P for pretty-printing.

As for the basic expressions, there's a lot of things we can do, but the most common one is traversing YAMLs, or in other words - looking up some key in the YAML document. This is done using . (dot) operator and in the most basic form, this would look like so:

# yq eval '.user.addresses' user.yaml

- street: "Oxford Street"
  city: London
  country: England
- street: "Ludwigstrasse"
  city: Munich
  country: Germany

In addition to the basic map navigation, you will often want to lookup specific index in an array (using [N]):

# yq eval ".user.addresses[1]" user.yaml

street: "Ludwigstrasse"
city: Munich
country: Germany

And finally, you might also find splat operator useful which flattens maps/arrays (notice the difference with the first example we looked at):

# yq eval ".user.addresses[]" user.yaml

street: "Oxford Street"
city: London
country: England
street: "Ludwigstrasse"
city: Munich
country: Germany

Apart from basic traversing you might also want to get familiar with selecting, which allows you to filter by boolean expressions. For this we use select(. == "some-pattern-here"). Here, simple example can be filtering based on leading digits:

yq eval '.user.orders[] | select(. == "43*")' user.yaml

4356436
4345753

This example also shows usage of pipe (|) - we use it to first navigate to the part of the document which we want to filter and then pass it on to select(...).

In the above example we used == to find fields that are equal to the pattern, but you can also use != to match ones that are not equal. Additionally you can omit the select function altogether and instead of values you will get only boolean results of the matching:

yq eval '.user.orders[] | (. != "43*")' user.yaml

false
false
true

Whether you're completely new to yq or you've been using it for a while, you'll surely run into issues where you will have no idea why your query doesn't return what you want. In those situations you can use -v flag to produce verbose output, which might give you info as to why the query behaves the way it does.
Advanced Querying

The previously section showed the basics, which are often sufficient for quick lookups and filtering, but sometimes you might want to use more advanced functions and operators, for example when automating certain tasks that involves YAML input and/or output. So, let's explore a few more things that yq has to offer.

Sometimes it might be useful to sort keys in the document, for example if you're versioning your YAMLs in git or just for general readability. It's also very convenient if you need to diff 2 YAML files. To do this we can use 'sortKeys(...)' function:

# yq eval 'sortKeys(.user)' user.yaml
user:
  active: true
  addresses:
    - street: "Oxford Street"
      city: London
      country: England
    - street: "Ludwigstrasse"
      city: Munich
      country: Germany
  gender: male
  name: John
  orders:
    - 4356436
    - 4345753
    - 2345234
  surname: Smith

If the input YAML document is dynamic and you're not sure what keys will be present, it might make sense to first check for their presence with has("key"):

yq eval '.log | has("message")' log.yaml
true

yq eval '.log | has("code")' log.yaml
false

Similar to the case with has("key"), you might need to first get the dynamic list of keys before making certain operations with the document, for that you can use keys function:

yq eval '.log | keys' log.yaml
- level
- message

Checking length of a value might be necessary for input filtering/validation or to make sure the value doesn't overflow some predefined bounds. This is done using length function:

yq eval '.log.message | length' log.yaml
31

yq eval '.user.addresses[0].street | length' user.yaml
13

For automation tasks that have parametrized inputs, you will surely need to pass environment variables into yq queries. You can obviously use normal shell environment variables, but you will end up with very tricky and hard to read quote escaping. Therefore, it might be better to use yq's env() function instead:

level="warning" yq eval '.log.level == env(level)' log.yaml
false

level="fatal" yq eval '.log.level == env(level)' log.yaml
true

field="surname" yq eval '.user.[env(field)]' user.yaml
Smith

To simplify processing of some fields or arrays you can also use some string function such as join or split to concatenate or breakup text:

yq eval '.user.orders | join(", ")' user.yaml
4356436, 4345753, 2345234

yq eval '.log.message | split(" ")' log.yaml
- Application
- exited
- with
- code
- 2

Last and probably most complex example for this section is transformation of data using ireduce. To have a good reason to use this function you would need a quite complex YAML document which is not something I want to dump here. So, instead, to at least give you an idea of how the function works, let's use it to implement "poor man's" version of join from previous example:

yq eval '.user.orders[] as $item ireduce (""; (. + " ") + $item)' user.yaml
 4356436 4345753 2345234

This one isn't as self explanatory as the previous ones, so let's break it down a bit. First half (.user.orders[] as $item ireduce) of the query takes some iterable field (sequence) from YAML and assigns it to variable - in this case $item. In the second part, we define initial value ""; (empty string) and an expression that will be evacuated for each $item - here that would be the value that was there previously, joined with space ((. + " ")) followed by the item we are currently iterating over (+ $item).
Manipulating and Modifying

Most of the time you will need to only do searches, lookups and filtering of existing documents, but from time to time you might need to also manipulate YAMLs and create new ones from them. yq provides a couple of operators to do these kinds of tasks, so let's go over them briefly and see a few examples.

The simplest one is union operator, which is really just a , (comma). It allows us to combine results of multiple queries. This can be useful if you need to extract multiple parts of YAML at the same time, but cannot do it with single query:

yq eval '.user, .log.level, .log.message' log.yaml
root
fatal
Application exited with code 2

Another fairly common use-case would be adding record to array or concatenating 2 arrays. This is done with + (plus) operator:

# yq eval '.user.orders += 2845234' user.yaml
...
  orders:
    - 4356436
    - 4345753
    - 2345234
    - 2845234

Another handy one is update operator (=), which (surprise, surprise) updates some field. Very simple example of updating log level in our sample YAML:

# yq eval '.log.level = "warning"' log.yaml

timestamp: 1620197250
user: root
log:
  level: warning
  message: |
    Application exited with code 2

It's important to point out here, that by default the result is sent to standard output and not to the original file. To make in-place update, you will need to use the -i option.

There a few more operators available, but aren't particularly useful (most of the time), so I won't show you bunch of examples that probably would not help you, instead I will give you some links to docs in case you want to know dig a little deeper:

    Number subtraction
    Multiplication (merging)
    Deletion

Handy Examples

Now that we know the theory, let's look at some examples and handy commands that you can incorporate into your workflow right away.

For obvious reasons, we start with Kubernetes as it's probably the most popular project that uses YAML for configuration. The simplest, yet very useful thing that yq can help us with is pretty-printing Kubernetes resources or querying specific sections of a manifest:

kubectl get pods some-pod -o yaml | yq eval '.spec' -

Another thing we can do is list resource name and specific attribute. This can be handy for finding or extracting all listening ports of Services or for example to lookup pods image for every pod in namespace:

kubectl get pods -o yaml | yq eval '{ .items[].metadata.name: .items[].spec.containers[0].image }' -
first-pod: quay.io/repository/image:latest
second-pod: icr.io/repository/image:latest
third-pod: icr.io/repository/image:1.0.0

Notice that above we had to use .items[] because when you get all instances of a resource, the returned Kind is a List of items.

In case of resources such as Pods, Deployments or Services which often have many instances in each namespace, it might be undesirable to just dump all of them into console and sift through them manually. So, instead you could filter them on some attribute, for example list name and listening port of only the services that are exposed on particular port:

kubectl get svc -o yaml | yq eval '.items[] | select(.spec.ports[].port == 8443) | { .metadata.name: .spec.ports[].targetPort }' -
service-one: https
service-two: 8080

As all Kubernetes "YAML Engineers" know, sometimes it can be difficult to remember all the fields in some particular resource, so why not just query all the keys for example for Deployment's spec.template.spec?

kubectl get deploy -o yaml | yq eval '.items[0].spec.template.spec | keys' -
- containers
- dnsPolicy
- nodeSelector
...

Moving on from Kubernetes, how about some docker-compose? Maybe you need to temporarily delete some section such as volumes or healthcheck - well here you go (this is destructive, so be careful):

yq eval -i 'del(.services.backend.healthcheck)' docker-compose.yml

In similar manner, you could also remove task from Ansible Playbook. Speaking of which - how about changing remote_user in all tasks in an Ansible Playbook - here, let's change it to root:

yq eval '.[].remote_user = "root"' playbook.yml

Closing Thoughts

I hope this "crash course" will help you get started with using yq, but as with any tool, you will only learn to use it by practicing and actually doing real world tasks, so next time you need to lookup something in YAML file, don't just dump it into terminal, but rather write a yq query to do the work for you. Also, if you're struggling to come up with query for your particular task and Google search doesn't turn up anything useful, try searching for solution that uses jq instead - the query syntax is almost the same and you might have better luck searching for jq solutions, considering that it's more popular/commonly used tool.
Top comments (0)
pic
Code of Conduct • Report abuse
profile
Pulumi
Promoted

Pulumi Azure GitHub Image
"There is no way around the fact that devops is complicated but Pulumi is a game changer for me" -Brian M.

Build and ship infrastructure faster using languages you know and love. Use Pulumi’s open source SDK to provision infrastructure on any cloud, and securely and collaboratively build and manage infrastructure using Pulumi Cloud.

Get Started
Read next
okikio profile image
From Docker to Podman - VS Code DevContainers

Okiki Ojo - Oct 14
sabbirsobhani profile image
How to Dynamically Change Favicon Icon in Next.js 14

Sabbir Sobhani - Nov 4
elizabethlomb profile image
Integrate MongoDB database with multiple collections using Mongoose in NestJS and Typescript

Elizabeth Morillo - Oct 25
efpage profile image
Easy use of MATH

Eckehard - Oct 16
Martin Heinz
My name is Martin Heinz and I'm a software developer/DevOps engineer. I'm from Slovakia, living in Bratislava.

    Location
    Bratislava, Slovakia
    Education
    Masters in Applied Computer Science
    Work
    DevOps Engineer at IBM
    Joined
    Aug 6, 2019


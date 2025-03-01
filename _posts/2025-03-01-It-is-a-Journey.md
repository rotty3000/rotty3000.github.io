---
title: It's a Journey
excerpt: About my recent transition from mostly Java to mostly Kubernetes and Go.
---

It's been a long time since I've written publicly. I've been busy with work and life. I'm sure everyone can relate.

I was going to dive right into technical details, but I think it's important to understand the why behind the what so I want to start with a brief overview of my recent journey.

The bulk of my career has been spent in Java. My beloved language where I spent the first 15 years of my career honing my skills every day; to this day is still a comfort zone for me.

I've been using it since the early 2000s where it was the main language for the curriculum at my university and while I've loved taking side roads into other languages and technologies, I've always come back to Java.

What I enjoyed so much about Java was, and is, the vastness of the ecosystem. The JVM is a powerful runtime that allows me to build performant applications. And the landscape of tools and libraries is so vast and prolific that I could always find something to help me build the right thing. The development experience was second to none for so long. Few language ecosystems could boast such a productive set of tools and features that allowed huge teams to collaborate on the same codebase. Starting from IDEs, build tools, testing frameworks, debugging tools, analytics tools, and so much more, the ecosystem is mature and full of options.

It's even hard to believe a day would come where I would opt for anything else... and yet here we are. But I have to admit the choice was kind of made for me in a sense.

Over the course of 2 decades I worked mostly in Java on a web centric enterprise application. A monolithic application with millions of lines of code, an engineering team well in the hundreds, having countless features and integrations.

These days however, I've been working on "Cloud Native" applications; applications that are built to run in infrastructure that is not under my control. Bare metal barely exists today and even where it does its use is almost exclusively buried deep under layers of abstraction. Even on my local machine I'm using Docker or Kubernetes (likely both) to run my applications.

I'm working a lot with Kubernetes. And, it just so happens that the defacto language for the Kubernetes ecosystem is Go; and the most interesting work being done in the Kubernetes ecosystem is being done in Go.

The Go ecosystem is already very mature and the tooling is very good. What I like very much about Go is the simplicity of the developer experience. You immediately have opinionated tooling for building, testing, packaging and running your application. The cross compilation story is so easy that I don't even think about it. Dependency management is just as easy and assembling an application from multiple packages is a breeze. I can easily debug both directly and remotely launched applications. I can write powerful tests, I can execute them in seconds and I can effortlessly verify code coverage. I can write complex applications and build them into a statically linked binary of only a few megabytes or I can package that binary into a container of only a few megabytes (most of the applications I've worked on have resulted in images of less than 15Mb).

All of this I can do within a matter of minutes after I've run `go mod init my/app` on a new project.

The language itself is simple enough for my brain to process quickly. And since I'm not enough of a language lawyer to care about the language features too much, I can focus on building my application. I've gotten used to the error handling and the concurrency model and honestly I rather like them. I am by no means a Go expert, but I've been able to pick it up quickly and apply it to build some really cool stuff. Of note are the HTTP server primitives in the standard library which are so good that I'm doing complex things in a matter of minutes.

Go is great both for building system tools and for building web applications and since those two places are where I live, I'm happy.

If you care to leave feedback, please do so on [LinkedIn](https://www.linkedin.com/in/raymond-auge/).

Again, I hope to write more about what I've been doing and what I've learned in the coming weeks.

Thanks for stopping by!

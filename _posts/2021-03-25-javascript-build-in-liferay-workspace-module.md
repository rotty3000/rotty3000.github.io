---
title: Adding JavaScript build to a Liferay Workspace OSGi module build
excerpt: Adding non-trivial JavaScript to a Liferay Workspace module project has always caused me some degree of pain. This post demonstrates how I accomplish this these days.
---

Over the past several years I've been challenged more frequently with working with JavaScript. I'm not a JavaScript expert by any means so often knowledge eludes me and I feel like an outsider at trying to grok it all.

Recently however, the degree to which I've had to immerse myself has lead to more understanding. With that in mind, this article is for JavaScript beginners, like myself, trying to be productive on the outer rim of the JavaScript universe.

Today's challenge starts with Liferay Workspace but what I should start by clarifying is that I am not talking about a [Liferay NPM Bundler](https://help.liferay.com/hc/en-us/articles/360028832852-liferay-npm-bundler) type of build, nor one that produces output intended to integrate (per say) with the [Liferay JavaScript Module Loaders](https://help.liferay.com/hc/en-us/articles/360029314891-JavaScript-Module-Loaders). Those procedures result in output that is proprietary to Liferay and therefore cannot work with pure JavaScript consumers (such as you'd expect to find within things like third party web components or SPAs of various types.) So the idea here is to produce JavaScript output that is *not* proprietary to Liferay.

Secondly, I should clarify that I am not suggesting that Liferay's JavaScript tooling is unnecessary. The tooling is both needed and great at addressing many concerns that need to be taken into consideration when participating in an ecosystem of a vast product like **Liferay DXP**. However, sometimes you just have other constraints. That's the perspective that is driving this discussion.

So, with all that out of the way, let's get started with a basic Liferay Workspace project setup.

You can follow along by looking at the [adding-javascript-build](https://github.com/rotty3000/com.github.rotty3000.workspace/tree/adding-javascript-build) branch of my workspace **Github** project. In this procedure I'm going to start with a new workspace to outline all the steps to get started.

### The workspace

Make sure that you've got [blade cli](https://help.liferay.com/hc/en-us/articles/360017885232-Installing-Blade-CLI-) installed.

Create (init) a new workspace by executing:

```bash
blade init -v portal-7.2-ga2 <name of workspace>
```

 *(You can see a list of the most recent versions of every Liferay Portal product available by executing `blade init -l`. And if you need an exhaustive list you can use `blade init -l --all`)* 

Once the workspace is created I always recommend verifying the very latest version of the workspace plugin is specified in the `settings.gradle` file in the workspace root. To find the latest version of the workspace plugin look [here](https://search.maven.org/artifact/com.liferay/com.liferay.gradle.plugins.workspace).

At the time of this writing the change to update my workspace plugin looked like this:

```diff
 buildscript {
 	dependencies {
-		classpath group: "com.liferay", name: "com.liferay.gradle.plugins.workspace", version: "3.4.2"
+		classpath group: "com.liferay", name: "com.liferay.gradle.plugins.workspace", version: "3.4.5"
 		classpath group: "net.saliman", name: "gradle-properties-plugin", version: "1.4.6"
 	} 
```

Next let's test that the workspace is properly configured by executing

```bash
./gradlew initBundle
```

Sure there are no modules to speak of but performing this action downloads the workspace specified version of gradle, if not already downloaded, it retrieves all the dependencies for a slew of default plugins with which the Liferay Workspace come pre-configured and finally it downloads the bundle of the Liferay version specified.

With the Liferay bundle initialised and assuming you've not changed any other workspace settings (in `gradle.properties`) it should now be possible to start a running instance of Liferay portal by executing the command:

```bash
bundles/tomcat-${tomcat_version}/bin/catalina.sh run
```

*(You can determine the tomcat version just by looking in the bundles directory. I usually use `tab` completion to auto-complete paths to save time.)*

This command starts Tomcat (*and Liferay*) in the foreground with logs outputted to `stdout`. You can stop  Tomcat and Liferay with `ctrl-D`.

Now personally, I invariably end up writing code that needs to be debugged. This means I end up needing to run Liferay portal in debug mode. Thankfully this is trivial with Tomcat's built in debug launch mode. Just add `jpda` to the command above before the `run` argument. The full command ends up being

```bash
bundles/tomcat-${tomcat_version}/bin/catalina.sh jpda run
```

and Tomcat starts up with remote debugging enabled on port `8000`.

One nice thing about the Liferay portal bundle is that without any other configuration it will run locally with the embedded **HyperSQL** database and **Elasticsearch** search engine to make it easy to get underway.

### The module

Now let's start a new module. Start by moving into the `modules` directory of the workspace and then execute the following:

```bash
blade create -t service -s java.util.function.Supplier adding-javascript-build
```

This is a simple `service` type project with a class that implements `java.util.function.Supplier` to be published as an **OSGi** service. This module is inane but demonstrative enough to get us going. We're not focusing on the OSGi part specifically at this time but we're setting out to prove that while we have an OSGi module we're also going to get some JavaScript goodness too.

### The generated class

Let's take a moment to touch on the class that was generated as part of the module creation. It looks like this (with comments removed):

```java
package adding.javascript.build;
import java.util.function.Supplier;
import org.osgi.service.component.annotations.Component;
@Component(
	immediate = true,
	property = {
	},
	service = Supplier.class
)
public class AddingJavascriptBuild implements Supplier {
}
```

The first thing you probably noticed is that it doesn't even compile. This is because the template didn't fill out any required methods for the service type we picked. Let's fix that, and also let's add the generic type while we're at it, make it return something interesting, make it a [gogo](https://felix.apache.org/documentation/subprojects/apache-felix-gogo.html) command and finally we'll add some (de)activate output so we can see what's happening as we (re)deploy the module:

```diff
@@ -1,11 +1,27 @@
 package adding.javascript.build;
 import java.util.function.Supplier;
+import org.osgi.service.component.annotations.Activate;
 import org.osgi.service.component.annotations.Component;
+import org.osgi.service.component.annotations.Deactivate;
 @Component(
 	immediate = true,
 	property = {
+		"osgi.command.scope=ajb",
+		"osgi.command.function=get"
 	},
 	service = Supplier.class
 )
 public class AddingJavascriptBuild implements Supplier<String> {
+	@Override
+	public String get() {
+		return "Something interesting!";
+	}
+	@Activate
+	void activate() {
+		System.out.println("Activated!");
+	}
+	@Deactivate
+	void deactivate() {
+		System.out.println("Deactivated!");
+	}
 }
```

Once we've built and deployed this module we should be able to telnet into Liferay's embedded Gogo shell by executing `telnet localhost 11311`. Once there execute the `ajb:get` command. You should see the output of our supplier service:

```bash
]$ telnet localhost 11311
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
____________________________
Welcome to Apache Felix Gogo

g! ajb:get
Something interesting!
g! 
```

### Enable WAB

Next, we're going to enable the module as a **skinny WAB** (Web Application Bundle). In Liferay parlance a *skinny WAB* is a *web application* that is shaped like a **JAR** instead of like a **WAR** and the only thing we need is to edit the `bnd.bnd` file that is in the root of the module and add the header `Web-ContextPath` like so:

```diff
 Bundle-Name: adding-javascript-build
 Bundle-SymbolicName: adding.javascript.build
 Bundle-Version: 1.0.0
+
+Web-ContextPath: /adding-javascript-build
```

The value of the header should contain a context path unique for our application.

The reason for turning our module into a WAB is so that we can serve resources. This will be essential later when we have some JavaScript to serve.

### Vanilla JavaScript

Now, if all we needed was some vanilla JavaScript code then we'd only need to place the files within the `src/main/resources/META-INF/resources/` directory of our module and that would be sufficient. The files would automatically be accessible from `<hostname>/o/adding-javascript-build/...` due to our module being a WAB. The `/o` is the path withing Liferay used to target OSGi modules and is added to any WAB paths. In a skinny WAB all resources under the path `src/main/resources/META-INF/resources/` (in the built JAR this translates to `/META-INF/resources`) are mapped to a default **Servlet** that serves them up using an appropriate mime type. Sweet right?

### Advanced JavaScript

Assuming that we're using more than vanilla JavaScript what we'll do is initialise a JavaScript build by performing:

```bash
npm init -y
```

This gives us a simple `package.json` file to start with in the root of our module directory.

Next let's create the file `src/main/resources/META-INF/resources/js/index.ts`. The `.ts`  extension stands for [TypeScript](https://www.typescriptlang.org/) which is a popular way these days of developing JavaScript in a type safe way (go figure ;) ). Let's start simple:

```javascript
const user = {
  firstName: "Rotty",
  lastName: "3000",
  role: "Programmer guy"
};

console.log("User: %O", user);

export {};
```

Right, so now we have a source file, and we have the basic `package.json` but we still can't build because we don't have a builder. If we tried to run the workspace build right now it would not be happy.

However, you should notice that Liferay Workspace recognises the fact that there is a JavaScript build in the module and does some setup and attempts to execute some npm tasks. However it errors out with something along the lines of:

```bash
> Task :modules:adding-javascript-build:downloadLiferayModuleConfigGenerator

...

> core-js@2.6.12 postinstall /home/rotty/projects/com.github.rotty3000.workspace/modules/adding-javascript-build/node_modules/core-js
> node -e "try{require('./postinstall')}catch(e){}"

...

npm ERR! core-js@2.6.12 postinstall: `node -e "try{require('./postinstall')}catch(e){}"`
npm ERR! Exit status 139
npm ERR! 
npm ERR! Failed at the core-js@2.6.12 postinstall script.
npm ERR! This is probably not a problem with npm. There is likely additional logging output above.

npm ERR! A complete log of this run can be found in:
npm ERR!     /home/rotty/.npm/_logs/2021-03-22T03_53_00_117Z-debug.log

> Task :modules:adding-javascript-build:downloadLiferayModuleConfigGenerator FAILED

...

BUILD FAILED in 31s
4 actionable tasks: 4 executed
```

Don't even bother trying to make sense of this, it's really just a long winded way of saying

>  I really have no idea what's going on... Please help!!!

Liferay Workspace is trying to use it's built in NPM builder plugin configuration but is failing to find what it thinks are critical pieces of information to tell it what to do. To get around this we need to add a `build` script defined in `package.json` that will perform an action of our choosing. Let's start with something very trivial:

```diff
@@ -4,7 +4,8 @@
 	"description": "",
 	"main": "index.js",
 	"scripts": {
-		"test": "echo \"Error: no test specified\" && exit 1"
+		"test": "echo \"Error: no test specified\" && exit 1",
+		"build": "echo 'test, test, test'"
 	},
 	"keywords": [],
 	"author": "",
```

If you rebuild now you should see that Liferay Workspace accepted that we're taking over and ran our simple `build` script. This means we're now in control. Great!

Since we're creating a Liferay module we surely never intend to publish this project to an NPM repository so we should declare the package `private` and we also don't need an entry point per say so let's make both edits next:

```diff
@@ -2,7 +2,7 @@
 	"name": "adding-javascript-build",
 	"version": "1.0.0",
 	"description": "",
-	"main": "index.js",
+	"private": true,
 	"scripts": {
 		"test": "echo \"Error: no test specified\" && exit 1",
 		"build": "echo 'test, test, test'"
```

### NPM/Node Version

When working with the most recent tooling we often run into limitations if using older versions of NPM and/or Node. Liferay's conservative defaults are a little too dated to work with the state of the art in JavaScript. Let's boost that up but not too high. We want to ensure the best compatibility in case the workspace contains older projects. I've found that the sweet spot is around version `12.20.0` of Node. Most order JavaScript projects only need slight alterations to work with this version and we also are able to execute almost all the latest JavaScript tooling.

Open the `build.gradle` file in the root of the Liferay Workspace and add the following (if you have existing build configurations just make sure to integrate this):

```groovy
allprojects {
	plugins.withId("com.liferay.node") {
		node.global = true
		node.nodeVersion = '12.21.0'
		node.npmVersion = '6.14.11'
	}
}
```

### [TypeScript](https://www.typescriptlang.org/)

In this specific scenario we have TypeScript source files which means we need to compile them so we need the tools to handle that:

```bash
npm install typescript --save-dev
```

We also need to configure some details of TypeScript compilation. For this we need a configuration file called [`tsconfig.json`](https://www.typescriptlang.org/tsconfig). Create this file in the module directory with the following contents:

```json
{
	"compilerOptions": {
		"allowJs": true,
		"checkJs": true,
		"module": "es6",
		"moduleResolution": "node",
		"noImplicitAny": true,
		"outDir": "./build/resources/main/META-INF/resources/",
		"rootDir": "./src/main/resources/META-INF/resources/",
		"sourceMap": true,
		"strict": true,
		"target": "es6"
	},
	"include": [
		"./src/main/resources/META-INF/resources/**/*"
	]
}
```

Check the documentation for the specific details of each configuration but this is approximately what is needed for our use case, where we have a *Maven-ish* project structure which produces a *skinny WAB* result so our configuration reflects that.

You can test the output by executing the TypeScript compiler command directly from the command line:

```bash
tsc
```

If you have any errors whether logical, or with type safety you should find out pretty quickly.

We could stop here if all we wanted was to compile our TypeScript sources into JavaScript. We could modify the `build` script to invoke the TypeScript compiler we'd be off to the races.

```diff
 	"private": true,
 	"scripts": {
 		"test": "echo \"Error: no test specified\" && exit 1",
-		"build": "echo 'test, test, test'"
+		"build": "tsc"
 	},
 	"keywords": [],
 	"author": "",

```

The resulting output of executing the gradle `jar` task is now:

```bash
]$ bnd print -l build/libs/adding.javascript.build-1.0.0.jar 
[LIST]
META-INF
  MANIFEST.MF
META-INF/resources
META-INF/resources <no contents>
META-INF/resources/js
  index.js
  index.js.map
  index.ts
OSGI-INF
  adding.javascript.build.AddingJavascriptBuild.xml
adding
adding <no contents>
adding/javascript
adding/javascript <no contents>
adding/javascript/build
  AddingJavascriptBuild.class
```

But we're not done yet so let's move ahead.

### [Webpack](https://webpack.js.org)

.. is "[a *static module bundler* for modern JavaScript applications.](1)"

We want Webpack to handle the complexity of _bundling_ (a.k.a. assembling) any dependencies we might introduce. We'll install it by executing:

```bash
npm install webpack webpack-cli --save-dev
```

While Webpack can function without configuration, in our case we have loftier goals than what the basic configuration supports. So let's start with a relatively simple `webpack.config.js` file. Keep in mind we also want to handle the fact that we have TypeScript:

```javascript
const path = require('path');

const PUBLIC_PATH = '/o/adding-javascript-build/';

module.exports = {
	mode: 'production',
	context: path.resolve(__dirname),
	entry: './src/main/resources/META-INF/resources/js',
	module: {
		rules: [
			{
				test: /\.tsx?$/,
				use: 'ts-loader',
				exclude: /node_modules/,
			},
		],
	},
	resolve: {
		extensions: ['.tsx', '.ts', '.js'],
	},
	output: {
		filename: 'js/bundle.js',
		libraryTarget: 'window',
		path: path.resolve('./build/resources/main/META-INF/resources/'),
		publicPath: PUBLIC_PATH,
	},
};
```

Several key points to look at here. First thing we notice is that in order to handle the TypeScript files we _use_ a plugin called `ts-loader` . We need to install that:

```bash
npm install ts-loader --save-dev
```

The rest of the configuration revolves around where our sources are, and where we want the output to end up. There are also places to configure details about the output module syntax and whether the result is optimised or production and so on. I recommend reading slowly through the webpack configuration documentation to learn the key concepts.

With this we can also change the `build` script to call Webpack, which along with the other Webpack related  changes makes for the following:

```diff
@@ -5,12 +5,15 @@
 	"private": true,
 	"scripts": {
 		"test": "echo \"Error: no test specified\" && exit 1",
-		"build": "tsc"
+		"build": "webpack"
 	},
 	"keywords": [],
 	"author": "",
 	"license": "ISC",
 	"devDependencies": {
-		"typescript": "^4.2.3"
+		"ts-loader": "^8.0.18",
+		"typescript": "^4.2.3",
+		"webpack": "^5.27.2",
+		"webpack-cli": "^4.5.0"
 	}
 }
```

If we run our build again we should now have a jar that contains the following:

```bash
]$ bnd print -l build/libs/adding.javascript.build-1.0.0.jar 
[LIST]
META-INF
  MANIFEST.MF
META-INF/resources
META-INF/resources <no contents>
META-INF/resources/js
  bundle.js
  index.ts
OSGI-INF
  adding.javascript.build.AddingJavascriptBuild.xml
adding
adding <no contents>
adding/javascript
adding/javascript <no contents>
adding/javascript/build
  AddingJavascriptBuild.class
```

And finally, if we want to load our JavaScript file in the portal we can do so by adding the following in the `bnd.bnd` file:

```diff
 Bundle-SymbolicName: adding.javascript.build
 Bundle-Version: 1.0.0
 
+Liferay-JS-Resources-Top-Head: /js/bundle.js?ts=${tstamp}
+Liferay-Top-Head-Weight: 1000
 Web-ContextPath: /adding-javascript-build
```

Couple of points to mention here. First is that we added a build-time auto-generated `ts` parameter to the URL. That will help flush client caches whenever we redeploy a new version of our WAB. Besides that, the URL will be prefixed by the full context path of the WAB at runtime for us. This could have been achieved with file name hashing via Webpack, but for my own reasons I wanted to avoid doing that. Maybe a point for future discussion...

The second point is about the header  `Liferay-Top-Head-Weight`. This allows us to give the file a weight among all other _top head_ JavaScript files in the portal so you have some measure of control on ordering. In this case the larger the number the later it gets added. Values range from `Integer.MIN_VALUE` to `Integer.MAX_VALUE`.

### Conclusion

The ability to add JavaScript builds in your Liferay Workspace modules is pretty essential these days. As it turns out once I waded through the mounds of information and grasped some of the concepts it turned out not to be so bad. In later posts I'll dig into some more advanced scenarios and use cases.

Until next time.



[1]: https://webpack.js.org/concepts/
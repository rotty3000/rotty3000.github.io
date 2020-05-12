---
title: Gradle Integration Test - Starting an external process
excerpt: Recently we worked on a gradle project where we wanted to run integration tests for a soap client.
---

Recently we worked on a gradle project where we wanted to run integration tests for a soap client. This meant starting an external process providing the soap endpoint. Luckily the folks over at [smartbear.com](https://smartbear.com/) provide a nice open source tool for running mocked soap services called [Soap UI](https://www.soapui.org/).

Getting a mocked service up and running was pretty trivial thanks to some pretty good documentation. I won't go into any details because I could never do their documentation justice in a short post. But creating a mock service with responses and some scripts was pretty simple.

Once we had a mock service built our main concern was running it in a CI environment using gradle and wrapped around the soap client test cases.


Executing the mock service standalone involves executing a command like the following:

```bash
sh SoapUI/bin/mockservicerunner.sh -s src/testIntegration/resources/soapui-settings.xml src/testIntegration/resources/soapui-project.xml
```

This starts a process which runs the mocked soap service until any input is sent to stdin which terminates it.

Coordinating this in gradle involves adding some actions before and after the integration tests run.

Note that we added the SoapUI binaries directly into source control so that they would be available to execute during the build.

The first thing we're going to do is create a variable to hold the external process so that we can stop it when we're done:

```groovy
def process
```

In this build integration tests are defined and executed by the `testIntegration` task so adding an action before execution involves configuring the `doFirst` closure for it.

```groovy
tasks.named('testIntegration') {
	doFirst {
		def cmd = "sh SoapUI/bin/mockservicerunner.sh -s src/testIntegration/resources/soapui-settings.xml src/testIntegration/resources/soapui-project.xml"

		process = cmd.execute([], project.projectDir)
		process.consumeProcessOutput(System.out, System.err)
		// give process some time to fully start
		sleep(2000)
	}
}
```

We used the groovy `execute` function and passed two parameters; the environment (`[]`) and the current working directory (`project.projectDir`) so that relative file paths would be relative to the project.

In order to see that the process is started correctly we also made sure groovy consumes all the output from the process and adds it to the gradle output using the `consumeProcessOutput` helper method.

We also need to shutdown the process when tests are done. So we created a `stopSoapUi` task with a `doLast` action:

```groovy
tasks.register('stopSoapUI') {
	doLast {
		println('Stopping SoapUI')
		if (process != null) {
			process.withWriter {it << ' '}
			process.waitForOrKill(1000)
			process = null
		}
	}
}
```

In the `stopSoapUI` task we send a character to the stdin of the process to ask it to terminate properly, then wait for a bit to before we ultimately forcibly kill the process (whichever happens first).

Now that we have our task to stop the process we need to make sure it gets executed regardless of what errors occur either in tests or otherwise. In gradle the way to achieve this type of transactional behaviour is by using the `finalizedBy` function. We add a `finalizedBy` by to the `testIntegration` configuration as follows:

```groovy
tasks.named('testIntegration') {
	doFirst {
		def cmd = "sh SoapUI/bin/mockservicerunner.sh -s src/testIntegration/resources/soapui-settings.xml src/testIntegration/resources/soapui-project.xml"

		process = cmd.execute([], project.projectDir)
		process.consumeProcessOutput(System.out, System.err)
		// give process some time to fully start
		sleep(2000)
	}
	finalizedBy tasks.named('stopSoapUI')
}
```

And there you have it! Using this approach you should be able to handle virtually any external process required for integration testing.

The full code is as follows:

```groovy
def process

tasks.register('stopSoapUI') {
	doLast {
		println('Stopping SoapUI')
		if (process != null) {
			process.withWriter {it << ' '}
			process.waitForOrKill(1000)
			process = null
		}
	}
}

tasks.named('testIntegration') {
	doFirst {
		def cmd = "sh SoapUI/bin/mockservicerunner.sh -s src/testIntegration/resources/soapui-settings.xml src/testIntegration/resources/soapui-project.xml"

		process = cmd.execute([], project.projectDir)
		process.consumeProcessOutput(System.out, System.err)
		// give process some time to fully start
		sleep(2000)
	}
	finalizedBy tasks.named('stopSoapUI')
}
```
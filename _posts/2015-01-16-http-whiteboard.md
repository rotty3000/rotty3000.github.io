---
title: HTTP Whiteboard - OSGI Compendium 6.0
excerpt: How webapps should have been!
---

### State of the Art

* Current Java webapps are generally uninspiring
* There is little to no definition of modularity in Java webapps
* Webapps easily become monoliths
* A weak webapp lifecycle of has been badly abused
* Cross platform support is incredibly complex due to what remains unspecified

### Http Whiteboard

* Started as *RFC 189 - Http Service Updates*
* Evolved into *OSGi Enterprise Section 140 - Http Whiteboard Specification*
** .. an addition which left the original *Http Service* untouched
* Added whiteboard style programming model

### Design Goals

* Bring support for Servlet 3.0 (minimally)
* Enhancements which were long missing
** multiple url patterns
** servlet filters
** event listeners
** error pages
* Clarify relationship between contexts in OSGi and Servlet spec
* Introspection of current state of http services
* Define a capability namespace

### From Monoliths to ... Anything, Please!

* Every modern software engineering discussion spews buckets of descriptive buzzwords all describing in every way anything that is *monolith*.

  See [The Reactive Manifesto](http://www.reactivemanifesto.org/) for details.

### ServletContextHelper

```java
@Component(
	property = {
		HttpWhiteboardConstants.HTTP_WHITEBOARD_CONTEXT_NAME + "=default",
		HttpWhiteboardConstants.HTTP_WHITEBOARD_CONTEXT_PATH + "=/"
	}
)
public class SampleServletContextHelper extends ServletContextHelper {
}
```

### Servlets

```java
@Component(
	property = {
		HttpWhiteboardConstants.HTTP_WHITEBOARD_SERVLET_PATTERN + "=/"
	},
	service = Servlet.class
)
public class SampleServlet extends HttpServlet {
	@Override
	protected void service(
			HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {

		PrintWriter writer = response.getWriter();

		writer.write("Hello World!");
	}
}
```

### Resources

```java
@Component(
	property = {
		HttpWhiteboardConstants.HTTP_WHITEBOARD_RESOURCE_PREFIX + "=/META-INF/resources",
		HttpWhiteboardConstants.HTTP_WHITEBOARD_RESOURCE_PATTERN + "=/resources/*"
	}
)
public class SampleResources {
}
```

### Filters

```java
@Component(
	property = {
		HttpWhiteboardConstants.HTTP_WHITEBOARD_FILTER_PATTERN + "=/*"
	}
)
public class SampleFilter implements Filter {
	@Override
	public void doFilter(
			ServletRequest request, ServletResponse response,
			FilterChain filterChain)
		throws IOException, ServletException {

		filterChain.doFilter(request, response);
	}
}
```

### Event Listeners

```java
@Component
public class SampleServletContextListener implements ServletContextListener {

	@Override
	public void contextDestroyed(ServletContextEvent arg0) {
	}

	@Override
	public void contextInitialized(ServletContextEvent arg0) {
	}

}
```

### Error Pages

```java
@Component(
	property={
		HttpWhiteboardConstants.HTTP_WHITEBOARD_SERVLET_ERROR_PAGE + "=java.io.IOException",
		HttpWhiteboardConstants.HTTP_WHITEBOARD_SERVLET_ERROR_PAGE + "=5xx"
	},
	service = Servlet.class
)
public class SampleErrorPage extends HttpServlet {

	@Override
	protected void service(
			HttpServletRequest request, HttpServletResponse response)
		throws IOException, ServletException {

		//
	}

}
```

### DTOs

```java
@Component(
	immediate  = true,
	property = {
		HttpWhiteboardConstants.HTTP_WHITEBOARD_SERVLET_PATTERN + "=/dto"
	},
	service = Servlet.class
)
public class SampleDTOReportServlet extends HttpServlet {

	@Override
	protected void service(
			HttpServletRequest request, HttpServletResponse response)
		throws IOException, ServletException {

		final HttpServiceRuntime httpServiceRuntime = _httpServiceRuntime;

		if (httpServiceRuntime == null) {
			response.sendError(
				HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
				"Something went terribly wrong...");

			return;
		}

		response.setContentType("application/json");
		response.setCharacterEncoding("UTF-8");

		PrintWriter writer = response.getWriter();

		writer.write(httpServiceRuntime.getRuntimeDTO().toString());

		writer.close();
	}

	@Reference(unbind = "-")
	protected void setHttpServiceRuntime(
		HttpServiceRuntime httpServiceRuntime) {

		_httpServiceRuntime = httpServiceRuntime;
	}

	private volatile HttpServiceRuntime _httpServiceRuntime;

}
```

### Verdict

* I'm excited with the result of the Http Whiteboard specification
* http://www.osgi.org/Specifications/Drafts[OSGi R6 Early Draft]
  (Since has released [Http Whiteboard Specification](https://docs.osgi.org/specification/osgi.cmpn/7.0.0/service.http.whiteboard.html))
* [Eclipse Project 4.5 M3 - New and Noteworthy](https://www.eclipse.org/eclipse/news/4.5/M3/) - **Friday**
* There's very little time left to get your feedback in. But, if you have any, please contact myself or any member of the EEG

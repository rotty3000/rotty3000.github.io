---
title: Liferay Integration Points
excerpt: Documentation for Liferay 7.0+ OSGi integration points
---

Liferay Portal provides ways for developers to extend it and to add new functionality. This is achieved by connecting some programing logic into what we call an *integration point*.

A good analogy for these *integration points* are a computer's USB ports. Through the USB ports many different types of devices can enhance the computer and provide new functionality; a sound output device, a mouse for input, a hard-disk for additional storage, even such devices as microscopes and musical instruments. Additionally, it's possible for any number of devices to be connected to the USB bus allowing for a virtually infinite number or enhancements to be added to the computer.

In Liferay these *Integration points* are implemented by applying the concept of *OSGi Services*.

An *OSGi Service* is defined as

> "_An object registered with the service registry under one or more interfaces together with properties._"

Breaking this down we find that a service must meet 4 specific criteria:

1. it must be a java object
2. it must implement one or more interfaces
3. it declares a set of properties
4. it is registered with the *_Service Registry_*

The *Service Registry* is the central registry for all services within the *OSGi Runtime*.

We can define the function that registers a service as

```
service_registration[object] = f(object, interfaces+, properties*)
```

*Integration Points* define a constraint for services from the service registry which should be plugged into it. These constraints typically include an interface but may use more advanced filtering which can include not only interface restrictions but may use concise expressions against service properties.

Constraints are expressed using the filter syntax describe in section 3.2.7 of the [OSGi Core Specification](https://docs.osgi.org/specification/osgi.core/7.0.0/framework.module.html#framework.module.filtersyntax).

Given the following filter expression:

```
(&(objectClass=com.example.Foo)(com.example.color=red))
```

_where `objectClass` describes the interface constraint._

The following service will match

```
service_registration[x] = f(x, com.example.Foo, "com.example.color=red")
```

The following services will *not* match

```
service_registration[y] = f(y, com.example.Foo, "com.example.color=green")
service_registration[z] = f(z, com.example.Fee, "com.example.color=red")
```

### List of Core Integration Points

Liferay Portal's core contains the following integration points described by the interface type and any additional filter expression to be matched.

|Interface|Filter|
|===|===|
|`com.liferay.mail.util.Hook` |`(objectClass=com.liferay.mail.util.Hook)`|
|`com.liferay.portal.kernel.comment.CommentManager`|`(objectClass=com.liferay.portal.kernel.comment.CommentManager)`|
|`com.liferay.portal.kernel.search.IndexerPostProcessor`|`(&(indexer.class.name=*)(objectClass=com.liferay.portal.kernel.search.IndexerPostProcessor))`|
|`com.liferay.portal.kernel.scheduler.SchedulerEntry`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portal.kernel.scheduler.SchedulerEntry))`|
|`com.liferay.portal.service.ServiceWrapper`|`(objectClass=com.liferay.portal.service.ServiceWrapper)`|
|`java.util.ResourceBundle`|`(&(!(javax.portlet.name=*))(language.id=*)(objectClass=java.util.ResourceBundle))`|
|`com.liferay.portal.kernel.poller.PollerProcessor`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portal.kernel.poller.PollerProcessor))`|
|`com.liferay.portal.kernel.pop.MessageListener`|`(objectClass=com.liferay.portal.kernel.pop.MessageListener)`|
|`com.liferay.portal.kernel.repository.registry.RepositoryDefiner`|`(objectClass=com.liferay.portal.kernel.repository.registry.RepositoryDefiner)`|
|`com.liferay.portal.kernel.sanitizer.Sanitizer`|`(objectClass=com.liferay.portal.kernel.sanitizer.Sanitizer)`|
|`com.liferay.portal.security.auth.AuthFailure`|`(&(key=*)(objectClass=com.liferay.portal.security.auth.AuthFailure))`|
|`com.liferay.portal.security.auth.Authenticator`|`(&(key=*)(objectClass=com.liferay.portal.security.auth.Authenticator))|`
|`com.liferay.portal.security.auth.AuthVerifier`|`(objectClass=com.liferay.portal.security.auth.AuthVerifier)`|
|`com.liferay.portal.security.auth.EmailAddressGenerator`|`(objectClass=com.liferay.portal.security.auth.EmailAddressGenerator)`|
|`com.liferay.portal.security.auth.EmailAddressValidator`|`(objectClass=com.liferay.portal.security.auth.EmailAddressValidator)`|
|`com.liferay.portal.security.auth.FullNameGenerator`|`(objectClass=com.liferay.portal.security.auth.FullNameGenerator)`|
|`com.liferay.portal.security.auth.FullNameValidator`|`(objectClass=com.liferay.portal.security.auth.FullNameValidator)`|
|`com.liferay.portal.security.auth.ScreenNameGenerator`|`(objectClass=com.liferay.portal.security.auth.ScreenNameGenerator)`|
|`com.liferay.portal.security.auth.ScreenNameValidator`|`(objectClass=com.liferay.portal.security.auth.ScreenNameValidator)`|
|`com.liferay.portal.security.ldap.AttributesTransformer`|`(objectClass=com.liferay.portal.security.ldap.AttributesTransformer)`|
|`com.liferay.portal.security.membershippolicy.OrganizationMembershipPolicy`|`(objectClass=com.liferay.portal.security.membershippolicy.OrganizationMembershipPolicy)`|
|`com.liferay.portal.security.membershippolicy.RoleMembershipPolicy`|`(objectClass=com.liferay.portal.security.membershippolicy.RoleMembershipPolicy)`|
|`com.liferay.portal.security.membershippolicy.SiteMembershipPolicy`|`(objectClass=com.liferay.portal.security.membershippolicy.SiteMembershipPolicy)`|
|`com.liferay.portal.security.membershippolicy.UserGroupMembershipPolicy`|`(objectClass=com.liferay.portal.security.membershippolicy.UserGroupMembershipPolicy)`|
|`com.liferay.portal.security.pwd.Toolkit`|`(objectClass=com.liferay.portal.security.pwd.Toolkit)`|
|`com.liferay.portal.security.sso.SSO`|`(objectClass=com.liferay.portal.security.sso.SSO)`|
|`com.liferay.portal.security.permission.BaseModelPermissionChecker`|`(&(model.class.name=*)(objectClass=com.liferay.portal.security.permission.BaseModelPermissionChecker))`|
|`com.liferay.portal.security.auth.AutoLogin`|`(objectClass=com.liferay.portal.security.auth.AutoLogin)`|
|`com.liferay.portal.kernel.struts.path.AuthPublicPath`|`(objectClass=com.liferay.portal.kernel.struts.path.AuthPublicPath)`|
|`com.liferay.portal.kernel.struts.StrutsAction` + `com.liferay.portal.kernel.struts.StrutsPortletAction`|`(&(\|(objectClass=com.liferay.portal.kernel.struts.StrutsAction)(objectClass=com.liferay.portal.kernel.struts.StrutsPortletAction))(path=*))`|
|`com.liferay.portal.model.LayoutTypeController`|`(&(layout.type=*)(objectClass=com.liferay.portal.model.LayoutTypeController))`|
|`com.liferay.portal.kernel.xmlrpc.Method`|`(objectClass=com.liferay.portal.kernel.xmlrpc.Method)`|
|`com.liferay.portlet.dynamicdatamapping.render.DDMFormFieldRenderer`|`(objectClass=com.liferay.portlet.dynamicdatamapping.render.DDMFormFieldRenderer)`|
|`com.liferay.portlet.dynamicdatamapping.render.DDMFormFieldValueRenderer`|`(objectClass=com.liferay.portlet.dynamicdatamapping.render.DDMFormFieldValueRenderer)`|
|`com.liferay.portlet.dynamicdatamapping.storage.StorageAdapter`|`(objectClass=com.liferay.portlet.dynamicdatamapping.storage.StorageAdapter)`|
|`com.liferay.portlet.social.model.SocialActivityInterpreter`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portlet.social.model.SocialActivityInterpreter))`|
|`com.liferay.portlet.social.model.SocialRequestInterpreter`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portlet.social.model.SocialRequestInterpreter))`|
|`com.liferay.portlet.ControlPanelEntry`|`(&(!(javax.portlet.name=*))(objectClass=com.liferay.portlet.ControlPanelEntry))`|
|`com.liferay.portal.kernel.portlet.FriendlyURLMapper`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portal.kernel.portlet.FriendlyURLMapper))`|
|`javax.portlet.filter.PortletFilter`|`(&(javax.portlet.name=*)(objectClass=javax.portlet.filter.PortletFilter))`|
|`com.liferay.portal.kernel.atom.AtomCollectionAdapter`|`(objectClass=com.liferay.portal.kernel.atom.AtomCollectionAdapter)`|
|`com.liferay.portal.kernel.format.PhoneNumberFormat`||
|`com.liferay.portal.kernel.lar.xstream.XStreamAliasRegistryUtil.XStreamAlias`||
|`com.liferay.portal.kernel.lar.xstream.XStreamConverter`||
|`com.liferay.portal.kernel.lar.StagedModelDataHandler`||
|`com.liferay.portal.kernel.lock.LockListener`||
|`com.liferay.portal.kernel.notifications.UserNotificationDefinition`||
|`com.liferay.portal.kernel.notifications.UserNotificationHandler`||
|`com.liferay.portal.kernel.portlet.bridges.mvc.ActionCommand`|`(&(action.command.name=*)(javax.portlet.name=*)(objectClass=com.liferay.portal.kernel.portlet.bridges.mvc.ActionCommand))`|
|`java.util.ResourceBundle`|`(&(javax.portlet.name=*)(objectClass=java.util.ResourceBundle))`|
|`com.liferay.portal.kernel.search.facet.util.FacetFactory`||
|`com.liferay.portal.kernel.search.Indexer`||
|`com.liferay.portal.kernel.search.SearchEngineConfigurator`||
|`javax.servlet.Filter`|`(&(objectClass=javax.servlet.Filter)(servlet-context-name=*)(servlet-filter-name=*))`|
|`com.liferay.portal.kernel.template.TemplateHandler`||
|`com.liferay.portal.kernel.trash.TrashHandler`||
|`com.liferay.portal.kernel.webdav.WebDAVStorage`||
|`com.liferay.portal.kernel.workflow.WorkflowHandler`||
|`com.liferay.portal.model.ModelListener`||
|`com.liferay.portal.security.auth.AuthToken`||
|`com.liferay.portlet.asset.model.AssetRendererFactory`||
|`com.liferay.portlet.dynamicdatamapping.util.DDMDisplay`||
|`com.liferay.portal.kernel.portlet.ConfigurationAction`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portal.kernel.portlet.ConfigurationAction))`|
|`com.liferay.portlet.ControlPanelEntry`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portlet.ControlPanelEntry))`|
|`com.liferay.portlet.expando.model.CustomAttributesDisplay`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portlet.expando.model.CustomAttributesDisplay))`|
|`com.liferay.portlet.dynamicdatamapping.util.DDMDisplay`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portlet.dynamicdatamapping.util.DDMDisplay))`|
|`com.liferay.portal.kernel.search.OpenSearch`|`(&(javax.portlet.name=*)(objectClass=com.liferay.portal.kernel.search.OpenSearch))`|

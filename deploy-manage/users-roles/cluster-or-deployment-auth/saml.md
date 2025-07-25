---
navigation_title: SAML
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/saml-realm.html
  - https://www.elastic.co/guide/en/cloud-enterprise/current/ece-sign-outgoing-saml-message.html
  - https://www.elastic.co/guide/en/cloud-enterprise/current/ece_sign_outgoing_saml_message.html
  - https://www.elastic.co/guide/en/cloud-enterprise/current/ece_optional_settings.html
  - https://www.elastic.co/guide/en/cloud-enterprise/current/ece-securing-clusters-SAML.html
  - https://www.elastic.co/guide/en/cloud/current/ec-securing-clusters-SAML.html
  - https://www.elastic.co/guide/en/cloud/current/ec-sign-outgoing-saml-message.html
  - https://www.elastic.co/guide/en/cloud-heroku/current/ech-securing-clusters-SAML.html
  - https://www.elastic.co/guide/en/cloud-heroku/current/echsign-outgoing-saml-message.html
  - https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-saml-authentication.html
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/saml-guide-stack.html
applies_to:
  deployment:
    self:
    ess:
    ece:
    eck:
products:
  - id: elasticsearch
  - id: cloud-enterprise
  - id: cloud-hosted
  - id: cloud-kubernetes
---

# SAML authentication [saml-realm]

The {{stack}} supports SAML single-sign-on (SSO) into {{kib}}, using {{es}} as a backend service.

The {{security-features}} provide this support using the Web Browser SSO profile of the SAML 2.0 protocol. This protocol is specifically designed to support authentication using an interactive web browser, so it does not operate as a standard authentication realm. Instead, there are {{kib}} and {{es}} {{security-features}} that work together to enable interactive SAML sessions.

This means that the SAML realm is not suitable for use by standard REST clients. If you configure a SAML realm for use in {{kib}}, you should also configure another realm, such as the [native realm](/deploy-manage/users-roles/cluster-or-deployment-auth/native.md) in your authentication chain.

Because this feature is designed with {{kib}} in mind, most sections of this guide assume {{kib}} is used. To learn how a custom web application could use the OpenID Connect REST APIs to authenticate the users to {{es}} with SAML, refer to [SAML without {{kib}}](#saml-no-kibana).

The SAML support in {{kib}} is designed with the expectation that it will be the primary (or sole) authentication method for users of that {{kib}} instance. After you enable SAML authentication in {{kib}}, it will affect all users who try to login. The [Configuring {{kib}}](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-configure-kibana) section provides more detail about how this works.

For a detailed walk-through of how to implement SAML authentication for {{kib}} with Microsoft Entra ID as an identity provider, refer to our guide [Set up SAML with Microsoft Entra ID](/deploy-manage/users-roles/cluster-or-deployment-auth/saml-entra.md).

To configure SAML, you need to perform the following steps:

1. [Configure the prerequisites](#prerequisites)
2. [Create one or more SAML realms](#saml-create-realm)
3. [Configure role mappings](#saml-role-mapping)
4. [Configure {{kib}} to use SAML as the authentication provider](#saml-configure-kibana)

Additional steps outlined in this document are optional.

::::{note}
{{stack}} SSO is a [subscription feature](https://www.elastic.co/subscriptions).
::::

::::{tip}
This topic describes implementing SAML SSO at the deployment or cluster level, for the purposes of authenticating with a {{kib}} instance.

Depending on your deployment type, you can also configure SSO for the following use cases:

* If you're using {{ech}} or {{serverless-full}}, then you can configure SAML SSO [at the organization level](/deploy-manage/users-roles/cloud-organization/configure-saml-authentication.md). SAML SSO configured at this level can be used to control access to both the {{ecloud}} Console and to specific {{ech}} deployments and {{serverless-full}} projects. [Learn more about deployment-level vs. organization-level SSO](/deploy-manage/users-roles/cloud-organization.md#organization-deployment-sso).
* If you're using {{ece}}, then you can configure SAML [at the installation level](/deploy-manage/users-roles/cloud-enterprise-orchestrator/saml.md), and then configure [SSO](/deploy-manage/users-roles/cloud-enterprise-orchestrator/configure-sso-for-deployments.md) for deployments.
::::

## Identity provider requirements [saml-guide-idp]

In SAML terminology, the {{stack}} is operating as a *Service Provider*.

The other component that is needed to enable SAML single-sign-on is the *Identity Provider*, which is a service that handles your credentials and performs that actual authentication of users.

If you are interested in configuring SSO into {{kib}}, then you need to provide {{es}} with information about your *Identity Provider*, and you will need to register the {{stack}} as a known *Service Provider* within that Identity Provider. There are also a few configuration changes that are required in {{kib}} to activate the SAML authentication provider.

### Supported IdPs

The {{stack}} supports the SAML 2.0 Web Browser SSO and Single Logout profiles, and can integrate with any Identity Provider (IdP) that supports at least the SAML 2.0 Web Browser SSO profile. It has been tested with a number of popular IdP implementations, such as [Microsoft Active Directory Federation Services (ADFS)](https://www.elastic.co/blog/how-to-configure-elasticsearch-saml-authentication-with-adfs), [Microsoft Entra ID](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md), and [Okta](https://www.elastic.co/blog/how-to-set-up-okta-saml-login-kibana-elastic-cloud).

### Required IdP information

The {{stack}} accepts a standard XML-formatted SAML *metadata* document, which defines the capabilities and features of your IdP. You should be able to download or generate such a document within your IdP administration interface. You can pass this IdP document as a URL, or download it and make the file available to {{es}} For more information, see [`idp.metadata.path`](#idp-metadata-path).

The IdP will have been assigned an identifier or *EntityID*, which is most commonly expressed in *Uniform Resource Identifier* (URI) form. Your admin interface might tell you what this is, or you might need to read the metadata document to find it - look for the `entityID` attribute on the `EntityDescriptor` element.

Most IdPs will provide an appropriate metadata file with all the features that the {{stack}} requires, and should only require the configuration steps described below. The minimum requirements that the {{stack}} has for the IdP’s metadata are:

* An `<EntityDescriptor>` with an `entityID` that matches the {{es}} [configuration](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-create-realm)
* An `<IDPSSODescriptor>` that supports the SAML 2.0 protocol (`urn:oasis:names:tc:SAML:2.0:protocol`).
* At least one `<KeyDescriptor>` that is configured for *signing* (that is, it has `use="signing"` or leaves the `use` unspecified)
* A `<SingleSignOnService>` with binding of HTTP-Redirect (`urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect`)
* If you want to support [Single Logout](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-logout), a `<SingleLogoutService>` with binding of HTTP-Redirect (`urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect`)

### Signing requirements

The {{stack}} requires that all messages from the IdP are signed. For authentication `<Response>` messages, the signature can be applied to either the response itself, or to the individual assertions. For `<LogoutRequest>` messages, the message itself must be signed, and the signature should be provided as a URL parameter, as required by the `HTTP-Redirect` binding.

## Prerequisites

Before you set up SAML single-sign on, you must have a [SAML IdP](#saml-guide-idp) configured.

If you're using a self-managed cluster, then perform the following additional steps:

* Enable TLS for HTTP.

    If your {{es}} cluster is operating in production mode, you must configure the HTTP interface to use SSL/TLS before you can enable SAML authentication. For more information, see [Encrypt HTTP client communications for {{es}}](/deploy-manage/security/set-up-basic-security-plus-https.md#encrypt-http-communication).

    If you started {{es}} [with security enabled](/deploy-manage/deploy/self-managed/installing-elasticsearch.md), then TLS is already enabled for HTTP.

    {{ech}}, {{ece}}, and {{eck}} have TLS enabled by default.

* Enable the token service.

    The {{es}} SAML implementation makes use of the {{es}} token service. If you configure TLS on the HTTP interface, this service is automatically enabled. It can be explicitly configured by adding the following setting in your [`elasticsearch.yml`](/deploy-manage/stack-settings.md) file:

    ```yaml
    xpack.security.authc.token.enabled: true
    ```

    {{ech}}, {{ece}}, and {{eck}} have TLS enabled by default.

## Create a SAML realm [saml-create-realm]

SAML authentication is enabled by configuring a SAML realm within the authentication chain for {{es}}.

This realm has a few mandatory settings, and a number of optional settings. The available settings are described in detail in [Security settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md):
* [SAML realm settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-saml-settings)
* [SAML realm signing settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-saml-signing-settings)
* [SAML realm encryption settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-saml-encryption-settings)
* [SAML realm SSL settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-saml-ssl-settings)

This guide will walk you through the most common settings.

Create a realm by adding the following to your [`elasticsearch.yml`](/deploy-manage/stack-settings.md) configuration file. Each configuration value is explained below.

If you're using {{ece}} or {{ech}}, and you're using machine learning or a deployment with hot-warm architecture, you must include this configuration in the user settings section for each node type.

```yaml
xpack.security.authc.realms.saml.saml1:
  order: 2
  idp.metadata.path: saml/idp-metadata.xml
  idp.entity_id: "<sso-example-url>"
  sp.entity_id:  "<kibana-example-url>"
  sp.acs: "<kibana-example-url>/api/security/saml/callback"
  sp.logout: "<kibana-example-url>/logout"
  attributes.principal: "urn:oid:0.9.2342.19200300.100.1.1"
  attributes.groups: "urn:oid:1.3.6.1.4.1.5923.1.5.1."
```

::::{dropdown} Common settings

xpack.security.authc.realms.saml.saml1
:   Defines a new `saml` authentication realm named "saml1". See [Realms](/deploy-manage/users-roles/cluster-or-deployment-auth/authentication-realms.md) for more explanation of realms.

order
:   The order of the realm within the realm chain. Realms with a lower order have highest priority and are consulted first. We recommend giving password-based realms such as file, native, LDAP, and Active Directory the lowest order (highest priority), followed by SSO realms such as SAML and OpenID Connect. If you have multiple realms of the same type, give the most frequently accessed realm the lowest order to have it consulted first.

  If you're using {{eck}}, then make sure not to disable Elasticsearch’s file realm set by ECK, as ECK relies on the file realm for its operation. Set the `order` setting of the SAML realm to a greater value than the `order` value set for the file and native realms, which is by default -100 and -99 respectively.

idp.metadata.path
:   $$$idp-metadata-path$$$ The path to the metadata file for your Identity Provider. The metadata file path can either be a path, or an HTTPS URL.

    :::{tip}
    If you want to pass a file path, then review the following:
    * File path settings are resolved relative to the {{es}} config directory. {{es}} will automatically monitor this file for changes and will reload the configuration whenever it is updated.
    * If you're using {{ech}} or {{ece}}, then you must upload the file before it can be referenced. For {{ech}}, upload the file [as a custom bundle](/deploy-manage/deploy/elastic-cloud/upload-custom-plugins-bundles.md). For {{ece}}, follow the equivalent [ECE procedure](/deploy-manage/deploy/cloud-enterprise/add-custom-bundles-plugins.md).
    * If you're using {{eck}}, then install the file as [custom configuration files](/deploy-manage/deploy/cloud-on-k8s/custom-configuration-files-plugins.md#use-a-volume-and-volume-mount-together-with-a-configmap-or-secret).
    :::

idp.entity_id
:   The identifier (SAML EntityID) that your IdP uses. It should match the `entityID` attribute within the metadata file.

sp.entity_id
:   A unique identifier for your {{kib}} instance, expressed as a URI. You will use this value when you add {{kib}} as a service provider within your IdP. We recommend that you use the base URL for your {{kib}} instance as the entity ID.

sp.acs
:   The *Assertion Consumer Service* (ACS) endpoint is the URL within {{kib}} that accepts authentication messages from the IdP. This ACS endpoint supports the SAML HTTP-POST binding only. It must be a URL that is accessible from the web browser of the user who is attempting to login to {{kib}}, it does not need to be directly accessible by {{es}} or the IdP. The correct value may vary depending on how you have installed {{kib}} and whether there are any proxies involved, but it will typically be `${kibana-url}/api/security/saml/callback` where *${kibana-url}* is the base URL for your {{kib}} instance.

sp.logout
:   The URL within {{kib}} that accepts logout messages from the IdP. Like the `sp.acs` URL, it must be accessible from the web browser, but does not need to be directly accessible by {{es}} or the IdP. The correct value may vary depending on how you have installed {{kib}} and whether there are any proxies involved, but it will typically be `${kibana-url}/logout` where *${kibana-url}* is the base URL for your {{kib}} instance.

attributes.principal
:   See [Attribute mapping](#saml-attributes-mapping).

attributes.groups
:   See [Attribute mapping](#saml-attributes-mapping).
::::

## Map SAML attributes to {{es}} attributes [saml-attributes-mapping]

When a user connects to {{kib}} through your Identity Provider, the Identity Provider will supply a SAML Assertion about the user. The assertion will contain an *Authentication Statement* indicating that the user has successfully authenticated to the IdP and one or more *Attribute Statements* that will include *Attributes* for the user.

These attributes might include information like:

* The user’s username
* The user’s email address
* The user’s groups or roles

Attributes in SAML are [usually](#saml-attribute-mapping-nameid) named using a URI such as `urn:oid:0.9.2342.19200300.100.1.1` or `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn`, and have one or more values associated with them.

These attribute identifiers vary between IdPs, and most IdPs offer ways to customize the URIs and their associated value.

{{es}} uses these attributes to infer information about the user who has logged in, and they can be used for [role mapping](#saml-role-mapping).

### How attributes appear in user metadata [saml-user-metadata]

By default users who authenticate via SAML will have some additional metadata fields.

* `saml_nameid` will be set to the value of the `NameID` element in the SAML authentication response
* `saml_nameid_format` will be set to the full URI of the NameID’s `format` attribute
* Every SAML Attribute that is provided in the authentication response (regardless of whether it is mapped to an {{es}} user property), will be added as the metadata field `saml(name)` where "name" is the full URI name of the attribute. For example `saml(urn:oid:0.9.2342.19200300.100.1.3)`.
* For every SAML Attribute that has a *friendlyName*, will also be added as the metadata field `saml_friendlyName` where "name" is the full URI name of the attribute. For example `saml_mail`.

This behavior can be disabled by adding `populate_user_metadata: false` to as a setting in the saml realm.

### Map attributes

In order for SAML attributes to be useful in {{es}}, {{es}} and the IdP need to have a common value for the names of the attributes. This is done manually, by configuring the IdP and the SAML realm to use the same URI name for each logical user attribute.

The recommended steps for configuring these SAML attributes are as follows:

1. Consult your IdP to see what user attributes it can provide. This varies greatly between providers, but you should be able to obtain a list from the documentation, or from your local admin.
2. Review the list of [user properties](#saml-es-user-properties) that {{es}} supports, and decide which of them are useful to you, and can be provided by your IdP. At a *minimum*, the `principal` attribute is required.
3. Configure your IdP to "release" those attributes to your {{kib}} SAML service provider. This process varies by provider: some will provide a user interface for this, while others may require that you edit configuration files.

   Because {{es}} does not require that any specific URIs are used, you can use any URIs to use for each attribute as they are recommended by the IDP or your local administrator.
4. Configure the SAML realm in {{es}} to associate the [{{es}} user properties](#saml-es-user-properties) to the URIs that you configured in your IdP. The [sample configuration](#saml-create-realm) configures the `principal` and `groups` attributes.

### Special attribute names [saml-attribute-mapping-nameid]

In general, {{es}} expects that the configured value for an attribute will be a URI, such as `urn:oid:0.9.2342.19200300.100.1.1`. However, there are some additional names that can be used:

`nameid`
:   This uses the SAML `NameID` value (all leading and trailing whitespace removed) instead of a SAML attribute. SAML `NameID` elements are an optional, but frequently provided, field within a SAML Assertion that the IdP may use to identify the Subject of that Assertion. In some cases the `NameID` will relate to the user’s login identifier (username) within the IdP, but in many cases they will be internally generated identifiers that have no obvious meaning outside of the IdP.

`nameid:persistent`
:   This uses the SAML `NameID` value (all leading and trailing whitespace removed), but only if the NameID format is `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`. A SAML `NameID` element has an optional `Format` attribute that indicates the semantics of the provided name. It is common for IdPs to be configured with "transient" NameIDs that present a new identifier for each session. Since it is rarely useful to use a transient NameID as part of an attribute mapping, the "nameid:persistent" attribute name can be used as a safety mechanism that will cause an error if you attempt to map from a `NameID` that does not have a persistent value.

:::{note}
Identity Providers can be either statically configured to release a `NameID` with a specific format, or they can be configured to try to conform with the requirements of the SP. The SP declares its requirements as part of the Authentication Request, using an element which is called the `NameIDPolicy`. If this is needed, you can set the relevant [settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-saml-settings) named `nameid_format` in order to request that the IdP releases a `NameID` with a specific format.
:::

*friendlyName*
:   A SAML attribute may have a *friendlyName* in addition to its URI based name. For example the attribute with a name of `urn:oid:0.9.2342.19200300.100.1.1` might also have a friendlyName of `uid`. You may use these friendly names within an attribute mapping, but it is recommended that you use the URI based names, as friendlyNames are neither standardized or mandatory.

The example below configures a realm to use a persistent nameid for the principal, and the attribute with the friendlyName "roles" for the user’s groups.

```yaml
xpack.security.authc.realms.saml.saml1:
  order: 2
  idp.metadata.path: saml/idp-metadata.xml
  idp.entity_id: "https://sso.example.com/"
  sp.entity_id:  "https://kibana.example.com/"
  sp.acs: "https://kibana.example.com/api/security/saml/callback"
  attributes.principal: "nameid:persistent"
  attributes.groups: "roles"
  nameid_format: "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent"
```


### Mappable {{es}} user properties [saml-es-user-properties]

The {{es}} SAML realm can be configured to map SAML `attributes` to the following properties on the authenticated user:

principal
:   *(Required)* This is the *username* that will be applied to a user that authenticates against this realm. The `principal` appears in places such as the {{es}} audit logs.

groups
:   *(Recommended)* If you want to use your IdP’s concept of groups or roles as the basis for a user’s {{es}} privileges, you should map them with this attribute. The `groups` are passed directly to your [role mapping rules](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-role-mapping).

    :::{note}
    Some IdPs are configured to send the `groups` list as a single value, comma-separated string. To map this SAML attribute to the `attributes.groups` setting in the {{es}} realm, you can configure a string delimiter using the `attribute_delimiters.groups` setting.<br><br>For example, splitting the SAML attribute value `engineering,elasticsearch-admins,employees` on a delimiter value of `,` will result in `engineering`, `elasticsearch-admins`, and `employees` as the list of groups for the user.
    ::::

name
:   *(Optional)* The user’s full name.

mail
:   *(Optional)* The user’s email address.

dn
:   *(Optional)* The user’s X.500 *Distinguished Name*.


### Extract partial values from SAML attributes [_extracting_partial_values_from_saml_attributes]

There are some occasions where the IdP’s attribute may contain more information than you want to use within {{es}}. A common example of this is one where the IdP works exclusively with email addresses, but you want the user’s `principal` to use the `local-name` part of the email address. For example if their email address was `james.wong@staff.example.com`, then you might want their principal to be `james.wong`.

This can be achieved using the `attribute_patterns` setting in the {{es}} realm, as demonstrated in the realm configuration below:

```yaml
xpack.security.authc.realms.saml.saml1:
  order: 2
  idp.metadata.path: saml/idp-metadata.xml
  idp.entity_id: "https://sso.example.com/"
  sp.entity_id:  "https://kibana.example.com/"
  sp.acs: "https://kibana.example.com/api/security/saml/callback"
  attributes.principal: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
  attribute_patterns.principal: "^([^@]+)@staff\\.example\\.com$"
```

In this case, the user’s `principal` is mapped from an email attribute, but a regular expression is applied to the value before it is assigned to the user. If the regular expression matches, then the result of the first group is used as effective value. If the regular expression does not match then the attribute mapping fails.

In this example, the email address must belong to the `staff.example.com` domain, and then the local-part (anything before the `@`) is used as the principal. Any users who try to login using a different email domain will fail because the regular expression will not match against their email address, and thus their principal attribute - which is mandatory - will not be populated.

::::{important}
Small mistakes in these regular expressions can have significant security consequences. For example, if we accidentally left off the trailing `$` from the example above, then we would match any email address where the domain starts with `staff.example.com`, and this would accept an email address such as `admin@staff.example.com.attacker.net`. It is important that you make sure your regular expressions are as precise as possible so that you don't open an avenue for user impersonation attacks.
::::

## Request specific authentication methods [req-authn-context]

It is sometimes necessary for a SAML SP to be able to impose specific restrictions regarding the authentication that will take place at an IdP, in order to assess the level of confidence that it can place in the corresponding authentication response. The restrictions might have to do with the authentication method (password, client certificates, etc), the user identification method during registration, and other details. {{es}} implements [SAML 2.0 Authentication Context](https://docs.oasis-open.org/security/saml/v2.0/saml-authn-context-2.0-os.pdf), which can be used for this purpose as defined in SAML 2.0 Core Specification.

The SAML SP defines a set of Authentication Context Class Reference values, which describe the restrictions to be imposed on the IdP, and sends these in the Authentication Request. The IdP attempts to grant these restrictions. If it cannot grant them, the authentication attempt fails. If the user is successfully authenticated, the Authentication Statement of the SAML Response contains an indication of the restrictions that were satisfied.

You can define the Authentication Context Class Reference values by using the `req_authn_context_class_ref` option in the SAML realm configuration. See [SAML realm settings](elasticsearch://reference/elasticsearch/configuration-reference/security-settings.md#ref-saml-settings).

{{es}} supports only the `exact` comparison method for the Authentication Context. When it receives the Authentication Response from the IdP, {{es}} examines the value of the Authentication Context Class Reference that is part of the Authentication Statement of the SAML Assertion. If it matches one of the requested values, the authentication is considered successful. Otherwise, the authentication attempt fails.

## Configure SAML logout [saml-logout]

The SAML protocol supports the concept of Single Logout (SLO). The level of support for SLO varies between Identity Providers. You should consult the documentation for your IdP to determine what Logout services it offers.

By default, the {{stack}} will support SAML SLO if the following are true:

* Your IdP metadata specifies that the IdP offers a SLO service
* Your IdP releases a NameID in the subject of the SAML assertion that it issues for your users
* You configure `sp.logout`
* The setting `idp.use_single_logout` is not `false`

### IdP SLO service [_idp_slo_service]

One of the values that {{es}} reads from the IdP’s SAML metadata is the `<SingleLogoutService>`. For Single Logout to work with the {{stack}}, {{es}} requires that this exist and support a binding of `urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect`.

The {{stack}} will send both `<LogoutRequest>` and `<LogoutResponse>` messages to this service as appropriate.


### The sp.logout setting [_the_sp_logout_setting]

The {{es}} realm setting `sp.logout` specifies a URL in {{kib}} to which the IdP can send both `<LogoutRequest>` and `<LogoutResponse>` messages. This service uses the SAML HTTP-Redirect binding.

{{es}} will process `<LogoutRequest>` messages, and perform a global signout that invalidates any existing {{es}} security tokens that are associated with the provided SAML session.

If you don't configure a value for `sp.logout`, {{es}} will refuse all `<LogoutRequest>` messages.

::::{note}
It is common for IdPs to require that `LogoutRequest` messages be signed, so you may need to configure [signing credentials](#saml-enc-sign).
::::

### The idp.use_single_logout setting [_the_idp_use_single_logout_setting]

If your IdP provides a `<SingleLogoutService>` but you do not want to use it, you can configure `idp.use_single_logout: false` in your SAML realm, and {{es}} will ignore the SLO service that your IdP provides. In this case, when a user logs out of {{kib}} it will invalidate their {{es}} session (security token), but will not perform any logout at the IdP.


### Using {{kib}} without single logout [_using_kib_without_single_logout]

If your IdP does not support Single Logout, or you choose not to use it, then {{kib}} will perform a "local logout" only.

This means that {{kib}} will invalidate the session token it is using to communicate with {{es}}, but will not be able to perform any sort of invalidation of the Identity Provider session. In most cases, this will mean that {{kib}} users are still considered to be logged in to the IdP. Consequently, if the user navigates to the {{kib}} landing page, they will be automatically reauthenticated, and will commence a new {{kib}} session without needing to enter any credentials.

The possible solutions to this problem are:

* Ask your IdP administrator or vendor to provide a Single Logout service
* If your Idp does provide a Single Logout Service, make sure it is included in the IdP metadata file, and do *not* set `idp.use_single_logout` to `false`.
* Advise your users to close their browser after logging out of {{kib}}
* Enable the `force_authn` setting on your SAML realm. This setting causes the {{stack}} to request fresh authentication from the IdP every time a user attempts to log in to {{kib}}. This setting defaults to `false` because it can be a more cumbersome user experience, but it can also be an effective protection to stop users piggy-backing on existing IdP sessions.

## Encryption and signing [saml-enc-sign]

The {{stack}} supports generating signed SAML messages (for authentication and/or logout), verifying signed SAML messages from the IdP (for both authentication and logout) and can process encrypted content.

You can configure {{es}} for signing, encryption or both, using a single key or individual keys.

The {{stack}} uses X.509 certificates with RSA private keys for SAML cryptography. These keys can be generated using any standard SSL tool, including the `elasticsearch-certutil` tool.

Your IdP may require that the {{stack}} have a cryptographic key for signing SAML messages, and that you provide the corresponding signing certificate within the Service Provider configuration (either within the {{stack}} SAML metadata file, or manually configured within the IdP administration interface).

While most IdPs do not expect authentication requests to be signed, it is commonly the case that signatures are required for logout requests. Your IdP will validate these signatures against the signing certificate that has been configured for the {{stack}} Service Provider.

Encryption certificates are rarely needed, but the {{stack}} supports them for cases where IdPs or local policies mandate their use.

### Generate certificates and keys [_generating_certificates_and_keys]

{{es}} supports certificates and keys in either PEM, PKCS#12 or JKS format. Some Identity Providers are more restrictive in the formats they support, and will require you to provide the certificates as a file in a particular format. You should consult the documentation for your IdP to determine what formats they support.

#### Example: Using `openssl`

```sh
openssl req -new -x509 -days 3650 -nodes -sha256 -out saml-sign.crt -keyout saml-sign.key
```

#### Example: Using `elasticsearch-certutil`

```{applies_to}
deployment:
  self:
```

Using the [`elasticsearch-certutil` tool](elasticsearch://reference/elasticsearch/command-line-tools/certutil.md), you can generate a signing certificate with the following command. Because PEM format is the most commonly supported format, the example generates a certificate in that format.

```sh
bin/elasticsearch-certutil cert --self-signed --pem --days 1100 --name saml-sign --out saml-sign.zip
```

This will do the following:

* generate a certificate and key pair (the `cert` subcommand)
* create the files in PEM format (`-pem` option)
* generate a certificate that is valid for 3 years (`-days 1100`)
* name the certificate `saml-sign` (`-name` option)
* save the certificate and key in the `saml-sign.zip` file (`-out` option)

The generated zip archive will contain 3 files:

* `saml-sign.crt`, the public certificate to be used for signing
* `saml-sign.key`, the private key for the certificate
* `ca.crt`, a CA certificate that is not need, and can be ignored.

Encryption certificates can be generated with the same process.


### Sign outgoing SAML messages [_configuring_es_for_signing]

By default, {{es}} will sign *all* outgoing SAML messages if a signing certificate and key has been configured.

:::{tip}
* In self-managed clusters, file path settings is resolved relative to the {{es}} config directory. {{es}} will automatically monitor this file for changes and will reload the configuration whenever it is updated.
* If you're using {{ech}} or {{ece}}, then you must upload any certificate or keystore files before they can be referenced in the configuration. For {{ech}}, upload them [as a custom bundle](/deploy-manage/deploy/elastic-cloud/upload-custom-plugins-bundles.md). For {{ece}}, follow the equivalent [ECE procedure](/deploy-manage/deploy/cloud-enterprise/add-custom-bundles-plugins.md). In both cases, you can add the files to your existing SAML bundle.
* If you're using {{eck}}, then install the files as [custom configuration files](/deploy-manage/deploy/cloud-on-k8s/custom-configuration-files-plugins.md#use-a-volume-and-volume-mount-together-with-a-configmap-or-secret).
:::

::::{tab-set}
:::{tab-item} PEM formatted keys

If you want to use **PEM formatted** keys and certificates for signing, then you should configure the following settings on the SAML realm:

`signing.certificate`
:   The path to the PEM formatted certificate file. e.g. `saml/saml-sign.crt`

`signing.key`
:   The path to the PEM formatted key file. e.g. `saml/saml-sign.key`

`signing.secure_key_passphrase`
:   The passphrase for the key, if the file is encrypted. This is a secure setting that must be uploaded to your [{{es}} keystore](/deploy-manage/security/secure-settings.md).

:::
:::{tab-item} PKCS#12 or Java Keystore
If you want to use **PKCS#12 formatted** files or a **Java Keystore** for signing, then you should configure the following settings on the SAML realm:

`signing.keystore.path`
:   The path to the PKCS#12 or JKS keystore. e.g. `saml/saml-sign.p12`

`signing.keystore.alias`
:   The alias of the key within the keystore. e.g. `signing-key`

`signing.keystore.secure_password`
:   The passphrase for the keystore, if the file is encrypted. This is a secure setting that must be uploaded to your [{{es}} keystore](/deploy-manage/security/secure-settings.md).
:::
::::

#### Sign only certain message types

If you want to sign some, but not all outgoing **SAML messages**, then configure `signing.saml_messages` with a comma separated list of message types to sign. Supported values are `AuthnRequest`, `LogoutRequest` and `LogoutResponse` and the default value is `*`.

For example:

```sh
xpack:
  security:
    authc:
      realms:
        saml-realm-name:
          order: 2
          ...
          signing.saml_messages: AuthnRequest <1>
          ...
```

1. This configuration ensures that only SAML authentication requests will be sent signed to the Identity Provider.


### Configuring {{es}} for encrypted messages [_configuring_es_for_encrypted_messages]

The {{es}} {{security-features}} support a single key for message decryption. If a key is configured, then {{es}} attempts to use it to decrypt `EncryptedAssertion` and `EncryptedAttribute` elements in Authentication responses, and `EncryptedID` elements in Logout requests.

{{es}} rejects any SAML message that contains an `EncryptedAssertion` that cannot be decrypted.

If an `Assertion` contains both encrypted and plain-text attributes, then failure to decrypt the encrypted attributes will not cause an automatic rejection. Rather, {{es}} processes the available plain-text attributes (and any `EncryptedAttributes` that could be decrypted).

:::{tip}
* In self-managed clusters, file path settings is resolved relative to the {{es}} config directory. {{es}} will automatically monitor this file for changes and will reload the configuration whenever it is updated.
* If you're using {{ech}} or {{ece}}, then you must upload any certificate or keystore files before they can be referenced in the configuration. For {{ech}}, upload them [as a custom bundle](/deploy-manage/deploy/elastic-cloud/upload-custom-plugins-bundles.md). For {{ece}}, follow the equivalent [ECE procedure](/deploy-manage/deploy/cloud-enterprise/add-custom-bundles-plugins.md). In both cases, you can add the files to your existing SAML bundle.
* If you're using {{eck}}, then install the files as [custom configuration files](/deploy-manage/deploy/cloud-on-k8s/custom-configuration-files-plugins.md#use-a-volume-and-volume-mount-together-with-a-configmap-or-secret).
:::

::::{tab-set}
:::{tab-item} PEM-formatted keys

If you want to use **PEM formatted** keys and certificates for SAML encryption, then you should configure the following settings on the SAML realm:

`encryption.certificate`
:   The path to the PEM formatted certificate file. e.g. `saml/saml-crypt.crt`

`encryption.key`
:   The path to the PEM formatted key file. e.g. `saml/saml-crypt.key`

`encryption.secure_key_passphrase`
:   The passphrase for the key, if the file is encrypted. This is a secure setting that must be uploaded to your [{{es}} keystore](/deploy-manage/security/secure-settings.md).

:::
:::{tab-item} PKCS#12 or Java Keystore

If you want to use **PKCS#12 formatted** files or a **Java Keystore** for SAML encryption, then you should configure the following settings on the SAML realm:

`encryption.keystore.path`
:   The path to the PKCS#12 or JKS keystore. e.g. `saml/saml-crypt.p12`

`encryption.keystore.alias`
:   The alias of the key within the keystore. e.g. `encryption-key`

`encryption.keystore.secure_password`
:   The passphrase for the keystore, if the file is encrypted. This is a secure setting that must be uploaded to your [{{es}} keystore](/deploy-manage/security/secure-settings.md).

:::
::::

## Generate SAML metadata for the Service Provider [saml-sp-metadata]

Some Identity Providers support importing a metadata file from the Service Provider. This will automatically configure many of the integration options between the IdP and the SP.

The {{stack}} supports generating such a metadata file using the [SAML service provider metadata API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-saml-service-provider-metadata) or the [`bin/elasticsearch-saml-metadata` command](elasticsearch://reference/elasticsearch/command-line-tools/saml-metadata.md).

### Using the SAML service provider metadata API

You can generate the SAML metadata by issuing the API request to {{es}} and store it as an XML file using tools like `jq`. For example, the following command generates the metadata for the SAML realm `realm1` and saves it to a `metadata.xml` file:

```console
curl -u user_name:password  -X GET http://localhost:9200/_security/saml/metadata/saml1 -H 'Content-Type: application/json' | jq -r '.[]' > metadata.xml
```

### Using the `elasticsearch-saml-metadata` command

You can generate the SAML metadata by running the [`bin/elasticsearch-saml-metadata` command](elasticsearch://reference/elasticsearch/command-line-tools/saml-metadata.md).

```{applies_to}
deployment:
  self:
  eck:
```

::::{tab-set}
::: {tab-item} Self-managed
```sh
bin/elasticsearch-saml-metadata --realm saml1
```
:::
::: {tab-item} ECK
To generate the Service Provider metadata using the `elasticsearch-saml-metadata` command in {{eck}}, you need to run the command using `kubectl`, and then copy the generated metadata file to your local machine. For example:

```sh
# Create metadata
kubectl exec -it elasticsearch-sample-es-default-0 -- sh -c "/usr/share/elasticsearch/bin/elasticsearch-saml-metadata --realm saml1"

# Copy metadata file
kubectl cp elasticsearch-sample-es-default-0:/usr/share/elasticsearch/saml-elasticsearch-metadata.xml saml-elasticsearch-metadata.xml
```
:::
::::



## Configure role mappings [saml-role-mapping]

When a user authenticates using SAML, they are identified to the {{stack}}, but this does not automatically grant them access to perform any actions or access any data.

Your SAML users cannot do anything until they are assigned roles. This can be done through either the [add role mapping API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-put-role-mapping) or with [authorization realms](/deploy-manage/users-roles/cluster-or-deployment-auth/realm-chains.md#authorization_realms).

You can map SAML users to roles in the following ways:

* Using the role mappings page in {{kib}}.
* Using the [role mapping API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-put-role-mapping).
* By delegating authorization to another realm.

::::{note}
You can't use [role mapping files](/deploy-manage/users-roles/cluster-or-deployment-auth/mapping-users-groups-to-roles.md#mapping-roles-file) to grant roles to users authenticating using SAML.
::::

### Example: Using the role mapping API

This is an example of a simple role mapping that grants the `example_role` role to any user who authenticates against the `saml1` realm:

```console
PUT /_security/role_mapping/saml-example
{
  "roles": [ "example_role" ], <1>
  "enabled": true,
  "rules": {
    "field": { "realm.name": "saml1" }
  }
}
```

1. The `example_role` role is **not** a builtin {{es}} role. This example assumes that you have created a custom role of your own, with appropriate access to your [data streams, indices,](/deploy-manage/users-roles/cluster-or-deployment-auth/role-structure.md#roles-indices-priv) and [{{kib}} features](/deploy-manage/users-roles/cluster-or-deployment-auth/kibana-privileges.md#kibana-feature-privileges).

### Example: Role mapping API, using SAML attributes

The attributes that are mapped via the realm configuration are used to process role mapping rules, and these rules determine which roles a user is granted.

The user fields that are provided to the role mapping are derived from the SAML attributes as follows:

* `username`: The `principal` attribute
* `dn`: The `dn` attribute
* `groups`: The `groups` attribute
* `metadata`: See [User metadata](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-user-metadata)

If your IdP has the ability to provide groups or roles to Service Providers, then you should map this SAML attribute to the `attributes.groups` setting in the {{es}} realm, and then make use of it in a role mapping.

For example:

This mapping grants the {{es}} `finance_data` role, to any users who authenticate via the `saml1` realm with the `finance-team` group.

```console
PUT /_security/role_mapping/saml-finance
{
  "roles": [ "finance_data" ],
  "enabled": true,
  "rules": { "all": [
        { "field": { "realm.name": "saml1" } },
        { "field": { "groups": "finance-team" } } <1>
  ] }
}
```

1. The `groups` attribute supports using wildcards (`*`). Refer to the [create or update role mappings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-put-role-mapping) for more information.

### Delegating SAML authorization to another realm

If your users also exist in a repository that can be directly accessed by {{es}} (such as an LDAP directory) then you can use [authorization realms](/deploy-manage/users-roles/cluster-or-deployment-auth/realm-chains.md#authorization_realms) instead of role mappings.

In this case, you perform the following steps:

1. In your SAML realm, assigned a SAML attribute to act as the lookup userid, by configuring the `attributes.principal` setting.
2. Create a new realm that can lookup users from your local repository (e.g. an `ldap` realm)
3. In your SAML realm, set `authorization_realms` to the name of the realm you created in step 2.

## Configure {{kib}} [saml-configure-kibana]

SAML authentication in {{kib}} requires additional settings in addition to the standard {{kib}} security configuration.

If you're using a self-managed cluster, then, because OIDC requires {{es}} nodes to use TLS on the HTTP interface, you must configure {{kib}} to use a `https` URL to connect to {{es}}, and you may need to configure `elasticsearch.ssl.certificateAuthorities` to trust the certificates that {{es}} has been configured to use.

SAML authentication in {{kib}} is subject to the following timeout settings in [`kibana.yml`](/deploy-manage/stack-settings.md):

* [`xpack.security.session.idleTimeout`](/deploy-manage/security/kibana-session-management.md#session-idle-timeout)
* [`xpack.security.session.lifespan`](/deploy-manage/security/kibana-session-management.md#session-lifespan)

You may want to adjust these timeouts based on your security requirements.

### Add the SAML provider to {{kib}}

::::{tip}
You can configure multiple authentication providers in {{kib}} and let users choose the provider they want to use. For more information, check [the {{kib}} authentication documentation](/deploy-manage/users-roles/cluster-or-deployment-auth/user-authentication.md).
::::

The three additional settings that are required for SAML support are shown below:

```yaml
xpack.security.authc.providers:
  saml.saml1:
    order: 0
    realm: "saml1"
```

The configuration values used in the example above are:

`xpack.security.authc.providers`
:   Add `saml` provider to instruct {{kib}} to use SAML SSO as the authentication method.

`xpack.security.authc.providers.saml.<provider-name>.realm`
:   Set this to the name of the SAML realm that you have used in your [{{es}} realm configuration](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-create-realm), for instance: `saml1`

### Supporting SAML and basic authentication in {{kib}} [saml-kibana-basic]

The SAML support in {{kib}} is designed on the expectation that it will be the primary (or sole) authentication method for users of that {{kib}} instance. However, it is possible to support both SAML and Basic authentication within a single {{kib}} instance by setting `xpack.security.authc.providers` as per the example below:

```yaml
xpack.security.authc.providers:
  saml.saml1:
    order: 0
    realm: "saml1"
  basic.basic1:
    order: 1
```

If {{kib}} is configured in this way, users are presented with a choice at the Login Selector UI. They log in with SAML or they provide a username and password and rely on one of the other security realms within {{es}}. Only users who have a username and password for a configured {{es}} authentication realm can log in via {{kib}} login form.

Alternatively, when the `basic` authentication provider is enabled, you can place a reverse proxy in front of {{kib}}, and configure it to send a basic authentication header (`Authorization: Basic ....`) for each request. If this header is present and valid, {{kib}} will not initiate the SAML authentication process.


### Operating multiple {{kib}} instances [_operating_multiple_kib_instances]

If you want to have multiple {{kib}} instances that authenticate against the same {{es}} cluster, then each {{kib}} instance that is configured for SAML authentication, requires its own SAML realm.

Each SAML realm must have its own unique Entity ID (`sp.entity_id`), and its own *Assertion Consumer Service* (`sp.acs`). Each {{kib}} instance will be mapped to the correct realm by looking up the matching `sp.acs` value.

These realms may use the same Identity Provider, but are not required to.

The following is example of having 3 different {{kib}} instances, 2 of which use the same internal IdP, and another which uses a different IdP.

```yaml
xpack.security.authc.realms.saml.saml_finance:
  order: 2
  idp.metadata.path: saml/idp-metadata.xml
  idp.entity_id: "<sso-example-url>"
  sp.entity_id:  "<kibana-finance-example-url>"
  sp.acs: "<kibana-finance-example-url>/api/security/saml/callback"
  sp.logout: "<kibana-finance-example-url>/logout"
  attributes.principal: "urn:oid:0.9.2342.19200300.100.1.1"
  attributes.groups: "urn:oid:1.3.6.1.4.1.5923.1.5.1."
xpack.security.authc.realms.saml.saml_sales:
  order: 3
  idp.metadata.path: saml/idp-metadata.xml
  idp.entity_id: "<sso-example-url>"
  sp.entity_id:  "<kibana-sales-example-url>"
  sp.acs: "<kibana-sales-example-url>/api/security/saml/callback"
  sp.logout: "<kibana-sales-example-url>/logout"
  attributes.principal: "urn:oid:0.9.2342.19200300.100.1.1"
  attributes.groups: "urn:oid:1.3.6.1.4.1.5923.1.5.1."
xpack.security.authc.realms.saml.saml_eng:
  order: 4
  idp.metadata.path: saml/idp-external.xml
  idp.entity_id: "<engineering-sso-example-url>"
  sp.entity_id:  "<kibana-engineering-example-url>"
  sp.acs: "<kibana-engineering-example-url>/api/security/saml/callback"
  sp.logout: "<kibana-engineering-example-url>/logout"
  attributes.principal: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"
```

It is possible to have one or more {{kib}} instances that use SAML, while other instances use basic authentication against another realm type (e.g. [Native](/deploy-manage/users-roles/cluster-or-deployment-auth/native.md) or [LDAP](/deploy-manage/users-roles/cluster-or-deployment-auth/ldap.md)).

## Troubleshooting SAML realm configuration [saml-troubleshooting]

The SAML 2.0 specification offers a lot of options and flexibility for the implementers of the standard which in turn adds to the complexity and the number of configuration options that are available both at the Service Provider ({{stack}}) and at the Identity Provider. Additionally, different security domains have different security requirements that need specific configuration to be satisfied. A conscious effort has been made to mask this complexity with sane defaults and the detailed documentation above but in case you encounter issues while configuring a SAML realm, you can look through our [SAML troubleshooting documentation](../../../troubleshoot/elasticsearch/security/trb-security-saml.md) that has suggestions and resolutions for common issues.


## SAML without {{kib}} [saml-no-kibana]

The SAML realm in {{es}} is designed to allow users to authenticate to {{kib}} and as such, most of the parts of the guide above make the assumption that {{kib}} is used. This section describes how a custom web application could use the relevant SAML REST APIs in order to authenticate the users to {{es}} with SAML.

::::{note}
This section assumes that you are familiar with the SAML 2.0 standard and more specifically with the SAML 2.0 Web Browser Single Sign On profile.
::::

Single sign-on realms such as OpenID Connect and SAML make use of the Token Service in {{es}} and in principle exchange a SAML or OpenID Connect Authentication response for an {{es}} access token and a refresh token. The access token is used as credentials for subsequent calls to {{es}}. The refresh token enables the user to get new {{es}} access tokens after the current one expires.

### SAML realm [saml-no-kibana-realm]

You must create a SAML realm and configure it accordingly in {{es}}. See [Configure {{es}} for SAML authentication](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-create-realm)


### Service Account user for accessing the APIs [saml-no-kibana-user]

The realm is designed with the assumption that there needs to be a privileged entity acting as an authentication proxy. In this case, the custom web application is the authentication proxy handling the authentication of end users (more correctly, "delegating" the authentication to the SAML Identity Provider). The SAML related APIs require authentication and the necessary authorization level for the authenticated user. For this reason, you must create a Service Account user and assign it a role that gives it the `manage_saml` cluster privilege. The use of the `manage_token` cluster privilege will be necessary after the authentication takes place, so that the service account user can maintain access in order refresh access tokens on behalf of the authenticated users or to subsequently log them out.

```console
POST /_security/role/saml-service-role
{
  "cluster" : ["manage_saml", "manage_token"]
}
```

```console
POST /_security/user/saml-service-user
{
  "password" : "<somePasswordHere>",
  "roles"    : ["saml-service-role"]
}
```


### Handling the SP-initiated authentication flow [saml-no-kibana-sp-init-sso]

On a high level, the custom web application would need to perform the following steps in order to authenticate a user with SAML against {{es}}:

1. Make an HTTP POST request to `_security/saml/prepare`, authenticating as the `saml-service-user` user. Use either the name of the SAML realm in the {{es}} configuration or the value for the Assertion Consumer Service URL in the request body. See the [SAML prepare authentication API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-saml-prepare-authentication) for more details.

    ```console
    POST /_security/saml/prepare
    {
      "realm" : "saml1"
    }
    ```

2. Handle the response from `/_security/saml/prepare`. The response from {{es}} will contain 3 parameters: `redirect`, `realm` and `id`. The custom web application would need to store the value for `id` in the user’s session (client side in a cookie or server side if session information is persisted this way). It must also redirect the user’s browser to the URL that was returned in the `redirect` parameter. The `id` value should not be disregarded as it is used as a nonce in SAML in order to mitigate against replay attacks.
3. Handle a subsequent response from the SAML IdP. After the user is successfully authenticated with the Identity Provider they will be redirected back to the Assertion Consumer Service URL. This `sp.acs` needs to be defined as a URL which the custom web application handles. When it receives this HTTP POST request, the custom web application must parse it and make an HTTP POST request itself to the `_security/saml/authenticate` API. It must authenticate as the `saml-service-user` user and pass the Base64 encoded SAML Response that was sent as the body of the request. It must also pass the value for `id` that it had saved in the user’s session previously.

    See [SAML authenticate API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-saml-authenticate) for more details.

    ```console
    POST /_security/saml/authenticate
    {
      "content" : "PHNhbWxwOlJlc3BvbnNlIHhtbG5zOnNhbWxwPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6cHJvdG9jb2wiIHhtbG5zOnNhbWw9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMD.....",
      "ids" : ["4fee3b046395c4e751011e97f8900b5273d56685"]
    }
    ```

    {{es}} will validate this and if all is correct will respond with an access token that can be used as a `Bearer` token for subsequent requests. It also supplies a refresh token that can be later used to refresh the given access token as described in [get token API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-get-token).

4. The response to calling `/_security/saml/authenticate` will contain only the username of the authenticated user. If you need to get the values for the SAML Attributes that were contained in the SAML Response for that user, you can call the Authenticate API `/_security/_authenticate/` using the access token as a `Bearer` token and the SAML attribute values will be contained in the response as part of the [User metadata](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-user-metadata).


### Handling the IdP-initiated authentication flow [saml-no-kibana-idp-init-sso]

{{es}} can also handle the IdP-initiated Single Sign On flow of the SAML 2 Web Browser SSO profile. In this case the authentication starts with an unsolicited authentication response from the SAML Identity Provider. The difference with the [SP initiated SSO](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-no-kibana-sp-init-sso) is that the web application needs to handle requests to the `sp.acs` that will not come as responses to previous redirections. As such, it will not have a session for the user already, and it will not have any stored values for the `id` parameter. The request to the `_security/saml/authenticate` API will look like the one below in this case:

```console
POST /_security/saml/authenticate
{
  "content" : "PHNhbWxwOlJlc3BvbnNlIHhtbG5zOnNhbWxwPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6cHJvdG9jb2wiIHhtbG5zOnNhbWw9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMD.....",
  "ids" : []
}
```


### Handling the logout flow [saml-no-kibana-slo]

1. At some point, if necessary, the custom web application can log the user out by using the [SAML logout API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-saml-logout) and passing the access token and refresh token as parameters. For example:

    ```console
    POST /_security/saml/logout
    {
      "token" : "46ToAxZVaXVVZTVKOVF5YU04ZFJVUDVSZlV3",
      "refresh_token": "mJdXLtmvTUSpoLwMvdBt_w"
    }
    ```

    If the SAML realm is configured accordingly and the IdP supports it (see [SAML logout](/deploy-manage/users-roles/cluster-or-deployment-auth/saml.md#saml-logout)), this request will trigger a SAML SP-initiated Single Logout. In this case, the response will include a `redirect` parameter indicating where the user needs to be redirected at the IdP in order to complete the logout.

2. Alternatively, the IdP might initiate the Single Logout flow at some point. In order to handle this, the Logout URL (`sp.logout`) needs to be handled by the custom web app. The query part of the URL that the user will be redirected to will contain a SAML Logout request and this query part needs to be relayed to {{es}} using the [SAML invalidate API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-saml-invalidate)

    ```console
    POST /_security/saml/invalidate
    {
      "query" : "SAMLRequest=nZFda4MwFIb%2FiuS%2BmviRpqFaClKQdbvo2g12M2KMraCJ9cRR9utnW4Wyi13sMie873MeznJ1aWrnS3VQGR0j4mLkKC1NUeljjA77zYyhVbIE0dR%2By7fmaHq7U%2BdegXWGpAZ%2B%2F4pR32luBFTAtWgUcCv56%2Fp5y30X87Yz1khTIycdgpUW9kY7WdsC9zxoXTvMvWuVV98YyMnSGH2SYE5pwALBIr9QKiwDGpW0oGVUznGeMyJZKFkQ4jBf5HnhUymjIhzCAL3KNFihbYx8TBYzzGaY7EnIyZwHzCWMfiDnbRIftkSjJr%2BFu0e9v%2B0EgOquRiiZjKpiVFp6j50T4WXoyNJ%2FEWC9fdqc1t%2F1%2B2F3aUpjzhPiXpqMz1%2FHSn4A&SigAlg=http%3A%2F%2Fwww.w3.org%2F2001%2F04%2Fxmldsig-more%23rsa-sha256&Signature=MsAYz2NFdovMG2mXf6TSpu5vlQQyEJAg%2B4KCwBqJTmrb3yGXKUtIgvjqf88eCAK32v3eN8vupjPC8LglYmke1ZnjK0%2FKxzkvSjTVA7mMQe2AQdKbkyC038zzRq%2FYHcjFDE%2Bz0qISwSHZY2NyLePmwU7SexEXnIz37jKC6NMEhus%3D",
      "realm" : "saml1"
    }
    ```

    The custom web application will then need to also handle the response, which will include a `redirect` parameter with a URL in the IdP that contains the SAML Logout response. The application should redirect the user there to complete the logout.


For SP-initiated Single Logout, the IdP may send back a logout response which can be verified by {{es}} using the [SAML complete logout API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-saml-complete-logout).




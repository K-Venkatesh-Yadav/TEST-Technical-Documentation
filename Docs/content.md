# **1. Foundation Base Introduction**

*1.1. About This Document*

This document describes how to use GE’s common technology stack, called Foundation, to build
containerized applications on Kubernetes.

*1.2. Purpose of This Document*

This document contains a guide into all functional areas of Foundation.

*1.3. Who Should Use This Document*

This document is intended for individuals who are building applications on top of the Foundation
layers.

*1.4. Related Resources*

For more information about Foundation, refer to the following:

    Foundation Base Release Notes: Provides release-specific information, including hardware and software prerequisites.

    Foundation Base Installation Guide: Describes how to install the Foundation Base system.

*1.5. System Overview*

Foundation is a set of Kubernetes-based solution components that support security and monitoring features for specific GE products built on top of it (for example, Outage Assist and Internal Notifications). Foundation deployment, scaling, and management are implemented using Kubernetes.

Foundation is comprised of a number of different layers:

**Kubernetes**: RKE2-based deployment of Kubernetes.

**Base**: Core capabilities such as security, logging, monitoring, and data-stores.

**OSB**: Integration layer, including messaging and integration patterns.

**UI Server**: Server-side components to aid with UI-based applications.

Foundation Base Introduction                             Proprietary - See Copyright Page
Foundation 25r04 Base User Guide


# **2. Design Principles**

*2.1. API Design*

As more microservices are exposing their APIs to the web browser, developers should follow a
model that can be expanded with new APIs and new versions, while at the same time creating new
APIs without conflicting with other developments.

The following model is strongly recommended for the paths of the services exposed outside the
Kubernetes cluster: [/component/api/v<version number>/endpoint](https://)

For example: [/aorManager/api/v2/aors/idp/api/v1/login](https://)

To ensure consistency, permissions associated with those endpoints should take the application or component name as the first part of the permission name. For example, to call the endpoint [/aorManager/api/v2/aors](https://) we should have a permission named after the application or component name; in our case, it would be aorManager.aors.read.readAll.

The microservice implementing the API should not be aware of the first part of the path (for example, aorman, idp, etc.). For the microservice, the endpoints are /api/v…. That means the API Gateway will have to rewrite the path to remove the first part. That also means you can port the service from one path to another (at the API Gateway level) without having to recompile the microservice.

*2.2. Security Components and Naming Conventions*

Component Description aorManager API for the AOR Manager sessionManager API for the API Gateway Session Manager (deals with existing sessions and termination of sessions) roleManager API for the Role Manager idp API for the Identity Provider, including SAML2 endpoints, login and domains.

*2.3. Approach to Secure UI Applications*

This section describes the recommended approach for authentication/authorization when there are
several Foundation Base applications that you want to combine and deploy on a single Kubernetes

Some of the applications may be available to some users, but not others.

A common architecture of a web application that consists of a UI (Web Server) and an API (API
Service) is deployed as follows:

The UI is implemented by a web server (such as NGINX or Apache) serving static files. The API is implemented by one or multiple microservices, written in Java, for example.

The proposed approach consists of adding the Istio sidecar proxy in front of both the web server and the microservice(s). The API Gateway will continue to enforce authentication; the UI cannot be accessed unless the user has been authenticated. The authorization aspect is enforced by the sidecar proxy; only users with certain permissions in their roles will be able to receive resources (for example, JavaScript, HTML, and images). If not authorized, the sidecar proxy will return a 403 HTTP
code, which is intercepted by the API Gateway to display an error page that says "You are not authorized to access this page".

*2.4. Secret Management*

Refer to Secret Management for more information about how to handle secrets in Foundation Base and applications based on it.

*2.5. Certificate Management*

Refer to Certificate Management for more information about how to handle certificates in Foundation Base and applications based on it.

# **3. Login Process**

This chapter explains the login process when a user accesses a service from the web browser. As an example, the Admin Console service is used, which is accessed from the /secadmin/ path.

Figure 2. <u>Login Process Stage 1</u>

The process starts with a user who wants to access the Admin Console service by entering the URL ([https://<cluster-name>/secadmin/](https://)) in a web browser (User Agent).

The request arrives at the API Gateway. The API Gateway then checks to see if there is already a ookie associated with the request (there is none). The API Gateway replies with a 302 /login to redirect the User Agent.

The User Agent requests /login at the API Gateway. The API Gateway creates a session associated with this User Agent and replies with 302 /uaa/oauth/authorize to redirect the User Agent.

The User Agent requests /uaa/oauth/authorize, which the API Gateway forwards to UAA as /oauth/authorize to begin the OAuth2 Authorization Code grant flow. The UAA replies with 302 /uaa/login to redirect the User Agent.

The User Agent requests /uaa/login, which the API Gateway forwards to UAA as /login. The UAA is configured to use an external SAML Identity Provider (IdP), which it replies with 302 /uaa/saml/discovery to redirect the User Agent to acquire the SAML IdP information.

The User Agent requests /uaa/saml/discovery, which the API Gateway forwards to UAA a /saml/discovery. The UAA replies with 302 /uaa/saml/login/alias/cloudfoundry-saml-login to redirect the User Agent.

The User Agent requests /uaa/saml/login/alias/cloudfoundry-saml-login, which the API Gateway forwards to the UAA as /saml/login/alias/cloudfoundry-saml-login. The UAA replies with 302 /idp/saml2/SingleSignOnService to redirect the User Agent to the URL for the SAML IdP.

Figure 3. <u>Login Process Stage 2</u>

The User Agent requests /idp/saml2/SingleSignOnService, which the API Gateway forwards to the IdP as /saml2/SingleSignOnService. The IdP processes the SAML request and creates a cookie to identify this login request. The IdP replies with 302 /idp/ to redirect the User Agent.

The User Agent requests /idp/, which the API Gateway forwards to the IdP. The IdP responds with the login page for the user to enter credentials.

User credentials are entered and submitted to the IdP. The IdP validates the user credentials with an LDAP server (not shown); if successful, it creates a SAML response that it embeds into an autopost page. The IdP replies with 302 /idp/saml2/autopost to redirect the User Agent.

Figure 4. <u>Login Process Stage 3</u>

The User Agent requests the /idp/saml2/autopost endpoint, which the API Gateway forwards to the IdP. The IdP generates the SAML Response for UAA and replies with 302 /uaa/saml/SSO/alias/cloudfoundry-saml-login to redirect the User Agent.

The User Agent requests /uaa/saml/SSO/alias/cloudfoundry-saml-login, which the API Gateway forwards to UAA. The UAA processes the SAML response to accept the authenticated user and replies with 302 /uaa/oauth/authroize to redirect the User Agent.

The User Agent requests /uaa/oauth/authorize, which the API Gateway forwards to UAA. The UAA generates the authorization code and encodes it with the response, and then replies with 302 /callback.

The User Agent requests /callback with the authorization code. The API Gateway extracts the authorization code and sends a /oauth/token request to UAA to get the access token. The UAA verifies the authorization code and issues the Bearer Token in response to the API Gateway. The API Gateway stores the Bearer Token into a distributed cache. Then the API Gateway replies with 302/secadmin/ to redirect the User Agent to the originally requested URL.

The User Agent requests /secadamin/. The API Gateway use the cookie from the User Agent to locate the Bearer Token from the distributed cache and validates whether the token has not expired. If valid, the API Gateway forwards the request to the Sec Admin UI, which is the service for /secadmin/.

# **4. The Identity Provider**

Customers may use the SAML Identity Provider (IdP) implementation that comes with Foundation Base, or they may connect to their own Identity Provider (such as SimpleSAMLphp, Okta, Azure AD,and so on). This section explains both how to configure Foundation Base’s IdP and how to configure an external one.

*4.1. Architecture*

The following diagram represents a high level view of the IdP architecture:
As shown in the diagram, the IdP component interacts with AD/LDAP (Active Directory/LDAP) or the
RSA AM (RSA Authentication Manager).

*4.2. Configuration*

The following steps describe how to initially set up the IdP using the Data Loader when deploying
the Foundation Base Umbrella chart:
IDP authenticators secret should be created beforehand, below shows an example

apiVersion: v1

kind: Secret

metadata: XXXXXXX

name: sws-login-app-server-authenticators-secret

namespace: foundation-cluster-zerotrust

type: Opaque
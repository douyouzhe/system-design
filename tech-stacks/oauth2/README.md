# OAuth 2.0

Before OAuth 2.0, third-party applications often required users to share their login credentials (username and password) with the application to access their resources on a service. This posed significant security risks, as the application could misuse or mishandle the user's credentials. OAuth 2.0 eliminates the need for sharing credentials by providing a secure mechanism for delegated access.

OAuth 2.0 (Open Authorization 2.0) is an authorization framework that allows third-party applications to obtain limited access to a user's resources on a web service, without needing to know the user's login credentials or gain full access to their account. It is commonly used in the context of granting access to APIs (Application Programming Interfaces) and enabling secure interactions between different software systems.

## Use cases
<p align="center">
<img src="/tech-stacks/oauth2/screenshot_ThirdPartyLogin.png" width="800">
</p>

* **Third-Party Application Integration:** OAuth 2.0 enables third-party applications to integrate with platforms or services, such as social media sites, without requiring users to share their login credentials. For example, when you use a mobile app to sign in to a service using your Google or Facebook account, OAuth 2.0 is likely being used behind the scenes to grant the app access to your account data.

* **Single Sign-On (SSO):** OAuth 2.0 can be used to implement Single Sign-On solutions, where users authenticate once and can then access multiple applications or services without the need to provide their credentials again. OAuth 2.0 acts as the authentication and authorization mechanism, allowing users to log in to one application and have their identity propagated to other authorized applications.

* **API Authorization:** OAuth 2.0 is commonly used to secure APIs. It allows developers to define scopes and permissions associated with their APIs, and clients can request access tokens with specific scopes to access protected resources. This enables fine-grained control over what resources and actions are accessible to the client.

* **Federated Identity:** OAuth 2.0 can be used in federated identity scenarios where a user's identity and access rights are managed by an identity provider (IdP), and other applications or services rely on the IdP for authentication and authorization. OAuth 2.0 allows applications to obtain access tokens from the IdP to access user resources.

## Key Roles

* **Resource Owner:** This refers to the end-user who owns the resources (such as data or services) on a web service and wants to grant access to those resources to a third-party application.

* **Client:** The client is the application or service that requests access to the user's resources. It can be a web application, mobile app, or even a server-side process.

* **Authorization Server:** The authorization server is responsible for authenticating the resource owner and issuing access tokens to the client after obtaining the owner's consent. It verifies the identity of the client and determines the level of access the client should have.

* **Resource Server:** The resource server hosts the protected resources, which the client wants to access. It validates the access token presented by the client and grants or denies access to the requested resources.

* **Access Token:** An access token is a credential that represents the authorization granted to the client. It is issued by the authorization server and used by the client to access protected resources on the resource server.

* **Grant Types:** OAuth 2.0 defines several grant types that describe different ways to obtain access tokens. These include the authorization code grant, implicit grant, client credentials grant, and others. Each grant type is suitable for different scenarios and security requirements.

## Protocol Flow
Oauth provides an "authorization layer" between the Client and the Resource Server so that it is not possible and will not be necessary for the Client to access the Resource server directly using the username and password, instead, the client will be using a token which has scope and expiration information.

<p align="center">
<img src="/tech-stacks/oauth2/flow.png" width="600">
</p>


* **User Authorization:**
The client redirects the user to the authorization server's authorization endpoint, along with the requested scope of access.
The user is presented with a login screen or other authentication mechanism provided by the authorization server.
The user authenticates with the authorization server, proving their identity.
The authorization server presents the user with a consent screen, requesting permission to grant access to the client.
The user reviews the requested permissions and grants or denies access to the client.

* **Authorization Code Exchange (Authorization Code Flow):**
If the user grants access, the authorization server redirects the user back to the client's redirect URI, with an authorization code included in the query parameters. The client requests the authorization server's token endpoint, providing the authorization code, and client credentials (client ID and client secret), and redirects URI.
The authorization server verifies the authorization code, and client credentials, and redirects URI.
If the verification is successful, the authorization server issues an access token and, optionally, a refresh token to the client.

* **Accessing Protected Resources:**
The client includes the access token in subsequent requests to the resource server's API by including it in the request headers or as a parameter.
The resource server receives the request and verifies the access token.
If the access token is valid and has the necessary scopes, the resource server allows the client to access the requested resources.
The client can perform actions on behalf of the user and retrieve the requested data.


## Authorization Grant
OAuth 2.0 defines several authorization grant types, also known as flows or ways, through which a client (third-party application) can obtain an access token from the authorization server. The choice of grant type depends on the specific requirements and characteristics of the client and the desired level of security. 

* **Authorization Code Grant:** This grant type is suitable for server-side applications or applications that can securely maintain a client secret.
The client redirects the user to the authorization server's authorization endpoint to request authorization.
The user authenticates and grants permission to the client.
The authorization server redirects the user back to the client with an authorization code.
The client exchanges the authorization code for an access token by sending a request to the authorization server's token endpoint.
The authorization server verifies the code and issues an access token to the client.
<p align="center">
<img src="/tech-stacks/oauth2/images-oauth_auth_code.png" width="600">
</p>

* **Implicit Grant:** This grant type is suitable for client-side applications, such as JavaScript applications running in a web browser or mobile apps.
The client redirects the user to the authorization server's authorization endpoint to request authorization.
The user authenticates and grants permission to the client.
The authorization server redirects the user back to the client with an access token embedded in the URL fragment.
The client extracts the access token from the URL fragment and can use it to access protected resources.
Note: Implicit grant does not involve the exchange of an authorization code, making it less secure compared to other grant types.
<p align="center">
<img src="/tech-stacks/oauth2/images-oauth_implicit.png" width="600">
</p>

* **Resource Owner Password Credentials Grant:** This grant type is suitable when the client is highly trusted and owned by the same entity as the authorization server.
The client directly collects the user's credentials (username and password).
The client sends the user's credentials along with its client credentials to the authorization server's token endpoint.
The authorization server verifies the credentials and issues an access token directly to the client.
<p align="center">
<img src="/tech-stacks/oauth2/images-oauth_password.png" width="600">
</p>


* **Client Credentials Grant:** This grant type is suitable for server-to-server communication or cases where the client is acting on its behalf rather than on behalf of a user.
The client sends its client credentials (client ID and client secret) to the authorization server's token endpoint.
The authorization server verifies the client credentials and issues an access token directly to the client.
<p align="center">
<img src="/tech-stacks/oauth2/images-oauth_client_credentials.png" width="600">
</p>

* **Refresh Token Grant:** This grant type allows the client to obtain a new access token without user involvement when the current access token expires or becomes invalid.
The client sends a request to the authorization server's token endpoint, including a refresh token obtained previously.
The authorization server verifies the refresh token and issues a new access token to the client.
These grant types provide different levels of security and are designed to cater to various client types and scenarios. It's important to choose the appropriate grant type based on the specific requirements and security considerations of your application.
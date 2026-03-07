# Understanding SAML 2.0 Authentication

##### Reference: https://web.archive.org/web/20240421215604/https://goteleport.com/blog/how-saml-authentication-works/ 
> **By Russell Jones**
> *This article was originally published on July 29th, 2019.*

## Table of Contents

1. [What is SAML 2.0?](https://www.google.com/search?q=%23what-is-saml-20)
2. [SAML Terminology](https://www.google.com/search?q=%23saml-terminology)
3. [The SAML Login Flow (Diagram)](https://www.google.com/search?q=%23the-saml-login-flow)
4. [Configuration](https://www.google.com/search?q=%23configuration)
5. [Authentication Request](https://www.google.com/search?q=%23authentication-request)
6. [Authentication Response](https://www.google.com/search?q=%23authentication-response)
7. [Adding SAML based SSO to your app](https://www.google.com/search?q=%23adding-saml-based-sso-to-your-app)
8. [Conclusion](https://www.google.com/search?q=%23conclusion)

---

## What is SAML 2.0?

Security Assertion Markup Language (SAML) 2.0 is one of the most widely used open standard for authentication and authorizing between multiple parties. SAML is an open protocol that give users the single sign-on (SSO) experience for applications. The other adopted open standard is OAuth and OpenID. Of the two, SAML 2.0, released in 2005, remains the 800 pound gorilla in Enterprise SSO space. This post provides a detailed introduction on how SAML works.

At its core, Security Assertion Markup Language (SAML) 2.0 is a means to exchange authorization and authentication information between services. SAML is frequently used to implement internal corporate single sign-on (SSO) solutions where the user logs into a service that acts as the single source of identity which then grants access to a subset of other internal services.

**The advantage of adopting SAML/SSO from a security perspective are clear:**

* **Single source of identity:** When an employee joins or leaves a company, you don’t have to worry about the myriad of internal services that now have to be updated, and the ones that will inevitably be missed.
* **Enforce consistent authentication:** SAML/SSO can be used to enforce consistent method of authentication across all internal corporate services, like multifactor authentication and session duration.

This particular post will be focused on providing an overview of the how and why of SSO and SAML.

---

## SAML Terminology

Unfortunately before going any further we have to define some SAML-specific terminology, of which a fair amount exists.

### Principal

The **principal** is the user trying to authenticate. You can think of this as the actual human behind the screen, for the remainder of this post, we'll assume it's John Smith.

It is helpful to think about principal having metadata attached to it. First name, last name, email address, etc. This metadata is also called *identity information* and its importance will be explained below.

### Identity Provider

An **Identity Provider**, frequently abbreviated as **IdP**, is the service that serves as the source of identity information and authentication decision. Think of identity providers as databases for identity information.

Identity providers authenticate principals and return identity information to service providers (see below). A few examples of common identity providers: Auth0, Active Directory Federation Services (ADFS), and Okta.

> **Note:** The industry best practice is to consolidate all user identities of an organization in a single identity provider.

### Service Providers

**Service providers** frequently abbreviated as **SP**, are the services that are requesting authentication and identity information about the principal. Service providers take authentication responses received from identity providers and use that information to create and configure sessions.

In other words, a service provider is an application that offers a single sign on (SSO) mechanism for its users to login and access its resources. Such applications, in addition to knowing a principal name or email address may need to request additional identity information about a principal, usually to implement role-based access control (RBAC).

Examples of such applications include Github, Google Apps. Teleport, our access platform solution, is also a service provider.

### Flows

SAML supports two different types of flows: those initiated by the service provider and those initiated by the identity provider. In this post, we’ll cover the common **SP-initiated** flow. In SP-initiated flows, you start out at the service provider, are redirected to the identity provider to authenticate, and are then redirected back to the service provider.

This flow is usually initiated when a user clicks on "Login with SSO" button or something similar.

### Bindings

**Bindings** are the format in which data is transferred between service providers and identity providers. The two most popular are **HTTP Redirect Binding** and **HTTP POST Binding**.

* **HTTP Redirect Bindings** transfer data using HTTP redirects and query parameters; this type of binding is typically used in authentication requests.
* **HTTP POST Binding** transfer data using HTTP POST forms, this type of binding is typically used in authentication responses.

In other words, bindings define *how* the identity information about the principal is communicated from an identity provider to a service provider.

### Assertions

**Assertions** are statements made by the identity provider about the principal. For example, the principal's email address and/or groups/roles the principal may be associated with. Assertions are used by the service provider to create and configure sessions for a principal.

In other words, assertions define *what* identity information about the principal is communicated from an identity provider to a service provider.

---

## The SAML Login Flow

To illustrate how SAML Login works, we are going to use Teleport as an example of a service provider and Auth0 for an identity provider.

```mermaid
sequenceDiagram
    autonumber
    actor Principal as User (John Smith)
    participant SP as Service Provider<br/>(Teleport)
    participant IdP as Identity Provider<br/>(Auth0)

    Principal->>SP: Clicks "Login via Auth0" (SSO)
    SP->>SP: Generates secure AuthnRequest
    SP->>Principal: HTTP 302 Redirect to IdP URL<br/>(AuthnRequest in query param)
    Principal->>IdP: Follows redirect to begin SAML authentication
    IdP->>Principal: Prompts for username/email, password, & 2FA
    Principal->>IdP: Submits credentials
    IdP->>IdP: Authenticates user, packages identity into Assertions
    IdP->>Principal: Returns HTML form (HTTP POST Binding)
    Principal->>SP: Browser auto-posts SAMLResponse to Teleport ACS URL
    SP->>SP: Validates signature and creates user session
    SP->>Principal: Grants access to application

```

**The SAML Login flow step-by-step:**

1. A user clicks on "Login via Auth0" button, choosing to login via Auth0 using SAML, as opposed to using Teleport's built-in user database. Teleport redirects the user to Auth0. SAML authentication begins. We'll refer to the user as principal from now on.
2. Auth0 asks the user for their username or email, their password and the second factor (2FA) authentication token.
3. If the supplied information is correct, Auth0 will authenticate the user and take the principal's identity and returns it back to Teleport as *assertions*.
4. Teleport receives the identity information from Auth0 and this allows it to create a user session.

---

## Configuration

Identity providers all have their own unique methods of configuration. However, the following minimal set of configuration is needed for the identity provider to work with a service provider.

* An **Assertion Consumer Service (ACS) URL** has to be configured. The ACS URL is an endpoint on the service provider where the identity provider will redirect to with its authentication response. This endpoint should be an HTTPS endpoint because it will be used to transfer Personally Identifiable Information (PII).
* Generate and upload a **signing key** to be used to sign authentication requests. More on this later.
* Configuration of **assertions**, which includes information about the principal. The source and format of this information will need to be configured. At the minimum, the identity provider will send a NameID and some assertions about the principal (like group membership).

Service provider configuration is typically simpler, and can often by automatically configured by parsing metadata provided by the identity provider. Simplified identity provider metadata XML is shown in Figure (2) below.

The most important tags are `SingleSignOnService` and `KeyDescriptor`. The `SingleSignOnService` tags define the binding and endpoints to send authentication requests to, and the `KeyDescriptor` tag contains the *public key* of the identity provider which will be used to validate the authentication response.

*Figure 2: Simplified metadata from an identity provider.*

```xml
<md:EntityDescriptor>
  <md:IDPSSODescriptor>
    <md:KeyDescriptor use="signing">
      <ds:KeyInfo>
        <ds:X509Data>
          <ds:X509Certificate>
            MIICMjCCAZugAwIBAgIBADANBgkqhkiG9w0BAQ0FADA2MQswCQYDVQQGEwJ1czEL
            MAkGA1UECAwCQ0ExDDAKBgNVBAoMA2lkcDEMMAoGA1UEAwwDaWRwMB4XDTE5MDQy
            NjE4NTIxOFoXDTIwMDQyNTE4NTIxOFowNjELMAkGA1UEBhMCdXMxCzAJBgNVBAgM
            AkNBMQwwCgYDVQQKDANpZHAxDDAKBgNVBAMMA2lkcDCBnzANBgkqhkiG9w0BAQEF
            AAOBjQAwgYkCgYEA1mKmlbr/SiHOhgdROpYeze96mw0WbO+BdJYDceeuNkaw0zOU
            CKZI6TNgrNsqEnLOyWYy5ywA9XA6Ni2qQTuKqapsMT3I1s9DMUg2ln7tTzNdhE02
            fY4GVjiCw7i9YJ+cgcMZh8qL0yoilrLpRLzLrRC6rApqYfEwn+5FPKtTt7cCAwEA
            AaNQME4wHQYDVR0OBBYEFNvFMRtHJ4D327dbRbxhWceXnwd0MB8GA1UdIwQYMBaA
            FNvFMRtHJ4D327dbRbxhWceXnwd0MAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQEN
            BQADgYEAX0I5zpGqI7vzzs8CDyokux1JZzfu+O3P5GfOwUaIG9y01FzxgbL2MRKQ
            oTXMAed97Q6vHA5cffvteu/rPcerpGmFj5h3wv5u+D0ch5s/Mk/Ug6S+x6k3CC+P
            kHimi6OEslFecDMhghUtPJAmhOGnTRwLr7hVeJXBHXWCTXA7aGE=
          </ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </md:KeyDescriptor>
    <md:NameIDFormat>
       urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
    </md:NameIDFormat>
    <md:SingleSignOnService
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
      Location="https://idp.example.com/saml"/>
    <md:SingleSignOnService
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
      Location="https://idp.example.com/saml"/>
  </md:IDPSSODescriptor>
</md:EntityDescriptor>

```

---

## Authentication Request

To begin the login process, the principal typically will navigate to the service providers login page, and then click on a button that reads something like, “Login via SSO.”

At this point the service provider will create an `AuthnRequest` XML document, serialize it (base64, compress, and URL encode), add it as a query parameter to the URL, and redirect the principal’s browser to the identity provider’s login page. As the name of the request implies, the service provider is requesting the identity provider to perform authentication on its behalf. It should go without saying that this (and all other requests and responses) should occur over HTTPS.

The `AuthnRequest` is often sent using HTTP Redirect Binding, and ends up looking something like the following:

```
https://idp.example.com/saml?SAMLRequest=nFdpk6JK0%2F0rHc7...

```

Where the SAMLRequest parameter is the encoded `AuthnRequest`, a simplified `AuthnRequest` follows.

*Figure 3: Simplified authentication request sent by the service provider to the identity provider.*

```xml
<samlp:AuthnRequest 
  AssertionConsumerServiceURL="https://sp.example.com/saml/acs" 
  Destination="https://idp.example.com/saml" 
  ID="bcf0b634-67b4-4dc9-a436-4e5cfcfb80e2" 
  IssueInstant="2019-04-17T18:15:16Z" 
  ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" 
  Version="2.0">
  <saml:Issuer>
  https://sp.example.com/saml/acs
  </saml:Issuer>
  <ds:Signature>
    <ds:SignedInfo>
      <ds:CanonicalizationMethod
      Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
      <ds:SignatureMethod
      Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" />
      <ds:Reference URI="#bcf0b634-67b4-4dc9-a436-4e5cfcfb80e2">
        <ds:Transforms>
          <ds:Transform
          Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature" />
          <ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
        </ds:Transforms>
        <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256" />
        <ds:DigestValue>
        XBCrZlFG5PxsHkeo5zNeRtDnB9rZEbo4R0HwFbl6O3c=
        </ds:DigestValue>
      </ds:Reference>
    </ds:SignedInfo>
    <ds:SignatureValue>
    PLslJMUXoW99FDpFdlUdKojYJbJWhWaJ8URQOOGAqXQ/QAF2mLsdJOOu834H9T7USVkeWn7e1NuK
    9pV6C0CEo/INaT/2knWH/OlRl2jH0e+NR9bLcGY49MBAqaqCd/FbHm/bvPEerM2SrLkYA9dmN5vE
    6NVm5cvdMue9KZRnRZ4j5BAstC03Fd9/5OGetgcDiIlNMTDjhEuSgijVR0+jkSgPu/mZLfQ3IT+9
    tcHvC9VUuigVRXGEqOHsd2yX4Rhl9f6u1Y0BAg82RerHg6VEzxM9ZDvIyKIq/Ne8Hz0COlVWrJ/a
    qoi9RzROfPFOsEW8ohLUeZ74QS93jYztfY4BzA==
    </ds:SignatureValue>
    <ds:KeyInfo>
      <ds:X509Data>
        <ds:X509Certificate>
        MIIDKjCCAhKgAwIBAgIQUBVD9B5t9U5IvGFV2F/igDANBgkqhkiG9w0BAQsFADBAMRUwEwYDVQQK
        EwxUZWxlcG9ydCBPU1MxJzAlBgNVBAMTHnRlbGVwb3J0LmxvY2FsaG9zdC5sb2NhbGRvbWFpbjAe
        Fw0xNzA4MTYxOTU0NTFaFw0yNzA4MTQxOTU0NTFaMEAxFTATBgNVBAoTDFRlbGVwb3J0IE9TUzEn
        MCUGA1UEAxMedGVsZXBvcnQubG9jYWxob3N0LmxvY2FsZG9tYWluMIIBIjANBgkqhkiG9w0BAQEF
        AAOCAQ8AMIIBCgKCAQEAopoQnx7RELetBxUlkRZnSHLRYakETx6OiwCqlArC89tXsTmbv71AsQJJ
        j0Xys7py5gfcKF2onjSj3FMX/ApxjdQIiq7+60g3WIRsaTSIGaHU/Gq/GQXPIfGxvDEp+mnytD7Q
        C39VyeUHIDicXv1IluGvolxtxjU8Fi/tGpCEuhREhfcOiXOaRMA8ABPHj6qLZOaq80yIxuUqv2jP
        aRw2jvAcCLUtOF4W5gs+3ms9d/qZRjIEPkrzPiEHRAVvcUb4Lutwx1YJcPUPviIVdcH7JiWj2eEU
        9fObjrxvLJ2ZMET4bz/thH3okOvkfvE0Xt0L6G9Gv08wqsmwWP0BNJYTwQIDAQABoyAwHjAOBgNV
        HQ8BAf8EBAMCB4AwDAYDVR0TAQH/BAIwADANBgkqhkiG9w0BAQsFAAOCAQEASguYy7pt3DoJ24rs
        MDTBa9+ZsVfgHHmCbwhXgyt1siXdA8USluYwOGnECo+yuOdfFK2545UPpNp5G5pGGxSf0LP1Usud
        ANQ97R6vF7XIYJAmVgHITMpjuVgGlJi1+Ro2NzoX1EF8X2GowJr6hehERCSL16PVXAo/b41ypMad
        3yTMKW6YNWEoBftZfhSjb9ZVpbGomgE20UMtp0d601MZy20M8zJBEH4yXhxzJ1/3nJpfW4c3VOK6
        QDDJpFFPl4GZwF31qcpIPSpHuokbskkhnSgiPjdc8k73q3bIyg7RonTvf8kfmD5TU+K9pOIKvF48
        JVyH0UBaXnojNYyltHgXnA==
        </ds:X509Certificate>
      </ds:X509Data>
    </ds:KeyInfo>
  </ds:Signature>
</samlp:AuthnRequest>

```

**A few things to note about the `AuthnRequest`:**

* The service provider generates a large secure random number and inserts it into the `ID` field in the `AuthnRequest` tag. This value is also stored locally (typically in a database) and is used to pair requests with responses from an identity provider and to prevent a malicious third party from sending an unsolicited response to a request (it would not know the `ID`).
* To prevent the re-use of expired `AuthnRequests`, the identity provider needs to store and track which `ID` values have been used thus far. Without some kind of time bound, this would lead to the identity provider needing an ever increasing amount of storage. The `IssueInstant` is used to generate that validity window for the request.
* The service provider should sign the `AuthnRequest`. The SAML signing scheme includes the signature, key used to sign the request, and information on how to calculate the signature all under the `Signature` tag. The identity provider should not only validate the included key was used to sign the request, but also that the key is the same as the one uploaded during configuration (see previous section). If the key does not match, is missing, or if the signature values do not match, the identity provider knows the request is not legitimate and to reject it. XML digital signatures, which is what SAML uses to sign requests, are a large and complex topic in itself, *Signing an XML document using XMLDSIG* is a good place to start to learn more.

---

## Authentication Response

Returning back to the authentication flow, at this point the principal is sitting at the login page of the identity provider. If the principal enters their correct login credentials, the identity provider will perform a 302 Redirect to the ACS URL of the service provider with body containing the authentication response. In the example we have been tracking, the ACS URL would look like the following:

```
https://sp.example.com/saml/acs

```

*Figure 4: Simplified authentication response sent by the identity provider to the service provider.*

```xml
<saml2p:Response 
  Destination="https://sp.example.com/saml/acs" 
  ID="id35287812421219341967493380" 
  InResponseTo="bcf0b634-67b4-4dc9-a436-4e5cfcfb80e2" 
  IssueInstant="2019-04-18T18:51:46.729Z">
  <ds:Signature>
    <ds:SignedInfo>
      <ds:CanonicalizationMethod
      Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
      <ds:SignatureMethod
      Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" />
      <ds:Reference URI="#id35287812421219341967493380">
        <ds:Transforms>
          <ds:Transform
          Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature" />
          <ds:Transform
          Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
        </ds:Transforms>
        <ds:DigestMethod
        Algorithm="http://www.w3.org/2001/04/xmlenc#sha256" />
        <ds:DigestValue>
        tyLUm4r2isgN+L6sRcqDSEa1Zb7WQbQJG6PpLcf3Mrc=
        </ds:DigestValue>
      </ds:Reference>
    </ds:SignedInfo>
    <ds:SignatureValue>
    XjqbZty/QkqTnMV8YsS2XJ3qgVLPGNC67o/WmzkzoAyl3SBOCGllV4UdijkTjhgykQP7MVXyCql0
    eRtIMJ++rbi3OxCSc0LN67znuTS7cAfcOQzYBtYX2R9w3GlEAO0kZusWYlP3cu/ObmQZUQ7CSgr4
    DRXsVWRhSmmpxHl6klC6c10eWiIlK7Ccpvvvb2hlwl8anyuO/CcKH0n/Rb9vHWtsAlqKXZ8G4X6M
    77AfRFC7yDWk+8B784109phQxcxoDYjuQNO5IkiRE6J2LnkmuaPoKVyTtpP2JYLiYMSBu8laDsnZ
    I/ewOtBwr16j9oOJpgHPQufQJfvcg+rPEwkptg==
    </ds:SignatureValue>
    <ds:KeyInfo>
      <ds:X509Data>
        <ds:X509Certificate>
        MIICMjCCAZugAwIBAgIBADANBgkqhkiG9w0BAQ0FADA2MQswCQYDVQQGEwJ1czEL
        MAkGA1UECAwCQ0ExDDAKBgNVBAoMA2lkcDEMMAoGA1UEAwwDaWRwMB4XDTE5MDQy
        NjE4NTIxOFoXDTIwMDQyNTE4NTIxOFowNjELMAkGA1UEBhMCdXMxCzAJBgNVBAgM
        AkNBMQwwCgYDVQQKDANpZHAxDDAKBgNVBAMMA2lkcDCBnzANBgkqhkiG9w0BAQEF
        AAOBjQAwgYkCgYEA1mKmlbr/SiHOhgdROpYeze96mw0WbO+BdJYDceeuNkaw0zOU
        CKZI6TNgrNsqEnLOyWYy5ywA9XA6Ni2qQTuKqapsMT3I1s9DMUg2ln7tTzNdhE02
        fY4GVjiCw7i9YJ+cgcMZh8qL0yoilrLpRLzLrRC6rApqYfEwn+5FPKtTt7cCAwEA
        AaNQME4wHQYDVR0OBBYEFNvFMRtHJ4D327dbRbxhWceXnwd0MB8GA1UdIwQYMBaA
        FNvFMRtHJ4D327dbRbxhWceXnwd0MAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQEN
        BQADgYEAX0I5zpGqI7vzzs8CDyokux1JZzfu+O3P5GfOwUaIG9y01FzxgbL2MRKQ
        oTXMAed97Q6vHA5cffvteu/rPcerpGmFj5h3wv5u+D0ch5s/Mk/Ug6S+x6k3CC+P
        kHimi6OEslFecDMhghUtPJAmhOGnTRwLr7hVeJXBHXWCTXA7aGE=
        </ds:X509Certificate>
      </ds:X509Data>
    </ds:KeyInfo>
  </ds:Signature>
  <saml2p:Status>
    <saml2p:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success" />
  </saml2p:Status>
  <saml2:Assertion
  ID="id35287812421980111258419174"
  IssueInstant="2019-04-18T18:51:46.729Z">
    <ds:Signature>
      <ds:SignedInfo>
        <ds:CanonicalizationMethod
        Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
        <ds:SignatureMethod
        Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" />
        <ds:Reference URI="#id35287812421980111258419174">
          <ds:Transforms>
            <ds:Transform
            Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature" />
            <ds:Transform
            Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
          </ds:Transforms>
          <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256" />
          <ds:DigestValue>
          VKPsgTPABNq1SvInCMXd04LZCvRYMnJzEeT5oIs70hw=
          </ds:DigestValue>
        </ds:Reference>
      </ds:SignedInfo>
      <ds:SignatureValue>
      gsUzQuivXX378HkYNI+plBkp1BvPUNmJD+kh825nHwIBNd019IxffVmOfRAQAkZhT6rqxWhO5/Yc
      JGR5J0qjJVmrRrJ/ipT4VfuJsbn346nEFSMU15D0h3UHrvl651C+NStyXsi8Q8502Qe0ChHOtEXM
      rw9HWPwYtJX0rlpNEzLUnEQPvJ4pd3bz9SIl/YXMNTxE7NCDOxPXKtA4namkkweilxTCynM6A1kn
      6gEWaXhLMwLLAV6kOtivdVksBPzR9BeZ7RPpXeqt0qN62L4NaHq3OsdjgtQr9sllssD1fEek1eU4
      giCzPgb1+LjvD9dpFH5pcLt9YlwHyYgEBBLOQg==
      </ds:SignatureValue>
      <ds:KeyInfo>
        <ds:X509Data>
          <ds:X509Certificate>
          MIICMjCCAZugAwIBAgIBADANBgkqhkiG9w0BAQ0FADA2MQswCQYDVQQGEwJ1czEL
          MAkGA1UECAwCQ0ExDDAKBgNVBAoMA2lkcDEMMAoGA1UEAwwDaWRwMB4XDTE5MDQy
          NjE4NTIxOFoXDTIwMDQyNTE4NTIxOFowNjELMAkGA1UEBhMCdXMxCzAJBgNVBAgM
          AkNBMQwwCgYDVQQKDANpZHAxDDAKBgNVBAMMA2lkcDCBnzANBgkqhkiG9w0BAQEF
          AAOBjQAwgYkCgYEA1mKmlbr/SiHOhgdROpYeze96mw0WbO+BdJYDceeuNkaw0zOU
          CKZI6TNgrNsqEnLOyWYy5ywA9XA6Ni2qQTuKqapsMT3I1s9DMUg2ln7tTzNdhE02
          fY4GVjiCw7i9YJ+cgcMZh8qL0yoilrLpRLzLrRC6rApqYfEwn+5FPKtTt7cCAwEA
          AaNQME4wHQYDVR0OBBYEFNvFMRtHJ4D327dbRbxhWceXnwd0MB8GA1UdIwQYMBaA
          FNvFMRtHJ4D327dbRbxhWceXnwd0MAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQEN
          BQADgYEAX0I5zpGqI7vzzs8CDyokux1JZzfu+O3P5GfOwUaIG9y01FzxgbL2MRKQ
          oTXMAed97Q6vHA5cffvteu/rPcerpGmFj5h3wv5u+D0ch5s/Mk/Ug6S+x6k3CC+P
          kHimi6OEslFecDMhghUtPJAmhOGnTRwLr7hVeJXBHXWCTXA7aGE=
          </ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </ds:Signature>
    <saml2:Subject>
      <saml2:NameID>
      jsmith@example.com
      </saml2:NameID>
      <saml2:SubjectConfirmation>
        <saml2:SubjectConfirmationData
        InResponseTo="bcf0b634-67b4-4dc9-a436-4e5cfcfb80e2"
        NotOnOrAfter="2019-04-18T18:56:46.730Z"
        Recipient="https://sp.example.com/saml/acs" />
      </saml2:SubjectConfirmation>
    </saml2:Subject>
    <saml2:Conditions
    NotBefore="2019-04-18T18:46:46.730Z"
    NotOnOrAfter="2019-04-18T18:56:46.730Z"
    xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">
    </saml2:Conditions>
    <saml2:AuthnStatement
    AuthnInstant="2019-04-18T18:51:46.729Z"
    SessionIndex="bcf0b634-67b4-4dc9-a436-4e5cfcfb80e2">
    </saml2:AuthnStatement>
    <saml2:AttributeStatement>
      <saml2:Attribute Name="logins">
        <saml2:AttributeValue>
        root
        </saml2:AttributeValue>
        <saml2:AttributeValue>
        jsmith
        </saml2:AttributeValue>
      </saml2:Attribute>
      <saml2:Attribute Name="groups">
        <saml2:AttributeValue>
        admins
        </saml2:AttributeValue>
        <saml2:AttributeValue>
        developers
        </saml2:AttributeValue>
      </saml2:Attribute>
    </saml2:AttributeStatement>
  </saml2:Assertion>
</saml2p:Response>

```

When a service provider receives an authentication response, it should first check if the `InResponseTo` attribute references an `ID` of an `AuthnRequest` that the service provider actually sent. In addition, the `IssueInstant` attribute should be used to scope the validity window of the response. The reasoning for why these are important has already been explained in the previous section.

The ACS URL to which the authentication response is delivered performs no authentication of the client, this is why the `Signature` in the authentication response must be checked to ensure the client is the identity provider. For readers familiar with the concept of webhooks, this is conceptually very similar: information is exchanged out-of-band ahead of time (often an API key in the case of webhooks) which is then used to authenticate the client.

The response also contains four new interesting tags: `Status`, `Subject`, `Conditions`, and `AttributeStatement`:

* The `Status` tag contains the authentication result: success or failure.
* The `Subject` tag identifies the principal that just authenticated. As seen in Figure (4), the `NameID` tag contains the email address of the principal that authenticated: `jsmith@example.com`.
* The `Conditions` tag defines constraints on the assertions, for example the `NotBefore` and `NotOnOrAfter` attributes define the time duration that the assertions are valid. This prevents a malicious actor from recording a valid authentication response and then replaying at a later time (perhaps after the principals account has been removed).
* The `AttributeStatement` tag contains assertions that the identity provider is making about the principal. As mentioned previously, assertions typically contain information like group membership within an organization, allowed logins, or any other relevant identifying information about the principal. In Figure (4), the principal is part of the groups `admins` and `developers` and is allowed to login as `root` and `jsmith`.

As mentioned above, it's important to keep in mind assertions about a principal are a snapshot in time of identifying information. Providing long lived assertions or sessions is, in general, a security risk. Keep assertions and sessions reasonably short lived and force re-authentication to ensure that any assertions that the identity provider made about the principal are still valid.

---

## Adding SAML based SSO to your app.

This post was a deep dive into how SAML authentication works, but luckily developers can use open-source libraries to quickly and easily add SAML to applications.

These are a few top libraries for adding SAML to your app.

* For Node.js try `node-saml`
* For PHP Apps try `simpleSAML`
* For Ruby & Rails Apps try `ruby-saml`
* For GoLang Apps try `SAML`
* For Django Apps try `django-saml2-auth`

The above list is a starting off point for adding SAML to your application. Since a SAML library is key for authentication it's important to pick a library which is being maintained and has mitigated any possible security issues or CVEs.

---

## Conclusion

What a service provider does with the response is up to the discretion of the service provider. However, typically a successful response will create a session on an internal service providing a single source of identity and consistent authentication across internal services.

One common example is a service providers wanting to know which user group a principal belongs to. This allows to implement role-based access control policy, which says that only users from "SSH" group should be given access to the production SSH environment. A different policy can be implemented for accessing Kubernetes clusters, or the CI/CD pipeline, etc.

Generally, SAML authentication solves three important problems:

1. **SAML offers significant improvement to user experience.** Users only have to remember their credentials with a single identity provider and not having to worry about user names and passwords for every application they use.
2. **SAML allows application developers to outsource identity management and authentication implementation to external providers** and not having to implement it themselves.
3. **And perhaps most importantly, SAML dramatically reduces the operational overhead of managing access within an organization.** If an employee leaves or transfers to another team, their access will be automatically revoked or downgraded across all applications connected to the identity provider.

Hopefully that provides a good snapshot of how SAML works. If you’d like to learn more about how we integrate with various SSO services you can read more on how to implement SSO for SSH Access or on how to implement SSO for Kubernetes.

---

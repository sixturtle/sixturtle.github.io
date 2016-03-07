---
layout: post
title:  "Secure JAX-RS API with JWT"
date:   2015-02-01 14:05:48 -0500
categories: blog
description: Java, JEE, JAX-RS, Web Services, REST, JWT, JSON Web Token, OpenID Connect, OAuth, Security 
---
Developing RESTful APIs using JAX-RS is not that difficult. However, authentication 
and authorization of these APIs require additional consideration.  In this article, 
we will see how an API, that is developed using JAX-RS, can be secured using
[OpenID Connect]. The complete working source code can be downloaded from 
[GitHub]. This code is built using [Nimbus] library to parse and verify a [JWT].

While the solution presented here is specific to JAX-RS based RESTful service but the 
same concept (and most code) can be applied to protect a RESTful service in any Web container
that supports HTTP filter.

![Authentication Architecture](/res/jwt.png)

# 1. Identity Provider (IDP)

Identity Provider is the central part of the security architecture. There are various 
open source OIDC solutions available, e.g. [Keycloak], in addition to various popuplar 
social providers like Google, Facebook and Twitter. 

# 2. Relying Party (Client)

Relying party (RP) is the web, mobile application that requests the identity token from
an OpenID Connect (OIDC) provider. An RP requires one time registration with the IDP 
and needs to provide Web Origin and Redirect URLs. At least this is the case with [Keycloak]. 
How an RP gets the identity token from an IDP is not the scope of this article. However, 
in case of [Keycloak], it provides a handy JavaScript that can be put into the web 
application to obtain an identity token via Web Browser which initiates HTTP redirects 
for authentication and then get redirected by IDP back to the Web application with the 
identity token.

# 3. Identity Token (JWT)

The identity token obtained from an OIDC server is a standard JSON Web Token ([JWT]) 
packaged in a simple JSON object.

{% highlight json %}
{
  "sub"       : "username",
  "iss"       : "https://your-idp-server",
  "aud"       : "client-12345",
  "nonce"     : "n-0S6_WzA2Mj",
  "auth_time" : 1311280969,
  "acr"       : "c2id.loa.hisec",
  "iat"       : 1311280970,
  "exp"       : 1311281970
}
{% endhighlight %}

The IDP can be configured to return more claims like email, first name, last name, 
organization etc. in the JWT.


# 4. JAX-RS Filter for JWT

A basic solution to secure a set of REST endpoints running in a Java Web container is to 
install a security filter. A typical security filter intercepts all the incoming requests 
and examins authentication and applies authorization logic based on the security context.


* **JWT Filter**

    The JAX-RS API allows us to subclass `javax.ws.rs.container.ContainerRequestFilter` to 
    intercept incoming request to a REST endpoint. It also allows us to apply a name-binding
    annotation to specific REST endpoint so that the filter does not intercept each and 
    every service request.

    {% highlight java %}
    @NameBinding
    @Target({ ElementType.TYPE, ElementType.METHOD })
    @Retention(RetentionPolicy.RUNTIME)
    public @interface JWTSecured {

    }


    @Provider
    @Priority(Priorities.AUTHENTICATION)
    @JWTSecured // optional marker to limit the scope of this filter to methods matching with this annotation
    public class JWTRequestFilter implements ContainerRequestFilter { 
        private Pattern     tokenPattern  = Pattern.compile("^Bearer$", Pattern.CASE_INSENSITIVE);
        private JWSVerifier jwsVerifier;
        ...

        public JWTRequestFilter() throws Exception {
            ...
            PublicKey publicKey = loadPublicKey(keystore, password, alias);
            if (publicKey != null) {
                jwsVerifier = new RSASSAVerifier((RSAPublicKey) publicKey);
            } else {
                throw new RuntimeException("Configuration error: unable to load JWT signing public key from keystore: " + keystore);
            }
        }

        @Override
        public void filter(final ContainerRequestContext requestContext) throws IOException {
            boolean isSecured = false;
            String authorizationHeader = requestContext.getHeaderString(HttpHeaders.AUTHORIZATION);
            if (authorizationHeader != null) {
                String token = parseBearerToken(authorizationHeader);
                if (token != null) {
                    JWTClaimsSet claims     = validateToken(token);
                    JWTPrincipal principal  = buildPrincipal(claims);
                    if (principal != null) {
                        // Build and inject JavaEE SecurityContext for @RoleAllowed, isUserInRole(), getUserPrincipal() to work
                        JWTSecurityContext ctx = new JWTSecurityContext(
                                                        principal,
                                                        requestContext.getSecurityContext().isSecure());
                        requestContext.setSecurityContext(ctx);
                        isSecured = true;
                    } 
                }
            }
            if (!isSecured) {
                throw new NotAuthorizedException(
                        "Unauthorized ",
                        Response.status(Status.UNAUTHORIZED));
            }
        }

        private JWTClaimsSet validateToken(final String token) {
            JWTClaimsSet claims = null;
            boolean isSecured = false;
            try {
                JWT jwt = JWTParser.parse(token);
                if (jwt instanceof SignedJWT) {
                    SignedJWT signedJWT = (SignedJWT) jwt;
                    if (signedJWT.verify(jwsVerifier)) {
                        claims = signedJWT.getJWTClaimsSet();
                        log.debug("JWT claims: {}", claims.getClaims());

                        Date expirationTime = claims.getExpirationTime();
                        Date now = new Date();
                        Date notBeforeTime = claims.getNotBeforeTime();
                        if (notBeforeTime.compareTo(now) > 0) {
                            throw new NotAuthorizedException(
                                        "Unauthorized: too early, token not valid yet",
                                        Response.status(Status.UNAUTHORIZED));
                        }
                        if (expirationTime.compareTo(now) <= 0) {
                            throw new NotAuthorizedException(
                                        "Unauthorized: too late, token expired",
                                        Response.status(Status.UNAUTHORIZED));
                        }
                        isSecured = true;
                    }
                }
                if (!isSecured) {
                    throw new NotAuthorizedException(
                            "Unauthorized",
                            Response.status(Status.UNAUTHORIZED));
                }
            } catch (ParseException | JOSEException e) {
                throw new NotAuthorizedException(
                            e.getMessage(),
                            Response.status(Status.UNAUTHORIZED),
                            e);
            }
            return claims;
        }
        ...
    }
    {% endhighlight %}

* **Security Context**
   
    We can implement a custom `javax.ws.rs.core.SecurityContext` to stuff in the 
    claims received from JWT so that rest of the JEE stack can work with the standard
    annotation `@RolesAllowed` and API `JWTPrincipal p = (JWTPrincipal) securityContext.getUserPrincipal();`
    
    {% highlight java %}
    public static class JWTSecurityContext implements SecurityContext {
        private JWTPrincipal principal;
        private boolean      isSecure;
        private Set<String>  roles = new HashSet<>();

        public JWTSecurityContext(final JWTPrincipal principal, final boolean isSecure) {
            this.principal  = principal;
            this.isSecure   = isSecure;
            String[] names  = principal.getRoles();
            for (int iIndex = 0; names != null && iIndex < names.length; ++iIndex) {
                roles.add(names[iIndex]);
            }
            names = principal.getOrganizations();
            for (int iIndex = 0; names != null && iIndex < names.length; ++iIndex) {
                roles.add(names[iIndex]);
            }
        }

        @Override
        public String getAuthenticationScheme() {
            return "JWT"; // informational
        }

        @Override
        public Principal getUserPrincipal() {
            return principal;
        }

        @Override
        public boolean isSecure() {
            return isSecure;
        }

        @Override
        public boolean isUserInRole(final String role) {
            return roles.contains(role);
        }
    } 
    {% endhighlight %}

* **Principal**
    
    A custom implementation of `java.security.Principal` can be used to map claims 
    from JWT and then used in the custom `SecurityContext`.
 
    {% highlight java %}
    public class JWTPrincipal implements Principal {
        private String name;
        private String email;
        private String firstName;
        private String lastName;
        private String[] organizations;
        private String[] roles;
        ...
    }
    {% endhighlight %}

    {% highlight java %}
    private JWTPrincipal buildPrincipal(final JWTClaimsSet claims) {
        JWTPrincipal principal = null;

        try {
            if (claims != null) {
                String subject   = claims.getSubject();
                String email     = (String) claims.getClaim("email");
                String firstName = (String) claims.getClaim("given_name");
                String lastName  = (String) claims.getClaim("family_name");

                // TODO: Extract custom attributes, e.g. roles, organization affiliation etc. and put into principal.

                principal = new JWTPrincipal(subject, email, firstName, lastName);
            }
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
        return principal;
    }
    {% endhighlight %}


* **Protect a sample REST API**
    
    This example shows how to use standard JEE annotation and API to protect a REST API.
    
    {% highlight java %}
    @Path("/echo")
    @Produces({ MediaType.TEXT_PLAIN })
    public class EchoAPI {
        @Context
        protected SecurityContext securityContext;

        @GET
        @JWTSecured // limits filter to be applied to just this method
        @RolesAllowed("USER") // just for demonstration, check JWTRequestFilter to see what roles are injected to security context
        public Response echo(@QueryParam("message") String message) {
            JWTPrincipal p = (JWTPrincipal) securityContext.getUserPrincipal();
            // TODO: inspect principal for fine grain security before proceeding
            return Response.ok().entity(message).build();
        }
    }
    {% endhighlight %}

The source code can be downloaded from [GitHub].

[GitHub]: https://github.com/sixturtle/examples/tree/master/jaxrs-jwt-filter "Source Code"
[OpenID Connect]: http://connect2id.com/learn/openid-connect "Open ID Connect Explained"
[Keycloak]: http://keycloak.jboss.org/ "Red Hat Keycloak"
[JWT]: https://jwt.io/
[Nimbus]: http://connect2id.com/products/nimbus-jose-jwt

# GWT RPC-Auth / XSRF API

## Goals
  * Provide a framework that integrates best-known practices for securing RPC requests from AJAX apps
  * Simple integration with existing authorization backends / frameworks
    * OAuth : http://code.google.com/apis/accounts/docs/OAuth.html
    * Simple server-provided username/password style
  * API style should act as an enhancement of existing GWT RPC API
    * An auth-enabled RPC endpoint should be usable without an auth provider (for hosted-mode, testing)

## Non-goals
  * Provide an implementation of an authorization backend
  * Provide application-level or business-logic security

# Overview

RPC-Auth will provide an API and design patterns to allow developers to ensure that the RPC endpoint is being accessed by a duly-authorized client.  In this framework, the design will revolve around maintaining a session token that the client application will transmit as part of each future RPC request.

```
// The developer is free to extend the token, provided it remains a Serializable type
// Instances constructed on the client may not have as strong a UUID value
class AuthToken extends Serializable {
  // The time at which the token was created
  public final long getCreationTimeMillis() {...}

  // A unique identifier for the token object
  public final String getUUID() {...}

  //final hashCode() and equals() also provided based on UUID
  final int hashCode() { ... }
  final boolean equals(Object o) { ... }
}

// Implemented in addition to RemoteService by the RemoteServiceServlet
interface RequiresAuthorization<T extends AuthToken> {

  // Applied to individual methods to indicate that setAuthToken should
  // not be called.  Typically applied to a login() function.
  public @interface NoAuthorization{}

  // Guaranteed to be called after every RPC request
  void clearAuthToken();

  // This method will be called immediately before any RPC functions and
  // will be called with null if no token is present.
  // The exception allows for an early-out.
  void setAuthToken(T token, Method m) throws InvalidAuthTokenException;
}

// Add additional method to ServiceDefTarget; each ServiceDefTarget will
// have its own default AuthCallback that just replays the last token
// it was given
interface ServiceDefTarget {
  // Delegates to AuthCallback.getAuthToken().
  AuthToken getAuthToken();

  // This intentionally lacks a getAuthCallback()

  // Optional, used for cross-proxy authorization setup
  void setAuthCallback(AuthCallback callback);
}

// Add additional methods to RemoteServiceServlet
class RemoteServiceServlet {
  // May be called only during a request with an AuthToken to send back
  void setReturnAuthToken(AuthToken token);
}

// A convenience subclass
class AuthorizedRemoteServiceServlet<T extends AuthToken> extends RemoteServiceServlet implements RequiresAuthorization<T> {
  // Gets the AuthToken associated with the current request
  protected T getRequestAuthToken();

  // Implement the tricky methods, since they require thread-awareness
  public final void clearAuthToken();
  public final void setAuthToken(T token, Method m) throws InvalidAuthException;

  // Post-request cleanup
  protected void doClearAuthToken();

  // Allow the end-user to inject validation logic
  protected void doSetAuthToken(T token, Method m) throws InvalidAuthException;
}

// The developer isn't required to implement this.  We'll provide a default
// implementation that's just used to share an AuthToken between multiple
// RPC proxy objects.  Different RPC security domains can be established by using
// distinct AuthCallback objects.
interface AuthCallback<T extends AuthToken> {
  // Called by ServiceDefTarget to get the AuthToken to use
  T getAuthToken();

  // Called before the AsyncCallback's onFailure; the same exception
  // appears there as well unless an alternate Throwable is thrown.
  void onFailure(InvalidAuthTokenException e) throws Throwable;

  // Called before the AsyncCallback's onSuccess with the latest token value
  void onSuccess(T token);
}
```

Even though this document focuses mainly on the RemoteServiceServlet API, the behaviors described will be implemented via the existing RPC / RPCRequest utility classes, so these functions will be available to developers who are not using RemoteServiceServlet:

```
class RPCRequest {
  public <T extends AuthToken> T getAuthToken(Class<T> desiredType) throws InvalidAuthTokenException;
}
```

# Simple login scenario

Define the RemoteService (and associated async variant):

```
interface MyRemoteService extends RemoteService {
  @NoAuthorization
  void login(String username, String password);

  String restrictedMethod();
}
```

Define the servlet code:
```
class MyRemoteServiceImpl extends AuthorizedRemoteServiceServlet<AuthToken> implements MyRemoteService {

  void login(String username, String password) {
    if (username.equals(password)) {
      AuthToken authToken = new AuthToken();
      // Register authToken.getUUID() in some kind of session-tracking database
      setReturnAuthToken(authToken);
    }
  }

  String restrictedMethod() {
    return getRequestAuthToken().getUUID();
  }

  void doSetAuthToken(AuthToken t, Method m) {
    // Check token validity
  }
}
```

Client code:
```
MyServiceAsync svc = GWT.create(MyService.class);
svc.login("foo", "foo", someCallback);
// later on
svn.restrictedMethod(someCallback);
```

After the initial call to login() succeeds, the remote proxy object will always send the last AuthToken that it has received to the server.

# Custom AuthToken data

It will be convenient to associate additional data in the client or server with the AuthToken, without the need to maintain an additional Map or other tracking data.  This design allows the user to extend the AuthToken class in any way that is desired, with the caveats that the class must remain serializable and that the additional data will add to the overhead of every RPC request.

Define a custom subclass of AuthToken:
```
class MyAuthToken extends AuthToken {
  private String customerId;

  // This must remain Serializable
  private MyAuthToken() {}

  public MyAuthToken(String customerId) {
    this.customerId = customerId;
  }

  public getCustomerId() {...}
}
```

The RemoteServiceServlet definition is similar, but note the differences:

```
class MyRemoteServiceImpl extends AuthorizedRemoteServiceServlet<MyAuthToken> implements MyRemoteService {
  void login(String username, String password) {
    if (username.equals(password)) {
      String customerId = getCustomerIdForUsername(username);
      MyAuthToken authToken = new MyAuthToken(customerId);
      // Register authToken.getUUID() in some kind of session-tracking database
      setReturnAuthToken(authToken);
    }
  }

  String restrictedMethod() {
    return getRequestAuthToken().getUUID();
  }

  void doSetAuthToken(MyAuthToken t, Method m) throws InvalidAuthTokenException {
    // Check that the session-tracking db has t.getUUID(), that it's not expired, etc.
  }
}
```

The underlying plumbing will ensure that the correct type of AuthToken is passed into the developer-provided code or throws an InvalidAuthTokenException if the wrong type of AuthToken has been provided in the payload.

# Shared authentication domains

For sufficiently complicated apps, it may be the case that different RPC proxy objects will inhabit the same or different authorization realms.  Because each proxy object delegates to its AuthCallback object to maintain the AuthToken that will be sent with the request, we can place multiple RPC proxy objects in the same security realm by sharing a common AuthCallback instance among them.  Objects that must live in a disjoint security realm will necessarily have a different AuthCallback instance.

```
// DefaultAuthCallback is provided by GWT and is the built-in AuthCallback for remote proxy objects
AuthCallback realm1 = new DefaultAuthCallback();

MyServiceAsync svc1 = GWT.create(MyService.class);
MyServiceAsync svc2 = GWT.create(MyService.class);
MyOtherServiceAsync svc3 = GWT.create(MyOtherService.class);

ServiceDefTarget t1 = (ServiceDefTarget)svc1;
ServiceDefTarget t2 = (ServiceDefTarget)svc2;
ServiceDefTarget t3 = (ServiceDefTarget)svc3;
t1.setAuthCallback(realm1);
t2.setAuthCallback(realm1);
t2.setServiceEntryPoint("/other/URL");
t3.setAuthCallback(realm1);
```

In this example, calling login() on any of the async services will allow any of the other services to pass the same authorization data to the server.  If the server has a time-based expiration policy for sessions, using any of the RPC endpoints will be sufficient to keep the session alive.
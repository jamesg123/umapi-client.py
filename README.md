# cce-umapi
Python User Management API Interface for Creative Cloud Enterprise

# Installation

1. Clone this repository or download the latest stable release.
2. From the command line, change to the `cce-umapi` directory.
3. To install, run the command `python setup.py install`.  [**NOTE**: You may need admin/root privileges (using `sudo`) to install new packages in your environment.  It is recommended that you use `virtualenv` to make a virtual python environment.  See the [virtualenvwrapper documentation](http://virtualenvwrapper.readthedocs.io/en/latest/index.html) for more information]
4. If you encounter errors that repor missing library files, you may need to install the packages `python-dev`, `libssl-dev` and `libffi-dev` (these are the Debian package names - your environment may vary).
5. (**optional**) To run tests, use the command `python setup.py nosetests`.

# Getting Started

Before making calls to the User Management API, you must do the following preparation steps:

1. Obtain admin access to an Adobe Enterprise Dashboard.
2. Set up a private/public certificate pair
3. Create an integration on Adobe.io

Step 1 is outside of the scope of this document.  Please contact an administrator of your stage or prod Dashbord environment to obtain access.

Steps 2 and 3 are outlined in the documentaiton available at [adobe.io](http://www.adobe.io).

Once access is obtained, and an integration is set up, you will need the following configuration items:

1. Organization ID
2. Tech Account ID
3. IMS Hostname
4. IMT Auth Token Endpoint (JWT Endpoint)
5. API Key
6. Client Secret
6. Private Certificate

Most of this information should be availble on the adobe.io page for your integration.

Once these initial steps are taken, and configuration items are identified, then you will be able to use this library to make API calls.

## Step 1 - Create JSON Web Token

The JSON Web Token (JWT) is used to get an authorization token for using the API.  The `JWT` object will build the JWT for use with the `AccessRequest` object.

```python
from umapi.auth import JWT

jwt = JWT(
  org_id,     # Organization ID
  tech_acct,  # Techincal Account ID
  ims_host,   # IMS Host
  api_key,    # API Key
  open(priv_key_filename, 'r')  # Private certificate is passed as a file-like object
)
```

## Step 2 - Use AccessRequest to Obtain Auth Token

The `AccessRequest` object uses the JWT to call an IMS endpoint to obtain an authorization token.

```python
from umapi.auth import AccessRequest

token = AccessRequest(
  "https://" + ims_host + ims_endpoint_jwt,   # Access Request Endpoint (IMS Host + JWT Endpoint)
  api_key,        # API Key
  client_secret,  # Client Secret
  jwt()           # JWT - note that the jwt object is callable - invoking it returns the JWT string expected by AccessRequest
)
```

**NOTE**: It is recommended that you only request a new authorization token when the existing token has expired, and not to request a new token with each call or batch of calls.  Once the token is generated, the `AccessRequest` object provides a `datetime` object representing the expiration timestamp of the current token.

```python
token.expiry
```

This timestamp can be used to create a persistent token store, so that the token can be retrieved locally and reused if it has not expired (such a persistant store system is outside the scope of this library).

## Step 3 - The Auth Object

The `Auth` object is used by a lower-level library to build the necessary authentication headers for making an API call.  It only requires the API key and auth token (generated by `AccessRequest`).

```python
from umapi.auth import Auth

token = AccessRequest( ... )
auth = Auth(api_key, token())  # note that the AccessRequest object is callable - the Auth object requires the token string, not the object itself
```

# Making Calls

Once the Auth object is built, the UMAPI object can be used to make API calls.  

## Get a List of Users

```python
from umapi import UMAPI

# api_endpoint - full endpoint URL (e.g. https://usermanagement-stage.adobe.io/v2/usermanagement)
# auth - Auth object
api = UMAPI(api_endpoint, auth)

users = api.users(org_id, page=0) # Organization ID is required, page is optional and defaults to 0)
```

## Get a List of Groups

```python
from umapi import UMAPI

# api_endpoint - full endpoint URL (e.g. https://usermanagement-stage.adobe.io/v2/usermanagement)
# auth - Auth object
api = UMAPI(api_endpoint, auth)

groups = api.groups(org_id, page=0) # Organization ID is required, page is optional and defaults to 0)
```

## Perform an Action

This example introduces the `Action` object, which represents an action or group of actions to perform on a user.  These actions include, but are not limited to, user creation, addition or removal of product entitlements, user update, and user deletion.

```python
from umapi import UMAPI, Action

api = UMAPI(api_endpoint, auth)

action = Action(user="user@example.com").do(
  createAdobeID={"email": "user@example.com"}
)

status = api.action(org_id, action)
```

## Perform a Multi-Step Action

```python
from umapi import UMAPI, Action

api = UMAPI(api_endpoint, auth)

action = Action(user="user@example.com").do(
  createAdobeID={"email": "user@example.com"},
  add=["product1", "product2"]
)

status = api.action(org_id, action)
```

## Perform Multiple Actions in One Call

A group of Actions can be wrapped in some type of collection or iterable (typically a list) and passed to `UMAPI.action`.

```python
from umapi import UMAPI, Action

api = UMAPI(api_endpoint, auth)

actions = [
    Action(user="user@example.com").do(
        remove=["product1"]
    ),
    Action(user="user@example.com").do(
        add=["product2"]
    ),
]

status = api.action(org_id, actions)
```

# Library API Documentation

## Core Objects

### UMAPI

Main User Management API interface class.  Used to make calls to the API.

Requires endpoint URL and auth object.

Example:

```python
api = UMAPI(
  endpoint="https://usermanagement-stage.adobe.io/v2/usermanagement",
  auth=Auth( ... )
)
```

Any call made this this object can raise the `UMAPIError` and `UMAPIRetryError`.  `UMAPI.action` can additionally raise `UMAPIRequestError` for responses containing an error result type, and `ActionFormatError` when an invalid action object is provided.

#### `UMAPI.users`

Get a list of users.

Requires the org_id.  Takes page number as an optional parameter (default=0).

Example:

```python
users = api.users(
  org_id="test_org_id",
  page=0
)['users']
```

The oject returned is the full API response object.  The main aspect is the "users" key, which contains a list of all users for the organization.

Example list of users:

```python
[{
    u'status': u'active',
    u'firstname': u'Example',
    u'lastname': u'User',
    u'groups': [u'Example Group'],
    u'country': u'US',
    u'type': u'enterpriseID',
    u'email': u'user@example.com'
}]
```

The list will contain up to 200 users.  The repsonse object also contains the "lastPage" property, which indicates if the end of the list has been reached.  If the organization contains more than 200 users, then the `lastPage` property can be used to paginate through all user data (by using the `page` parameter to `UMAPI.users`).

#### `UMAPI.groups`

Get a list of permission/product entitlement groups.

Requires the org_id.  Takes page number as an optional parameter (default=0).

Example:

```python
groups = api.groups(
  org_id="test_org_id",
  page=0
)['groups']
```

The oject returned is the full API response object.  The main aspect is the "groups" key, which contains a list of all groups for the organization.

Example list of groups:

```python
[{
  u'memberCount': u'68',
  u'groupName': u'Administrators'
},{
  u'memberCount': u'4',
  u'groupName': u'Group 2'
}]
```

Also of interest is the `lastPage` attribute of the response object.  See the `UMAPI.users` section for more details.

#### `UMAPI.action`

Perform some kind of action - create users, add/remove groups, edit users, etc.  `UMAPI.action` depends on the Action object, which is detailed in the Action section of this documentation.

Requires both the org_id and action parameters.

Example:

```python
action = Action(user="user@example.com").do(
  update={"firstname": "Example", "lastname": "User"}
)
result = api.action(
  org_id="test_org_id",
  action=action
)
```

The result object returned is the complete result object returned by the User Management API.  If the response contains a result type of "error", then the action call will raise a `UMAPIRequestError`.  Success and partial result types do not raise any exceptions.

The exact format of the result object is detailed on the [adobe.io documentation page for Management calls](https://www.adobe.io/products/usermanagement/docs/api/manageref).

### Action

The `Action` object models input to the UMAPI Management interface.  More specifically, it models an element of the action array that serves as the top-level object to management calls.

Example:

```python
Action(user="user@example.com").do(
  addAdobeID={"email": "user@example.com"}
)
```

This Action object models the object needed to create an Adobe ID.  It is converted internally to this JSON:

```json
{
  "user": "user@example.com",
  "do": [
    {
      "addAdobeID": {
        "email": "user@example.com"
      }
    }
  ]
}
```

The Action object always requires the user ID, but supports additional top-level action properties such as `requestID` and `domain`.  It reads these attributes from `**kwargs` so there are no restrictions on which of these attributes can be provided.

Example Python:

```python
Action(user="user", requestID="abc123", domain="example.com").do(
  createEnterpriseID={"email": "user@example.com"}
)
```

Equivalent JSON:

```json
{
  "user": "user",
  "requestID": "abc123",
  "domain": "example.com",
  "do": [
    {
      "addAdobeID": {
        "createEnterpriseID": "user@example.com"
      }
    }
  ]
}
```

#### `Action.do`

The `Action` object has one method - `do`.  This is used to define a list of actions to perform on the user for the call.

Like the object constructor, `do()` gets its parameters from `**kwargs`.  However, there are no required attributes.

The name of each `do()` argument should correspond to a key of the "do" container.  Those keys include "addAdobeID", "createEnterpriseID", "add", "remove", etc.  Refer to the [official documentation](https://www.adobe.io/products/usermanagement/docs/gettingstarted) for more details.

`do()` parameter values should be objects (strings, dicts, lists, etc) strucutred in the way expected by the API.

Example:

```python
Action(user="user", domain="example.com").do(
  createEnterpriseID={"email": "user@example.com", "firstname": "Example", "lastname": "User"}
)
```

In that example, the "createEnterpriseID" portion of the JSON would render like this:

```json
"createEnterpriseID": {"email": "user@example.com", "firstname": "Example", "lastname": "User"}
```

The exact structure of the `createEnterpriseID` parameter object is preserved.

## umapi.auth

### JWT

### AccessRequest

### Auth

## umapi.error

### UMAPIError

### UMAPIRetryError

### UMAPIRequestError

### ActionFormatError

# Django login + OIDC Authorization/Access Token Exchange
# Example Workflow
1. Login
Minimal psuedocode:  
`curl -X POST BASE_URL/login/ "credentials: include" "headers: Accept: application/json" "body: { <email>, <password> }"`

2. Create and store `code_challenge` and `code_verifier` on client-side. This verifies that the client doesn't change during interaction with the server.
See examples for [creating both items](https://github.com/ARPA-H-FEAST/smart-server/blob/main/interface-prototype/src/utilities/pkce.js) and [storing in-browser](https://github.com/ARPA-H-FEAST/smart-server/blob/main/interface-prototype/src/main.js#L33-L40). 

3. OIDC Authorize
User authorizes that OIDC activities are in-scope for the session. See [authorization-grant](#authorization-grant) below. This redirects to a page served by the OAuth server.

4. Code exchange
After authorization, user is given an exchange code that must be exchanged for the `Bearer` token (a JWT). This process could be automated if desired. See [step 4 below](#authorization-grant)

# Technical details of above steps
### Establish connection
1. Should establish CSRF token
### Login: See below for user endpoints.
1. Successful login grants a user session cookie (browser-managed, no action needed by UI)
### Front-end needs to create two things and store in session/local storage:
1. `code_challenge` and `code_verifier`
2. See https://github.com/ARPA-H-FEAST/smart-server/blob/main/interface-prototype/src/main.js#L33-L40 and https://github.com/ARPA-H-FEAST/smart-server/blob/main/interface-prototype/src/utilities/pkce.js for code that should be adaptable
### Authorization grant
1. User clicks "Authorization"
2. `code_challenge`, `code_verifier`, and several hard-coded arguments are passed to the OAuth server via URL:  
```
await fetch(BASE_URL/oauth/authorize/?response_type=code&
code_challenge=${code_challenge}&
code_challenge_method=S256&
client_id=hardcoded-client-id&
redirect_uri=hardcoded-redirect-on-authorization
)
```

1. Page is redirected to client "Authorization" page, as configured in "Application Registration" (redirect URL is automated to the above `hardcoded-redirect-on-authorization`)
2. User scope authorization is defined in Django server code (see https://github.com/ARPA-H-FEAST/smart-server/blob/main/server/settings.py#L121-L127)
3. On user authorization, server returns a `code` field in the URL response
	1. This code must be stored by the frontend for use in the "Code Exchange" step, which follows
	2. In Vue, I have scraped this from the route URL, code is here: https://github.com/ARPA-H-FEAST/smart-server/blob/main/interface-prototype/src/views/Callback.vue#L52-L56
	3. React/JS will need to do something similar: The code is returned as `FRONTEND_URL/callback/?code=xxxx`  
4. The token must then be exchanged for the JWT, using `code` and `code_verifier`
```
        const res = await fetch(
          `BASE_URL/oauth/token/`,
          {
            headers: headers,
            method: 'POST',
            body: JSON.stringify(
              {
                client_id: [harcoded_client_id],
                code: 'code',
                code_verifier: code_verifier,
                redirect_uri: hardcoded-redirect-uri,
                grant_type: 'authorization_code',
              })
          }
      )
```
## JWT
Finally, the JWT is granted. All endpoints that route to FHIR must include the header `"Authorization": "Bearer: " + JWT`


# API Endpionts
## Root: `/api/<dest>/`
Will return CSRF token (on first hit); browser-managed but visible to JavaScript. See e.g. [cookies.js](https://github.com/ARPA-H-FEAST/smart-server/blob/54428072d24074e93b0cfb4e6132947b30c1d86c/interface-prototype/src/utilities/cookies.js#L3) for example of cookie parsing.
## Destinations (prefix `/api`)
`/users/`:
### Requests:
GET (no args)
### Responses:
{"error": "not logged in"}  
{"error": "other error"}  
if not admin:  
&nbsp;&nbsp;Array([user only])  
if admin:  
&nbsp;&nbsp;Array([serialized user objects])  

`create-user`: 
### Requests:
POST: {first_name: String, last_name: String, email: String, password: String, category: Int} (category is implemented as an enum, something like "1: Patient", "2: Researcher", etc.)
### Responses:
{"error": "error msg"}  
{"created": [username], [admin]: True [optional]}  

`update-user`: 
### Requests
1: Full update: POST: {first_name: String, last_name: String, email: String, password: String, category: Int} (category is implemented as an enum, something like "1: Patient", "2: Researcher", etc.)  
2: Update password: POST: {new_password: bool [True], old_password: String, new_password: String}
### Responses
{"error": "error msg"}  
{"update": [username], [admin: True]}  

`delete-user`: 
### Requests
POST: {user_id: Int} //  TODO: I will need to confirm this signature, let me know if this becomes blocking  
### Responses
{"username": None}  // TODO: This almost certainly needs attention

`login`: 
### Requsts:
POST: Object: { email, password }  
### Responses
{"detail":"error message"}  // TODO - "detail" should really be "error". Let me know if this becomes blocking  
{username: String, role: [Researcher|Patient|Clinician] (String), isAuthenticated: bool [True], [admin: True]}  

`logout`:
### Requests
GET (No args)  
### Responses
{"detail": "Not logged in" | "Successfully logged out"}   
`whoami`:
### Requests
GET (no args)  
## Responses
if not logged in:  
	{"isAuthenticated": False}  
else:  
	{"isAuthenticated": True, username: String, role: [Researcher|Patient|Clinician]String}  

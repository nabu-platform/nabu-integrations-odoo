# Endpoints

They are all basically the same thing to the fact that they are just wrappers for the Odoo internal API, but these wrappers are slightly different in how the protocols work:

/xmlrpc is for RPC calls where the POST body must be XML; credentials are part of the POST

/jsonrpc is for RPC calls where the POST body must be JSON; credentials are part of the POST

/web/dataset is for RPC calls where the POST body must be JSON and is basically the same as /jsonrpc except that the TOKENIZED credentials are passed in the session_id cookie; this makes it the proper API to use in a web based application, provided that its running under the exact same host-name, so that you don't have CORS headaches. In this scenario, you typically have a reverse proxy running your web app under some specific path on the server. For example odoo.example.com/odoo might serve your standard Odoo deployment, but odoo.example.com/vuejs might serve your Single Page Application. Since they are both under "odoo.example.com" ... no CORS preflight checks are made.

## Behavior on /web/dataset/call_kw


The documentation indicates that the login sends back a session_id cookie (which it does) and a uid (which it does) which can be used on subsequent calls.
However any combination of session_id with uid, login, password, database,... does NOT work, it ends in a "session expired" exception.
Only when we explicitly do NOT send the session_id along, does it start to work BUT...

At the time of writing (2023-10-09), it seems that odoo authenticates the http connection the login occurs on.

Given these facts:

- if you do a login followed by a call in the same run, it works, even if no session_id cookies etc are sent in the second call
- if you skip the login call, it fails (so the login does _something_)
- if you log in in transactionA and call something else in transactionB, it will fail presumably because the connection is not reused
- if you log in and call a in the same run, we established that it works. If you then asynchronously (so in a separate call) but within say a second call a again, it no longer works.

So it seems the HTTP connection itself is authenticated and subsequent calls on that connection are valid.
It is hard to guarantee connection reuse because this is generally not important, however as long as you do not perform concurrent calls, it should probably reuse the connection.
We have explicitly configured a http client with 1 max connection per target as well.


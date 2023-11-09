# About

At the time of writing the saas version is at 16.2 (even though there are newer versions available in git).
You can check the version in the errors that odoo sends back.

All api calls seem to enter via: https://github.com/odoo/odoo/blob/saas-16.2/odoo/api.py

They are then dispatched to the specific models, e.g. https://github.com/odoo/odoo/blob/saas-16.2/addons/account/models/account_account.py

The kwargs are mapped by name to the method in the model, so for instance if we want to call this method:

```
def create(self, vals_list)
```

We need a kwargs with a field ``vals_list``. It seems that they have kept the parameter names in sync across models to ensure kwargs mappings should work for all models.

In general you can pass in EITHER args OR kwargs, but not both as they conflict. Args seem to be applied in order where kwargs are applied by name.
Most existing examples use args rather than kwargs.

## Create issues

When we want to create a new item we pass through this function in api.py:

```
def _call_kw_model_create(method, self, args, kwargs):
	# special case for method 'create'
	context, args, kwargs = split_context(method, args, kwargs)
	recs = self.with_context(context or {})
	_logger.debug("call %s.%s(%s)", recs, method.__name__, Params(args, kwargs))
	result = method(recs, *args, **kwargs)
	return result.id if isinstance(args[0], Mapping) else result.ids
```

We were creating something with a kwargs that contained a vals_list, however the last line in that function accesses args[0] directly which actually ended in an out of bounds exception.
You can't just send a dummy value in args because it can not be combined with kwargs.

So instead for create we _have_ to use args. However, if you send in args, you can not entirely ommit kwargs (you will get a missing required parameter exception) nor send a null value for kwargs because this method is called which will end up with a sort of null pointer exception :

```
def split_context(method, args, kwargs):
	""" Extract the context from a pair of positional and keyword arguments.
		Return a triple ``context, args, kwargs``.
	"""
	# altering kwargs is a cause of errors, for instance when retrying a request
	# after a serialization error: the retry is done without context!
	kwargs = kwargs.copy()
	return kwargs.pop('context', None), args, kwargs
```

```
kwargs = kwargs.copy()
AttributeError: 'NoneType' object has no attribute 'copy'
```

Ironically even though the return value of the create has an "id" field, that field is null. Instead the actual id is contained in the "result" field.

# Endpoints

Quote:

"
	They are all basically the same thing to the fact that they are just wrappers for the Odoo internal API, but these wrappers are slightly different in how the protocols work:

	/xmlrpc is for RPC calls where the POST body must be XML; credentials are part of the POST

	/jsonrpc is for RPC calls where the POST body must be JSON; credentials are part of the POST

	/web/dataset is for RPC calls where the POST body must be JSON and is basically the same as /jsonrpc except that the TOKENIZED credentials are passed in the session_id cookie; this makes it the proper API to use in a web based application, provided that its running under the exact same host-name, so that you don't have CORS headaches. In this scenario, you typically have a reverse proxy running your web app under some specific path on the server. For example odoo.example.com/odoo might serve your standard Odoo deployment, but odoo.example.com/vuejs might serve your Single Page Application. Since they are both under "odoo.example.com" ... no CORS preflight checks are made.
"



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






Links:

https://www.odoo.com/documentation/12.0/developer/howtos/backend.html#json-rpc-library

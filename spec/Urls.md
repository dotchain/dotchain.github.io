# DOT URLs

All DOT models and their contents are referred to by using regular
URLs.

For example, the model for a single task could be something like:

    https://example.com/dot/tasks/123


## Websocket URL and Model ID

Any DOT client (such as [dotjs](https://github.com/dotchain/dotjs))
needs to figure out the websocket URL that implements the [DOT
Protocol](Protocol.md).

The websocket URL corresponding to a model URL is simply the same URL
with the last element of the path component stripped out.  And the
protocol scheme is replaced with the respective websocket scheme.  For
the task model URL, the corresponding websocket URL would be:

     wss://example.com/dot/tasks
    

The last path component is the Model ID.  In the task url
`https://example.com/dot/tasks/123`, the model ID would be `123`. The
client should not send the full URL in the [DOT protocol](Protocol.md)
messages but instead, only use the last path component.

### URL encoding

For the purposes of the [DOT Protocol](Protocol.md), all the Model
IDs should be specified after **decoding** the path component of any
URL encodings.

### Query parameters

Query parameters should not be used with model URLs. Servers and
clients should strip them if the application uses them.

## Fragments and references

It is often useful to refer to sub-fields of a model.  Since models
have a virtual JSON representation, there is a natural mechanism to
refer to a sub-tree of the model -- by use of the URL [fragment
identifier](https://en.wikipedia.org/wiki/Fragment_identifier).

For example, to refer to the email of the author of task, the
corresponding URL could be:

    https://example.com/dot/tasks/123#author/email


Note the use of slashes within the fragment. The whole fragment is
first split into parts using slash as a separator and this constitutes
the **Path** (see [Operations](Operations.md)) in the virtual JSON
representation of the model.

Clients SHOULD support subscribing and referring to objects via the
fragment -- it is a simple mechanism to refer to only the contents
scoped to that section of the model.

## Meta data

Most models will have a set of related data:

| Name | Purpose |
|------|---------|
| auth | authentication and authorization info about this model |
| meta | meta-data such as ContentType and renderer |
| sessions | set of users who are currently active on this model |
| agents | agents configuration for this model |

Most of these associated data are also often presented as models and
so require a unique URL.  Their URLs are formed by simply suffixing
the model URL with a period followed by the name of the associated
data. For the standard example, the above related data URLs would look
like this: 


| Related data | Related data URL |
|--------------|------------------|
| auth         | https://example.com/dot/tasks/123.auth |
| meta         | https://example.com/dot/tasks/123.meta |
| sessions     | https://example.com/dot/tasks/123.sessions |
| agents       | https://example.com/dot/tasks/123.agents |


**Notes**

1. Some model URLs may already have dots in their last path
component. This is allowed.

2. When using fragments, the fragment refers to the contents
of the associated data rather the original data.

3. These associated data cannot be chained.  That is, there is no
`123.auth.meta` for example.

## Versions

It is useful to refer to a model snapshot at a particular version. The
`@` symbol is used to specify the version, like so:

    https://example.com/dot/tasks/123@abcd-efgh#author

Note that it is valid to use a fragment with a versioned URL.  It is
invalid to use this with the [DOT Protocol](Protocol.md) though since
that is strictly about live models.

Note that there are no hard rules for version IDs -- it is suggested
that all clients stick to simple hex characters for IDs to make it
work well with URLs like the above.

## Fetching model URLs

All the URLs presented here should be fetch-able via HTTP(s) directly.
Fetching them should return the latest available (the specified
version) snapshot JSON data encoded as Mime Type `application/json`

## Issues

The authentication scheme for fetching URLs is not well defined.




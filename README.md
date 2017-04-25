# Django-Channels
Traditionally, django is simply a request-response framework.
https://heroku-blog-files.s3.amazonaws.com/posts/1473343845-django-asgi-websockets.png

### Questions
1. No, all your code runs synchronously without any sockets or event loops to block. You can use async code within a Django view or channel consumer if you like - for example, to fetch lots of URLs in parallel - but it doesn’t affect the overall deployed site.
2. There are a couple of other limitations - messages must be made of serializable types, and stay under a certain size limit - but these are implementation details you won’t need to worry about until you get to more advanced usage.
3. It’s optional for a backend implementation to understand this - after all, it’s only important at scale, where you want to shard the two types differently — but it’s present nonetheless.
4. `http.request` this is also a channel. what is it. what sorts of messages pass to such channel --- `answer` so it came to notice that like other ws channels, the older http wsgi request are sent to the http.request channel, if left unhandled, this channel layer is routed by default to proper view.

## WHAT IS CHANNEL?
Channel is simply an ordered queue(FIFO). Note: it is not a network dedicated pipe or something, though it sounds like that. So, multiple users can write to the same channel backend, each of which can be uniquely identified.

```python
def my_consumer(message):
  pass
```

```python
channel_routing = {
  'some-channel': 'myapp.consumers.my_consumer'
}
```

And each message passed to the consumer obtained form the channel has **content** attribute (a dict) and  **channel** attribute, which is the channel it came from.

So, with channels, django runs in a worker mode.
https://heroku-blog-files.s3.amazonaws.com/posts/1473343845-django-wsgi.png

**Interface server**: contains both the web server(for WSGI) and the websocket server(for ASGI requests).

**Channel Backend**: http messages and socket messages (combination of python pluggable code or some datastore like - Redis or a shared memory segment responsible from transporting messages)

**Workers**: listen to the channels and run relevant consumer code when message is ready

hence, interface servers changes the outside request to messages on channels. Https request are hadlded like traditional handling using view and template. 

### Good part
notifying clients of changes in real time(chat or *in admin, when other user is simultaneoulsy editing something*)

## Channel Types
1. Dispatch work to consumers (incoming messages has to be routed to the consumers)
2. used for replies... iterface servers are listening to them. Interface server keep track of the reply channels which are uniquely named and also where the client terminates...

django-channels distinguishes between either using special character (`!`) in the response channel's name. while the normal/incoming channel can only have `a-zA-Z0-9_-`

## Groups
Channels can't broadcast. They only deliver to a single channel. And here comes the user of groups. 

```python
def ws_connect(message):
    Group(group_id).add(message.reply_channel)
    # we added a reply_channel to the group for later
    Group(group_id).send({
        'text': json.dumps({
            'username': message.user.username,
            'userid': message.user.id,
            'content': 'has joined the group'
        })
    })
    
    message.reply_channel.send({'accept': True})
```

similarly, we can handle `websocket.disconnect`


But, channels are stateless, channel server has no concept of closing a channel if an interface server goes away - after all, channels are meant to hold messages until a consumer comes along to pickup the message, until it expires.

##### Channels are network-transparent

# Consumers
Default channel `http.request` are routed to the django view, and things will be same as before. 

But still if we want to work with above channel we could use `AsgiRequest` class to translate ASGIRequest into a WSGIRequest, and `AsgiHandler` class to translate HttpResponse into ASGI messages like, `AsgiHandler.encode_response(Httpesponse())` 
And to route to such consumers, we could create a route and add `http.request` as channel and `http_consumer`.

we can see how, `routers.py` and `urls.py` are pretty much similar to each other we use `channel_routing` instead of `urlpatterns` and `routing.route` instead of `urls.url`.

# A Basic application

```python
# consumers.py

def ws_receive(message):
  message.reply_channel.send({
    'text': message.content['text'],
  })
```

```python
# urls.py

from channels.routing import route

channel_routing = [
  route('websocket.receive': 'myapp.consumers.ws_connect')
]
```

And in the front-end 

```javascript
socket = new WebSocket('ws://'+window.location.host+'/test/');

# when we receive a message on the socket
socket.onmessage = function(e) {
  alert(e.data);
}

# when the socket opens
socket.onopen = function() {
  # to send a message
  socket.send("test message")
}

# when socket ready or open
if (socket.readyState = WebSocket.OPEN) socket.onopen()
```

**Note** Channels added to a group expires if their messages expire but disconnect handler will get called nearly allof the time anyway.

**Note** We suggest you design your applications the same way - rather than relying on 100% guaranteed delivery, which *Channels won’t give you*, look at each failure case and program something to expect and handle it - be that retry logic, partial content handling, or just having something not work that one time.

# Running with Channels
We will probably be runningwitho one or more interface servers, and one or more worker servers, connected by some channel layer.

So, Interface servers, each may be servicing various types of requests: websocket, or Http or SMS gateway.

Worker servers where django will run actula logic, while channel layer willbe responsible for the transporting content of channels to the interface server across the network.

Channel layers store all the channel data in a dict in memory and isnt actually cross-processed. It only works inside `runserver`, and this runs the `interface` and `worker` servers in different threads in the same process. So in production, we will *need Redis and asgi_redis* as it works cross-process.

## Channel Layer types

### Redis
using `asgi_redis`

Redis allows to use single redis server with high throughput or multiple redis servers in sharded mode.
```python
CHANNEL_LAYERS = {
  'BACKEND': asgi_redis.ChannelLayer
  ...
}
```

#### Sharding
https://channels.readthedocs.io/en/stable/backends.html#sharding

### IPC
uses POSIX shared memory segments and semaphores in order to allow different processes on the same machine to communicate.

**Note** processes on the "same" machine. So only for single-machine deployments only.

### In-memory
when using `runserver`, where a server thread, this channel layer, and worker thread all co-exist in same python process.

using `runworker`, it will exit, hence not for cross-process communication.

Channels comes with `daphne`, an interface server that can handle both HTTP and WebSockets at the same time, and then ties this into run when you run `runserver`.

*under the hood, runserver is running daphne in one thread and a worker with --autoreload in another*

Either 
1. `runserver`

Or

1. `runserver --noworker`
2. `runworker` optionally `-v 2` to see logs

# Authentication
[auth](https://github.com/django/channels/blob/master/channels/auth.py)

There are few decorators that we can use on consumers.
1. `channel_session`
2. `channel_session_user`
3. `channel_session_user_from_http`
4. `channel_and_http_session_user_from_http`

`channel_session` is responsible for creating a channel_session attribute in the message that persists across consumers with same `reply_channel` attribute.

`channel_session_user` will add attribute message.user from the channel_session, rather than http_session. And this also turns channel_session implicitly.

`channel_session_user_from_http` will call the *`transfer_user`* method which transfers the http session to channel session. And also add attribute *message.user*.

`channel_and_http_session_user_from_http` will do transferring and adding message.user attribute, but along with that rehydrates the http_session.


# Persisting Data
`reply_channel` attribute is a unique pointer to the open ws and varies between clients.
channels is network-transparent and can run in multiple workers, so storing it in global var wont help.

So we use something like django's session framework. And channels provide `channel_session` decorator and we could use `message.channel_session` that acts like normal django session.

```python
message.channel_session['room'] = chat_room_label
# so that we could use it later to see which chat room user belongs to.
```

# Security
we can get `http_session` decorator
  to get `message.http_session` attribute that behaves just like `request.session`. 
`http_session_user` decorator to get `message.user` attribute.

There's also the `channel_session_user` decorator
  which loads user from channel session rather than http session, and a function called `transfer_user`

furthermore, `channel_session_user_from_http` which combines both.

If not running websockets on a separate domain, remember to provide django-session-id as part of the url,

```
socket = new WebScoket('ws://host/?session_key=abcdef')
```

# Routing

---
layout: post
title: Using Aleph as a Clojure WebSocket Client
tags:
 - Software
 - Clojure
---

The (Netty) WebSocket client in 
[ztellman/aleph](https://github.com/ztellman/aleph) is the best one I've used 
in Clojure. 

As one StackOverflow commenter 
[notes](https://stackoverflow.com/a/16451345/802303):
> It can take some time to get used to the asynchronous style and aleph's core 
  abstractions
  
Well, how about wrapping it in something that provides a familiar, 
callback-based interface? 

## WebSocket Client Example

To be clear, this is often worse than just using `aleph`'s API. But, if this is 
what's discouraging use of this library, here it is:

```clojure
(require '[aleph.http :as http])
(require '[manifold.stream :as s])
(require '[manifold.deferred :as d])

(defn ws-conn
  "Open a WebSocket connection with the given handlers.

  All handlers take [sock msg] except for :on-connect, which only takes [sock]
  -- `sock` being the duplex stream.

  - :on-connect   Called once when the connection is established.
  - :on-msg       Called with each message.
  - :on-close     Called once upon socket close with a map {:stat _, :desc _}.

  The optional :aleph parameter is configuration to pass through to Aleph's
  WebSocket client."
  [uri & {:keys [on-connect on-msg on-close aleph]}]
  (let [sock (http/websocket-client uri aleph)
        handle-messages (fn [sock] 
                          (d/chain 
                            (s/consume (fn [msg] (on-msg sock msg)) sock) 
                            (fn [sock-closed] sock)))
        handle-shutdown (fn [sock] 
                          (let [state (:sink (s/description sock))] 
                            (on-close 
                              sock {:stat (:websocket-close-code state) 
                                    :desc (:websocket-close-msg state)})))]
    (d/chain sock #(doto % on-connect) handle-messages handle-shutdown)
    @sock))

(defn ws-send [sock msg] (s/put! sock msg))
(defn ws-close [sock] (s/close! sock))
```

Here's how you'd use it to print out tick data from Bitstamp:

```clojure
(def sub-msg
  (str "{\"event\":\"bts:subscribe\",\"data\":"
       "{\"channel\":\"live_trades_btcusd\"}}"))

(def sock
  (ws-conn "wss://ws.bitstamp.net"
           :on-connect (fn [sock] 
                         (println "Connected.")
                         (ws-send sock sub-msg))
           :on-msg (fn [sock msg] (println ">" msg))
           :on-close (fn [sock msg] (println "Closed:" msg))))

;; After you've seen enough...
(ws-close sock)
```

## Message Size

One word of caution — if you're connecting to a particularly noisy socket that
sends large messages (e.g. [BitMEX](https://bitmex.com/)), you might have to 
tune the frame & frame payload size.

```clojure
(http/websocket-client 
  "wss://www.bitmex.com/realtime" 
  {:max-frame-payload 1e7 :max-frame-size 1e7})
```

## WebSocket Server Example

`aleph` also lets you write a server to test your client (or use for whatever 
else) without too much trouble.

```clojure
(defn uppercase-handler
  "Handle a message by upper casing it and echoing it back."
  [socket msg]
  (s/put! socket (string/upper-case msg)))

(def server
  (http/start-server
    (fn [req] 
      (d/chain 
        (http/websocket-connection req) 
        (fn [socket] 
          (s/consume (fn [msg] (uppercase-handler socket msg)) socket))))
    {:port 9999}))

(def client
  (ws-conn "ws://127.0.0.1:9999"
           :on-connect (fn [sock] (println "Connected."))
           :on-msg (fn [sock msg] (println ">" msg))
           :on-close (fn [sock desc] (println "Closed:" desc))))

(ws-send client "A message")
; > A MESSAGE

(ws-send client "Another message")
; > ANOTHER MESSAGE

(.close server)
; > Closed: {:stat nil, :desc nil}
```

## Advantages 

In my experience, clients based on Netty end up being more durable and reliable
than Jetty based clients. Additionally, `aleph` has plenty of contributors, a 
good history of fixing issues and upgrading the library. It also handles tons 
and tons of other networking functionality under a consistent interface, so if
you're going to add a dependency to your application and learn the paradigms 
that it suggests, it might as well be one that you can continue to use as your
application matures.

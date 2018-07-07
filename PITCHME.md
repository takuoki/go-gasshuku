#### 「Go言語によるWebアプリケーション開発」
#### を読んでみた（１章だけ）

@takuoki

---

### Profile

沖　卓生
@takuoki

ベルトラ株式会社

---

### Requirement

* チャットアプリケーション
* テキスト入力 → 他の人も見れる
* チャットルーム入室時、WebSocketを接続
* WebSocketなので、リアルタイム反映

---

### Architecture

* ブラウザからWebSocketを通して送信されるメッセージを受ける子が必要
* ブラウザから受けたメッセージを皆にブロードキャストしてくれる子が必要
* ブロードキャストされたメッセージをWebSocketを通してブラウザに送信する子が必要

---

### client.go

```go
type client struct {
  socket *websocket.Conn
  send   chan []byte
  room   *room
}

func (c *client) read() {
  for {
    if _, msg, err := c.socket.ReadMessage(); err == nil {
      c.room.forward <- msg
    } else {
      break
    }
  }
  c.socket.Close()
}

func (c *client) write() {
  for msg := range c.send {
    if err := c.socket.WriteMessage(websocket.TextMessage, msg); err != nil {
      break
    }
  }
  c.socket.Close()
}
```

---

### room.go

```go
type room struct {
  forward chan []byte
  join    chan *client
  leave   chan *client
  clients map[*client]bool
  tracer  trace.Tracer
}

func (r *room) run() {
  for {
    select {
    case client := <-r.join:
      r.clients[client] = true
    case client := <-r.leave:
      delete(r.clients, client)
      close(client.send)
    case msg := <-r.forward:
      for client := range r.clients {
        select {
        case client.send <- msg:
        default:
          delete(r.clients, client)
          close(client.send)
        }
      }
    }
  }
}

func (r *room) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  socket, err := upgrader.Upgrade(w, req, nil)
  if err != nil {
    log.Fatal("ServeHTTP:", err)
    return
  }
  client := &client{
    socket: socket,
    send:   make(chan []byte, messageBufferSize),
    room:   r,
  }
  r.join <- client
  defer func() { r.leave <- client }()

  go client.write()
  client.read()
}
```

---

### Impression

* goroutineって、ループの中の処理を切り出すくらいの使い方しかやったことない
* 小さなワーカーを立てて、Channelでやり取りするAppを理解するという意味では、とても良いサンプル
* ただ、第１章からWebSocketのアプリは、ちょっと重いかも

---

### Ending

* `veltra engineering` or `try golang`

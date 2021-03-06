#### 「Go言語によるWebアプリケーション開発」
#### を読んでみた（１章だけ）

@takuoki

---

### Profile

沖　卓生

ベルトラ株式会社

* @takuoki (GitHub)
* @takuokl (Twitter) |

---

### Requirements

* チャットアプリケーション
* テキスト入力 → 他の人も見れる
* チャットルーム入室時、WebSocketを接続
* WebSocketなので、リアルタイム反映

---

### Points

* 入退室するクライアントを管理する
* ブラウザから送信されるメッセージを受ける
* メッセージを皆にブロードキャストする
* メッセージをブラウザに送信する
* これらがgoroutineとして常駐する

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
```

---

### main.go

```go
func main() {
  var addr = flag.String("addr", ":8080", "address of application")
  flag.Parse()

  r := newRoom()
  http.Handle("/", &templateHandler{filename: "chat.html"})
  http.Handle("/room", r)

  go r.run()

  log.Println("Start web server with port:", *addr)
  if err := http.ListenAndServe(*addr, nil); err != nil {
    log.Fatal("ListenAndServe:", err)
  }
}
```

---

### Impressions

* Channelを使うサンプルとしてとても良い
* 第１章からWebSocketはちょっと重いかも
* この先、OAuth認証や画像Uploadなど楽しみ

---

### Ending

[VELTRA Engineering](https://medium.com/veltra-engineering)

[Try Golang!](https://medium.com/veltra-engineering/try-golang/home)

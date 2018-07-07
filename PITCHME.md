#### 「Go言語によるWebアプリケーション開発」
#### を読んでみた（１章だけ）

@takuoki

---

### 自己紹介

沖　卓生
@takuoki

ベルトラ株式会社

---

### 作成するアプリケーション

* チャットアプリケーション
* テキスト入力 → 他の人も見れる
* チャットルーム入室時、WebSocketを接続
* WebSocketなので、リアルタイム反映

---

### アーキテクチャ概要

* ブラウザからWebSocketを通して送信されるメッセージを受ける子が必要
* ブラウザから受けたメッセージを皆にブロードキャストしてくれる子が必要
* ブロードキャストされたメッセージをWebSocketを通してブラウザに送信する子が必要

---

### クライアント

* ブラウザとの通信はWebSocket
* 内部の他のgoroutineとのやり取りはChannel
* デーモンのように動き続ける処理が２つ
  * read: WebSocketの受信
  * write: WebSocketの送信

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

### チャットルーム

* 入室したらクライアントを追加（退室したら削除）
* クライアントからメッセージが送信されてきたら、今いるクライアントにブロードキャスト

---

### 感想

* goroutineって、ループの中の処理を切り出すくらいの使い方しかやったことない
* 小さなワーカーを立てて、Channelでやり取りするAppを理解するという意味では、とても良いサンプル
* ただ、第１章からWebSocketのアプリは、ちょっと重いかも

---

### おわりに

* `veltra engineering` Blog
* `try golang`

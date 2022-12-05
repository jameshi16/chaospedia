---
description: >-
  An attempt at hosting a Minecraft server without portforwarding, and without
  any additional downloads on the client's end.
---

# TCP Hole Punching for Minecraft

## Failure TL;DR

Without first and foremost checking WireShark, I made an assumption that Minecraft uses the same socket to check for the server availability.

## Idea

While UDP hole punching is prevalent to allow gamers to play co-op games, Minecraft hasn't ventured into that space yet. Hence, the most idealistic way to play Minecraft with your friends would be to host a Minecraft server.

A Minecraft Server runs on TCP, and requires portforwarding on the host's part; otherwise, the server is managed by an organization, or via a VPS or cloud. However, privacy-crazed individuals like myself want to run a server without exposing it to the world, potentially from the comfort of a university dorm (i.e. no portforwarding) for free.&#x20;

One way to do such a thing is to host a proxy server via Oracle Cloud, but Oracle Cloud is _limited_ by region and bandwidth.

Another way to do it is to get your friends to download things like Hamachi or ZeroTier; but I don't want my friends to download software that they're uncomfortable with. Furthermore, it is a hassle to onboard new users.

TCP hole punching comes in handy in situations like these. There is a document describing how TCP hole punching can be done on Bryan Ford's home page: [https://bford.info/pub/net/p2pnat/](https://bford.info/pub/net/p2pnat/)

## Implementation

These are not tested outside of a localhost machine; I only realized that it was too late when I inspected how Minecraft checked for server availability via WireShark.

I was already had a proxy server running [Fast Reverse Proxy](https://github.com/fatedier/frp) on Oracle Cloud; all I needed to do was to make that server a rendezvous server, which can be done like so:

{% tabs %}
{% tab title="frps-attachment.go" %}
```go
package main

import (
  "html"
  "log"

  "encoding/json"
  "net/http"
)

// TODO: just make this constant, there's no point
var allowResponse []byte

type allowResponseShape struct {
  reject bool
  unchange bool
}

func createAllowResponse() ([]byte, error) {
  res := allowResponseShape {
    reject: false,
    unchange: true,
  }

  data, err := json.Marshal(res)
  return data, err
}

func init() {
  var err error
  allowResponse, err = createAllowResponse()

  if err != nil {
    log.Fatal("Can't create response.", err)
  }
}

func RunAttachment() {
  http.HandleFunc("/attachment", func(w http.ResponseWriter, r *http.Request) {
    log.Printf("%q", html.EscapeString(r.URL.Path))
    w.Write(allowResponse)
    w.WriteHeader(200)
  })

  log.Fatal(http.ListenAndServe("127.0.0.1:8080", nil))
}
```
{% endtab %}

{% tab title="main.go" %}
```go
package main

import (
  "fmt"
  "log"
  "net"
  "sync"
)

func main() {
  managers := make([]net.Conn, 0)
  wg := new(sync.WaitGroup)
  wg.Add(2)

  go func() {
    defer wg.Done()

    RunAttachment()
  }()

  go func() {
    defer wg.Done()

    listener, err := net.Listen("tcp", "127.0.0.1:8082")
    if err != nil {
      log.Fatal("Cannot create TCP socket", err)
    }

    for {
      conn, err := listener.Accept()
      if err != nil {
        log.Println("[W] Cannot accept connection", err)
        continue
      }

      wg.Add(1)
      go func() {
        defer wg.Done()

        for _, manager := range managers {
          manager.Write([]byte(fmt.Sprintf("New (%s,%s)", conn.LocalAddr(), conn.RemoteAddr())))
          log.Println("Sent ", manager.RemoteAddr(), " remote address", conn.RemoteAddr())
        }
      }()
    }
  }()

  go func() {
    defer wg.Done()

    listener, err := net.Listen("tcp", "127.0.0.1:8083")
    if err != nil {
      log.Fatal("Cannot create TCP socket", err)
    }

    for {
      conn, err := listener.Accept()
      if err != nil {
        log.Println("[W] Cannot accept manager connection", err)
      } else {
        managers = append(managers, conn)
      }
    }
  }()

  wg.Wait()
}
```
{% endtab %}
{% endtabs %}

This is meant to be run as an FRP plugin running on port `8080`.The idea is to let the proxy / rendezvous server continue to route connections to the Minecraft server even if TCP hole punching failed. Upon being contacted through port `8083`, the server connects the connecting client as a manager. A manager receives IP addresses that tries to connect to this rendezvous server via a target port, which is `8082` in this case.

On the host with the minecraft server, the following code is run:

```go
package main

import (
  "bufio"
  "fmt"
  "io"
  "log"
  "net"
  "strings"
  "sync"

  reuse "github.com/libp2p/go-reuseport"
)

func getTargetClientFromPacket(packet string) (string, error) {
  ips := strings.Split(packet[1:len(packet) - 2], ",")
  if len(ips) == 1 {
    return "", fmt.Errorf("Can't get target Client")
  }

  return ips[1], nil
}

func main() {
  attempting_clients := make([]string, 0)
  attempting_mutex := new(sync.RWMutex)
  established_clients := make([]string, 0)
  established_locker := new(sync.Mutex)
  established_updated := sync.NewCond(established_locker)
  // Stage 1: Connect to manager
  conn, err := net.Dial("tcp", "127.0.0.1:8083")
  if err != nil {
    log.Fatalln("Cannot connect to manager.", err)
  }

  // Stage 2: Have 1 goroutine listen to manager, 1 to
  // attempt connections, 1 to proxy connections.

  wg := new(sync.WaitGroup)
  wg.Add(3)

  go func() {
    defer wg.Done()

    for {
      reader := bufio.NewReader(conn)
      data, err := reader.ReadString(byte(')'))
      if err != nil && err != io.EOF {
        log.Fatalln("Weird packet received from manager")
      }

      log.Println("Packet: ", data)
      target, err := getTargetClientFromPacket(data)
      if err != nil {
        log.Fatalln("Cannot get target client.", err)
      }
      log.Println("New attempt: ", target)
      attempting_mutex.Lock()
      attempting_clients = append(attempting_clients, target)
      attempting_mutex.Unlock()
    }
  }()

  listener, _ := reuse.Listen("tcp", "127.0.0.1:25565")
  connector, _ := net.Dial("tcp", "127.0.0.1:25566")

  go func() {
    defer wg.Done()

    for {
      attempting_mutex.RLock()
      for i, target := range attempting_clients {
        _, err := reuse.Dial("tcp", "127.0.0.1:25565", target)
        if err != nil {
          log.Println("Failed to connect to ", target, "; try again later.")
          continue
        }

        log.Println("Successful connection to ", target)
        established_clients = append(established_clients, target)
        attempting_mutex.RUnlock()
        attempting_mutex.Lock()
        attempting_clients[len(attempting_clients) - 1], attempting_clients[i] = attempting_clients[i], attempting_clients[len(attempting_clients) - 1]
        attempting_clients = attempting_clients[:len(attempting_clients) - 1]
        attempting_mutex.Unlock()
        established_updated.Broadcast()
        break
      }
      attempting_mutex.RUnlock()
    }
  }()

  go func() {
    defer wg.Done()

    for {
      conn, err := listener.Accept()
      if err != nil {
        continue
      }

      wg.Add(1)
      go func() {
        defer wg.Done()

        for {
          _, err := bufio.NewReader(conn).WriteTo(connector)
          if err != nil && err == io.EOF {
            break
          }

          if err != nil {
            log.Fatalln("Error while reading from TCP stream ", err)
          }
        }
      }()
    }
  }()

  wg.Wait()
}

```

This uses the `go-reuseport` library, which is an essential package to make TCP hole punching work. The code essentially opens port `25565` as both a listener and dialer, which is essential for a successful hole punch. Port `25566` is the actual Minecraft server, while port `8083` is the manager port from the earlier code block for the rendezvous server.

Theoratically, when a Minecraft client tries to connect to the rendevous server **and** the Minecraft server **with the **_**same**_** IP address & port**, the rendevous server will send the Minecraft client's connection information to the Minecraft server (behind a NAT), and the Minecraft server will desperately attempt to connect to the Minecraft client.

Since the Minecraft client has _tried_ contacting the Minecraft server before, the tuple entry `(Minecraft Server IP, Minecraft Server Port, Minecraft Client IP, Minecraft Client Port)` shuold be on the client's NAT, which should allow the Minecraft server to connect to the client via a reverse connection. Even if this fails, the action of an attempted connection will add the tuple entry `(Minecraft Client IP, Minecraft Client Port, Minecraft Server IP, Minecraft Server Port)` into the server's NAT.

With both NATs possessing the correct entries, there should be a successful connection - _at least in theory_.

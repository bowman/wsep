# wsep

`wsep` is a high performance, WebSocket-based command execution protocol. It can be thought of as SSH without
encryption.

It's useful in cases where you want to provide a command exec interface into a remote environment. It's implemented
with WebSocket so it may be used directly by a browser frontend.

## Examples

### Client

```go
	conn, _, err := websocket.Dial(ctx, "ws://remote.exec.addr", nil)
	if err != nil {
    // handle error
  }
	defer conn.Close(websocket.StatusAbnormalClosure, "terminate process")

	executor := wsep.RemoteExecer(conn)
	process, err := executor.Start(ctx, proto.Command{
    Command: "cat",
    Args: []string{"go.mod"},
	})
	if err != nil {
    // handle error
  }

	go io.Copy(os.Stdout, process.Stdout())
	go io.Copy(os.Stderr, process.Stderr())

	err = process.Wait()
	if err != nil {
    // process failed
    return
  }
  // process succeeded
  conn.Close(websocket.StatusNormalClosure, "normal closure")
```

### Server

```go
func (s Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ws, err := websocket.Accept(w, r, &websocket.AcceptOptions{InsecureSkipVerify: true})
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
  }
  defer ws.Close()

	err = wsep.Serve(r.Context(), ws, wsep.LocalExecer{})
	if err != nil {
    // handle error
	}
}
```

### Performance Goals

Test command

```shell script
head -c 100000000 /dev/urandom > /tmp/random; cat /tmp/random | pv | time coder sh ammar sh -c "cat > /dev/null"
```

streams at

```shell script
95.4MiB 0:00:08 [10.7MiB/s] [                                <=>                                                                                                                                                  ]
       15.34 real         2.16 user         1.54 sys
```

The same command over wush stdout (which is high performance) streams at 17MB/s. This leads me to believe
there's a large gap we can close.

## Protocol

Each Message is represented as a single WebSocket message. A newline character separates a JSON header from the binary body.

Some messages may omit the body.

The overhead of the additional frame is 2 to 6 bytes. In high throughput cases, messages contain ~32KB of data,
so this overhead is negligible.

### Client Messages

#### Start

This must be the first Client message.

```json
{
  "type": "start",
  "command": {
    "command": "cat",
    "args": ["/dev/urandom"],
    "tty": false
  }
}
```

#### Stdin

```json
{ "type": "stdin" }
```

and a body follows after a newline character.

#### Resize

```json
{ "type": "resize", "cols": 80, "rows": 80 }
```

Only valid on tty messages.

#### CloseStdin

No more Stdin messages may be sent after this.

```json
{ "type": "close_stdin" }
```

### Server Messages

#### Pid

This is sent immediately after the command starts.

```json
{ "type": "pid", "pid": 0 }
```

#### Stdout

```json
{ "type": "stdout" }
```

and a body follows after a newline character.

#### Stderr

```json
{ "type": "stderr" }
```

and a body follows after a newline character.

#### ExitCode

This is the last message sent by the server.

```json
{ "type": "exit_code", "code": 255 }
```

A normal closure follows.

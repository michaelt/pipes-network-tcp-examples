pipes-network-tcp-examples
==========================

These mechanically follow the pleasantly
transparent 'hello world'-ish examples in
http://www.yesodweb.com/blog/2014/03/network-conduit-async
which mix networking ABCs with elementary
concurrency and conduitry (here, pipe-istry).

Apart from `pipes-network` and `async` we need the pleasant
`word8` library. They use `lens-family` for the lens operations.

The pipes variants follow Michael S's text in this
order:

-   `Examples/ServerToUpper.hs`
    -   a server on 4000 that sends back input sent e.g. with telnet
        upper-cased or 'angry'
-   `Examples/ServerDoubler.hs`
    -   a server on 4001 that sends back 
        input doubled, `Char8` by `Char8`
-   `Examples/ClientToUpper.hs`
    -   a client through which the user interacts
        directly to the "angry" server 
-   `Examples/ClientPipeline.hs`
    -   a client that sends material to the
        "angry" server and the doubling server and
        returns it to the user
-   `Examples/ProxyToUpper.hs`
    -   a proxy on 4002 that sends input to the
        angry server on 4000
-   `Examples/ProxyAuth.hs`
    -   a proxy on 4003 that asks for demands
        authorization before condescending to send
        user input to the angry server on 4000

The following remarks will require that eight
instances of a terminal all be opened in the main
directory of the repository; if you like you can
`cabal install` from the main directory, and a
crude option parser will make the examples usable with
one executable:
```bash
$ pipes-network-tcp-examples --help
Usage: pipes-network-tcp-examples COMMAND

Available options:
  -h,--help                Show this help text

Available commands:
  ClientPipeline
  ClientToUpper
  ProxyAuth
  ProxyToUpper
  ServePipes
  ServerDouble
  ServerToUpper
```

Since most examples use the uppercasing service,
which looks like this:

```haskell
main :: IO ()
main = do putStrLn "Opening upper-casing service on 4000"
          serve (Host "127.0.0.1") "4000" $ \(client,_) -> 
               runEffect $ fromSocket client 4096
                           >-> Bytes.map toUpper
                           >-> toSocket client
```

we start it in one terminal:
```bash
term1$ runhaskell Examples/ServerToUpper.hs
Opening upper-casing service on 4000
```
or:
```bash
term1$ pipes-network-tcp-examples ServerToUpper
Opening upper-casing service on 4000
```
then in another terminal we can write
```bash
term2$ telnet localhost 4000
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello -- <-- our input
HELLO
...
```
or we can scrap telnet and use a dedicated Haskell client, which reads like this:
```haskell
main = connect "127.0.0.1" "4000" $ \(connectionSocket,_) -> do
  let act1 = runEffect $ PB.stdin >-> toSocket connectionSocket
      act2 = runEffect $ fromSocket connectionSocket 4096 >-> PB.stdout
  concurrently act1 act2 
  return ()
```
thus:
```bash
term3$ runhaskell Examples/ClientToUpper.hs  # or pipes-network-tcp-examples ClientToUpper
el pueblo unido jamas sera vencido!  -- our input
EL PUEBLO UNIDO JAMAS SERA VENCIDO!
el pueblo unido jamas sera vencido!
EL PUEBLO UNIDO JAMAS SERA VENCIDO!
...
```

In a flurry of terminal-openings we can also start
up the doubling service

```bash
term4$ runhaskell Examples/ServerDouble.hs # or pipes-network-tcp-examples ServerDouble
```
then elsewhere
```bash
term5$ telnet localhost 4001
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hheelllloo
```
But let's try the Haskell client that interacts with 4000 and 4001 together,
i.e.:
```haskell
main = connect "127.0.0.1" "4000" $ \(socket1,_) ->
       connect "127.0.0.1" "4001" $ \(socket2,_) ->
        do let act1 = runEffect $ PB.stdin >-> toSocket socket1
               act2 = runEffect $ fromSocket socket1 4096 >-> toSocket socket2
               act3 = runEffect $ fromSocket socket2 4096 >-> PB.stdout
           runConcurrently $ Concurrently act1 *>
                             Concurrently act2 *>
                             Concurrently act3
```
(note the use of the `Applicative` instance for `Concurrently` from the
`async` library), thus:
```bash
term6$ runhaskell Examples/ClientPipeline.hs   # or pipes-network-tcp-examples ClientPipeline
hello
HHEELLLLOO
```

Don't tell the children they can access the
'angry' server directly on localhost 4000; we will
demand authorization on 4003

`term7$ runhaskell Examples/ProxyAuth.hs`

or

`term7$ pipes-network-tcp-examples ProxyAuth`

which then elsewhere permits
```bash
term8$ telnet localhost 4003
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Username: spaceballs
Password: 12345
Successfully authenticated.
joy to the world!
JOY TO THE WORLD!
```

## Minimal Session Types: Equality Server in Cloud Haskell
This note briefly illustrates:
- a high-level program written in [Cloud Haskell](http://haskell-distributed.github.io) and annotated with _standard_ session types 
- its corresponding decomposition into trios processes with _minimal_ session types.

Following the session type that is described in the introduction of our submission, we consider an equality service `eqServer` that receives two values of type `Int` and sends back a value of type `Bool`. The corresponding client process is called `eqClient`. 

For the sake of illustration, below we use a _pseudo_ Cloud Haskell notation: we neglect details of the `spawn` function, namely dealing with function closures and network locations. Also, we explicitly annotate channels with their intended session type.

### The High-level Program (with Standard Session Types)
We would like programmers to write Cloud Haskell programs whose channels come annotated with *standard session types*:
```haskell
master :: Process ()
master = do
  -- spawning server and client:
  (s,r) <- newChan :: (Chan (?Int;?Int;!Bool;End), Chan (!Int;!Int;?Bool;End))
  spawn $ eqServer s
  spawn $ eqClient r

-- server definition:
eqServer :: Chan (?Int;?Int;!Bool;End) -> Process ()
eqServer s = do
    a <- receiveChan s
    b <- receiveChan s
    sendChan s (a == b)

-- client definition:
eqClient :: Chan (!Int;!Int;?Bool;End)-> Process ()
eqClient  r = do
    sendChan r 4
    sendChan r 2
    b <- receiveChan r
```
We remark that this example is _not_ a Cloud Haskell program, since it uses session-typed channels. 
Our goal is to exploit our decomposition into trios processes to generate a valid Cloud Haskell program. 

### The Decomposed Program (with Minimal Session types)

The interaction between the server and the client involves a session protocol of lenght three: first two integers are passed around, then a boolean is exchanged. This means that we will have three leading trios on each side. 

Following the process decompositions defined in our submission, we obtain the following program: 

```haskell
masterB :: Process ()
masterB = do

  -- propagator channels for the breakdown:
  (c1',c1) <- newChan :: (SendPort (), ReceivePort ())
  (c2',c2) <- newChan :: (SendPort Int, ReceivePort Int)
  (c3',c3) <- newChan :: (SendPort (Int,Int), ReceivePort (Int,Int))
  (c4',c4) <- newChan :: (SendPort (), ReceivePort ())

  (c5',c5) <- newChan :: (SendPort (), ReceivePort ())
  (c6',c6) <- newChan :: (SendPort (), ReceivePort ())
  (c7',c7) <- newChan :: (SendPort (), ReceivePort ())
  (c8',c8) <- newChan :: (SendPort (), ReceivePort ())

  -- decomposition of channels:
  (r1,s1) <- newChan :: (SendPort Int, ReceivePort Int)
  (r2,s2) <- newChan :: (SendPort Int, ReceivePort Int)
  (s3,r3) <- newChan :: (SendPort Bool, ReceivePort Bool)

  -- spawning the three trios for the server:
  spawn $ eqServerB1 c2 s1 c3
  spawn $ eqServerB2 c3 s2 c4'
  spawn $ eqServerB3 c4 s3 c5'
  spawn $ eqServerB4 c5

  -- spawning the three trios for the client:
  spawn $ eqClientB1 c6 r1 c7'
  spawn $ eqClientB2 c7 r2 c8'
  spawn $ eqClientB3 c8 r3 c9'
  spawn $ eqClientB4 c9

  -- activate processes in parallel, coresponding
  -- to the breakdown of a parallel composition
  sendChan c1 ()
  sencChan c5 ()


-- breakdown of eqServer:
eqServerB1 :: ReceivePort () -> RecvPort Int -> SendPort Int -> Proc ()
eqServerB1 c1 s1 c2' = do
   _ <- receiveChan c1
   a <- receiveChan u1
   sendChan c3' a

eqServerB2 :: ReceivePort Int -> RecvPort Int -> SendPort (Int,Int) -> Proc ()
eqServerB2 c2 s2 c3' = do
    a <- receiveChan c2
    b <- receiveChan u1
    sendChan c3' (a,b)

eqServerB3 :: ReceivePort (Int, Int) -> SendPort Bool -> Proc ()
eqServerB3 c3 s3 c4' = do
   (a,b) <- receiveChan c3
   sendChan (a==b)

eqServerB4 :: ReceivePort () -> Proc ()
eqServerB4 c4 =
  do
    _ <- receiveChan c4

-- breakdown of eqClient:
eqClientB1 :: ReceivePort () -> SendPort Int -> SendPort () -> Proc ()
eqClientB1 c5 r1 c6' = do
   _ <- receiveChan c6
   sendChan r1 4
   sendChan c7' ()

eqClientB2 :: ReceivePort () -> SendPort Int -> SendPort () -> Proc ()
eqClientB2 c6 r2 c7' = do
    _ <- receiveChan c7
    sendChan r2 2
    sendChan c8' ()

eqClientB3 :: ReceivePort () -> ReceivePort Bool -> Proc ()
eqClientB3 c7 r3 c8' = do
   _ <- receiveChan c7
   check <- receiveChan r3
   sendChan c8' ()

eqClientB4 :: ReceivePort () -> Proc ()
eqClientB4 c8 = do
    _ <- receiveChan c8
```

Notice that the resulting decomposition is a valid Cloud Haskell program! 

It only uses Cloud Haskell typed-channels, i.e., channels that can carry a value of a single type (in this case,  `Int` or `Bool`). 

#### Remark
Our decomposition breaks a process down into a series of trios and decomposed channels. In this (simple) example, the master process has a role of parallel composition. 

  We may notice that `masterB` is not exactly a decomposition of parallel composition as it does not have a guard propagator for an activation  (denoted _c_k_ in the submission), which would be redundant here. 
  
  Therefore, according to the breakdown of parallel composition, `masterB` spawns trios of broken down subprocesses and activates them with `sendChan c1 ()` and `sendChan c5 ()`. 



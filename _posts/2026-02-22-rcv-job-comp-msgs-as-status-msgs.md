## Receiving Job Completion Messages as Status Messages

When you are developing on IBM i you can do your compiles interactively - at the end, the compiler sends a completion message, whether the compile was successful or not. This is nice, but the compiler blocks your session during the compilation.

You can opt to make your compiles in batch - so your session is free to do other work. And when the compiled finnishes, it will send you a message whether the compiler job finished successfully or not.

Now you can set your message queue to `*BREAK` - which means, that your session will get interupted and the message appears on the screen. Nice, you know in real time that the compiler weas successful - but you were interrupted. OK set your message queue to `*NOTIFY` - you will see a small `MW` (for Message Waiting) at the bottom of your session, when the compiler was finished - but you can't see, if the job completed successful or not.

Wouldn't it be nice, to see the result - without being interrupted?

I don't remember, where and when I have seen this "trick" first - but is was somewhere in the early to mid 1990s, and I use it since then. And when I searched for it lately, I hard a hard time, finding the tip somewhere online, so I thought, it would be good to have it in my blog.

I haven't invented this idea - and sadly, I also don't know who exactly has - so I have to give credit to the *unknown programmer* who saved me a lot of time in the last 30 years - Thank You!

But now dive in ...

### Creating a Break Message Handling Program

First, we have to create a small CLP / CLLE program that will handle the messages for us. 

```cl
PGM (&MSGQ &MSGQLIB &MSGMRK)
DCL &MSGQ TYPE(*CHAR) LEN(10)
DCL &MSGQLIB TYPE(*CHAR) LEN(10)
DCL &MSGMRK TYPE(*CHAR) LEN(4)

DCL &MSGID TYPE(*CHAR) LEN(7)
DCL &MSGDTA TYPE(*CHAR) LEN(1000)
DCL &MSGDTALEN TYPE(*DEC) LEN(5 0)
DCL &MSGF TYPE(*CHAR) LEN(10)
DCL &MSGFLIB TYPE(*CHAR) LEN(10)

RCVMSG MSGQ(&MSGQLIB/&MSGQ) MSGKEY(&MSGMRK) +
       RMV(*NO) MSGDTA(&MSGDTA) +
       MSGDTALEN(&MSGDTALEN) MSGID(&MSGID) +
       MSGF(&MSGF) MSGFLIB(&MSGFLIB)

/* handle normal and abnormal job completion messages */
IF (&MSGID *EQ 'CPF1241' *OR &MSGID *EQ 'CPF1240') DO
  SNDPGMMSG MSGID(&MSGID) MSGF(&MSGFLIB/&MSGF) +
            MSGDTA(&MSGDTA) TOPGMQ(*EXT) +
            TOMSGQ(*TOPGMQ) MSGTYPE(*STATUS)
  GOTO FINITO
ENDDO

/* all other messages require user intervention */
DSPMSG MSGQ(&MSGQLIB/&MSGQ)

FINITO: ENDPGM 
```

The parameters for a *break message handler* program are defined by the system. The is pretty straight forward. First, the program receives the message with the correct *marker* from the message queue.

If the message is a `CPF1241` (Normal End) or `CPF1240` (Abnormal End) it sends the message out again as a status message - this will show up in the message line of your interactive session.

If it's any other message, the program will simply to a DSPMSG of the Message Queue - which means, the message will *break* through.

Compile this small program to your personal library or anywhere you want it. For the rest of this post, we will assume you compiled it as `YOURLIB/BRKMSGHDL`.

### Attaching the Break Message Handler to your Message Queue

Next, we need to attach our *break message handler* program to our message queue. The best is, to do that when you sign-in - in the *initializing program* (`INLPGM`) of your user profile.

Check with the command `CHGPRF` if you already have an `INLPGM`. If this is the case, you can add the commands to it - otherwise create a new one.

```cl
PGM

DCL &MSGQ TYPE(*CHAR) LEN(10)
DCL &MSGQLIB TYPE(*CHAR) LEN(10)

/* get current users message queue */
RTVUSRPRF USRPRF(*CURRENT) +
          MSGQ(&MSGQ) MSGQLIB(&MSGQLIB)
MONMSG  MSGID(CPF0000)

/* attach the break message handler to the msgq */
CHGMSGQ MSGQ(&MSGQLIB/&MSGQ) DLVRY(*BREAK) +
        PGM(YOURLIB/BRKMSGHDL)
MONMSG  MSGID(CPF0000)

ENDPGM
```

First, the program retrieves the message queue of the current user.

Next, it tries to change the message queue to `*BREAK` and attaches the handler. This is not a permanent change, because it lasts only as long, as that session is logged in. 

And it only works **one** time - only the first logged in session can attach the *break message handler* program. This is the reason why a `MONMSG CPF0000` after that command is prefenting any error messages.

It's also a good idea to have a `MONMSG` after nearly every command in your `INLPGM` - otherwise the program could fail, you might be unable to log in. 

Compile this program to your personal library or where you want it. Let's assume you compiled it as `YOURLIB/STARTMEUP`.

### Changing your User Profile

The last step is to attach the *initializing program* to your user profile.

```clle
CHGPRF INLPGM(YOURLIB/STARTMEUP)
```

The command `CHGPRF` can be prompted with `F4` if you like - the `INLPGM` is something like the `.profile` in typical **nix* operating systems. It is executed when a user logs in.

### Finally

If anything works correctly, you can sign-off and sign-in again. Try the command `SBMJOB CMD(DLYJOB DLY(5)) JOB(TESTMSG)` - after 5 seconds (or so) you should see a message `Job ... completed normally` in the status/message line of you session.

The completion messages stay in your message quere - so if you ever miss one, you can always look with `DSPMSG`.

As long as I can remember, I used this *trick* - and I hope it will help you too.

If you have questions or ideas - I have opened the 
[Discussions](https://github.com/qpgmr-de/qpgmr-de.github.io/discussions)
page of this repository - so, if you like, leave a comment.
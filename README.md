# NAME

Milter::Client - Send commands to milters and get responses

# VERSION

version 0.001

# SYNOPSIS

```perl
use Milter::Client qw{:constants};

# A message as an MTA would hand it to the milter, from the connection to the end of the message
my $sock = Milter::Client::milter_socket('unix:/run/opendmarc/opendmarc.sock');
my @cmds = Milter::Client::conversation(
    "From: test\@test.test\nSubject: Test\n\nTesting 123\n",
    { client_ip => '192.0.2.1', helo => 'test.test', mail_from => 'test@test.test', rcpt_to => ['test@test.test'] },
);
my ( $code, $payload, $mods ) = Milter::Client::sendmail( $sock, { timeout => 5 }, @cmds );
print Milter::Client::decode( @$_ )->{action}, "\n" for @$mods;

# Or build the conversation by hand.  This is a pretty maximal exchange, mirroring a standard SMTP dialog.
my $sockfile = '/var/run/feet.sock';
our @gibbering = (
    # Version, actions, protocol steps.  Below is every action and step of version 6.  You can omit this entirely.
    [SMFIC_OPTNEG,  6, 0x1FF, 0x1FFFFF],
    # You'll want to use the proper sock type based on how the client connected to the MTA
    # If it isn't a unix socket, the last argument needs to be an IP address
    [SMFIC_CONNECT, 'test.test', SMFIA_UNIX, 0, $sockfile],
    # Everything from here is pretty self-explanatory based on its SMTP equivalents.
    [SMFIC_HELO,    'test.test'],
    [SMFIC_MAIL,    '<test@test.test>'], # Envelope Sender
    [SMFIC_RCPT,    '<test@test.test>'], # Envelope Recipient
    [SMFIC_DATA,    ],
    # One command per header: name, then value.
    [SMFIC_HEADER,  'From',    'test@test.test'],
    [SMFIC_HEADER,  'To',      'test@test.test'],
    [SMFIC_HEADER,  'Subject', 'Test'],
    [SMFIC_EOH,     ],
    # Bodies over 64k need to be sent in chunks, see body_chunks()
    [SMFIC_BODY,    "Testing 123"],
    [SMFIC_BODYEOB, ],
    [SMFIC_QUIT,    ],
);

# Returns whenever you either run out of commands or get a code other than SMFIS_CONTINUE or SMFIS_OPTNEG
# You'll get some kind of SMFIR_* constant returned, usually SMFIR_REPLYCODE when it's a REJ/DEFER w/ SMTP & ESMTP response codes.
( $code, $payload ) = Milter::Client::sendmail( Milter::Client::milter_socket($sockfile), @gibbering );

# Pass options as a hashref before the commands to find out when the milter hangs or dies,
# rather than presuming it wanted you to continue.
( $code, $payload ) = Milter::Client::sendmail( Milter::Client::milter_socket($sockfile), { timeout => 5 }, @gibbering );
warn "Milter hung"  if $code eq CLIENT_TIMEOUT;
warn "Milter died"  if $code eq CLIENT_EOF;
```

# DESCRIPTION

The milter protocol is what an MTA such as postfix or sendmail speaks to a mail filter.
This module speaks the MTA's half of it, so a milter can be tested without an MTA.

`milterprobe` is a program on top of it, which gives one message to a milter and prints what the milter did with it.

# WHY

It occurred to me the lack of public development for innovative milters may have something to do with the difficulty of testing them.

There are no milter client modules which are not themselves an MTA I am aware of.
As you might imagine, that complicates the sort of automated testing you might want to do to distribute modular milters.

# FUNCTIONS

## ($code, $payload, $modifications) = sendmail($socket, \[\\%options\], @commands)

Send commands to a milter & read the responses.

Terminates whenever you run out of commands or get something other than SMFIR\_CONTINUE.

Requests to modify the message (add a header, change the body, quarantine it and so forth) are not a final reply.
They are returned in `$modifications`, an arrayref of `[ $code, $payload ]`, which `decode()` reads.

Headers are sent one per command, as `[SMFIC_HEADER, $name, $value]`.
The older form of `[SMFIC_HEADER, $all_the_headers]` still works, but the milter will see it as one header with an empty value.

Macros go as `[SMFIC_MACRO, $command, $name =` $value, ...\]>, sent before the command they belong to,
for example `[SMFIC_MACRO, SMFIC_MAIL, i =` 'QUEUEID'\]> before SMFIC\_MAIL for the queue id.

No reply is waited for after SMFIC\_MACRO or SMFIC\_QUIT, as milters do not send one.

The milter's answer to SMFIC\_OPTNEG is obeyed as an MTA obeys it.
A command for a step the milter asked to leave out is not sent, and neither is a macro for it.
No reply is waited for to a step the milter said it sends no reply to.
Without SMFIC\_OPTNEG, every command is sent and every reply waited for.

Without options, a milter which says nothing within a second is presumed to want you to continue.
Pass a hashref of options before the commands to change that:

- `timeout`

    Seconds (fractions allowed) to wait for each reply.
    When given, a milter which does not reply in time returns `CLIENT_TIMEOUT`, and one which hangs up returns `CLIENT_EOF`.

See Synopsis for more details.

## $socket = milter\_socket($spec, \[$timeout\])

Connect to a milter, and return the socket for `sendmail()`.

`$spec` is written as postfix and sendmail write it:
`unix:/path` (or `local:/path`, or just `/path`) for a unix socket,
and `inet:port@host` or `inet6:port@host` for TCP.
The host defaults to localhost.

`$timeout`, in seconds, applies to the connect.

Dies when it cannot connect, or when `$spec` is none of those.

## @commands = conversation($raw, \[\\%envelope\])

The commands an MTA would send a milter for one message on a connection of its own, for `sendmail()`:
OPTNEG offering every version 6 action and step, CONNECT, HELO, MAIL, RCPT, DATA,
one HEADER per header, EOH, the body in chunks with bare LF turned into CRLF as on the wire, end of message, and QUIT.

`$raw` is the message, header and body, as `split_message()` takes it.
`%envelope` says what the MTA knew about the client and the envelope:

- `client_ip`

    The address of the client.  Without it, CONNECT says the family is unknown.

- `client_host`

    The name of the client.  Defaults to the address in brackets, or `localhost`.

- `helo`

    What the client said in HELO.  Defaults to `unknown`.

- `mail_from`

    The envelope sender, without angle brackets.  Defaults to the null sender.

- `rcpt_to`

    The envelope recipient, or an arrayref of them, one RCPT each.  Defaults to one null recipient.

- `queue_id`

    Sent as the `{i}` macro before MAIL, as postfix does.

- `macros`

    A hashref of the stage (`connect`, `helo`, `mail`, `rcpt`, `data`, `eoh` or `eom`) to a hashref of macro names and values, sent before that stage.

Dies on a stage it does not know.

## ($fields, $body) = split\_message($raw)

Split a raw message the way an MTA would before handing it to a milter.

`$fields` is an arrayref of `[ $name, $value ]` in the order they appear.
The value has the single space after the colon removed, and keeps any folding (with CR removed) as a milter would see it.
Lines in the header which are not a header field are dropped.

`$body` is everything after the blank line which ends the header.

## @commands = body\_chunks($body)

Split a message body into SMFIC\_BODY commands no bigger than MILTER\_CHUNK\_SIZE, as an MTA would.

An empty body yields no commands.

## \\%what = decode($code, $payload)

What a reply from `sendmail()` means, as a hashref with an `action` and the fields that action has.
It takes a modification, as `@$_` of each in `$modifications`, or the final code and payload.

- `add_header`: `name`, `value`
- `insert_header`, `change_header`: `index`, `name`, `value`
- `delete_header`: `index`, `name`

    A change to an empty value, which is how a milter removes a header.

- `add_rcpt`: `rcpt`, and `args` when the milter gave ESMTP arguments
- `delete_rcpt`: `rcpt`
- `change_from`: `from`, and `args` when the milter gave ESMTP arguments
- `replace_body`: `body`, one chunk of the new body
- `quarantine`: `reason`
- `reply`: `reply`, the SMTP reply such as `550 5.7.1 Go away`
- `accept`, `continue`, `discard`, `reject`, `tempfail`, `timeout`, `eof`

`index` counts from 1 among the headers of that name for a change, and among all headers for an insert.
Header names and values are as the milter sent them, folding and all.

Dies on a code it does not know.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# CONTRIBUTOR

Andy Baugh <andy@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

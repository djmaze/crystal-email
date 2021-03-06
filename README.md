# EMail for Crystal

Simple email sending library for the [Crystal programming language](https://crystal-lang.org).

You can:

- construct an email with a plain text message, a HTML message and/or some attachment files.
- include resources(e.g. images) used in the HTML message.
- set multiple recipients to the email.
- use multibyte characters(only UTF-8) in the email.
- send the email by using local or remote SMTP server.
- use TLS connection by `STARTTLS` command.
- use SMTP-AUTH by `AUTH PLAIN` or `AUTH LOGIN` when using TLS.
- send multiple emails concurrently by using multiple smtp connections.

You can not:

- use ESMTP features except those mentioned above.

## Installation

Add this to your application's `shard.yml`:

```yaml
dependencies:
  email:
    github: arcage/crystal-email
```

## Usage

```crystal
# send simple 1 message

require "email"

EMail.send("your.mx.server.name", 25) do
  # [*] : useable multiple times

  # required
  subject  "Subject of the mail"
  from     "your@mail.addr" # [*]
  to       "to@some.domain" # [*]

  # optional
  cc            "cc@some.domain"     # [*]
  bcc           "bcc@some.domain"    # [*]
  reply_to      "reply_to@your.mail" # [*]
  sender        "sender@your.mail"
  return_path   "return@your.mail"
  envelope_from "enverope_from@your.mail"

  # required at least one `message`, `message_html` or `attach`
  message  <<-EOM
    Message body of the mail.

    --
    Your Signature
    EOM

  attach "./attachment.docx"                            # [*]

  # HTML email support
  # `message_resource` works almost same as `attach`, expect it requires `cid:` argument.
  message_html <<-EOM
    <html>
    <body>
    <h1>Message body of the mail.</h1>
    <img src="cid:logo@some.domain">
    <footer>
    Your Signature    
    </footer>
    </body>
    </html>
    EOM
  message_resource "./logo.png", cid: "logo@some.domain" # [*]
end
```

This code will output log entries to `STDOUT` as follows:

```text
2018/01/25 20:35:09 [crystal-email/12347] INFO [EMail_Client] Start TCP session to your.mx.server.name:25
2018/01/25 20:35:10 [crystal-email/12347] INFO [EMail_Client] Successfully sent a message from <enverope_from@your.mail> to 3 recipient(s)
2018/01/25 20:35:10 [crystal-email/12347] INFO [EMail_Client] Close TCP session to your.mx.server.name:25
```

You can add some option arguments to `EMail.send`.

- `client_name : String` (Default: `"EMail_Client"`)

    Set the client name. It is used as a part of _Message-Id_ header.

- `helo_domain : String` (Default: `"[" + local_ip_addr + "]"`)

    Set the parameter string for SMTP `EHLO`(or `HELO`) command.

- `on_failed : EMail::Client::OnFailedProc` (Default: None)

    Set callback function to be called when sending email is failed while in SMTP session. It will be called with email message object that tried to send, and SMTP command and response history. In this function, you can do something to handle errors: e.g. "_investigating the causes of the fail_", "_notifying you of the fail_", and so on.

    `EMail::Client::OnFailedProc` is an alias of the Proc type `EMail::Message, Array(String) ->`.

- `on_fatal_error : EMail::Client::OnFatalErrorProc` (Default: None)

    Set callback function to be calld when an exception is raised during SMTP hanling. It will be called with the raised Exception instance.

    `EMail::Client::OnFatalErrorProc` is an alias of the Proc type `Exception ->`.

- `use_tls : Bool` (Default: `false`)

    Try to use `STARTTLS` command to send email with TLS encryption.

- `auth : Tuple(String, String)` (Default: None)

    Set login id and password to use `AUTH PLAIN` or `AUTH LOGIN` command: e.g. `{"login_id", "password"}`.

    This option must be use with `ust_tls: true`.

- Logger option 1

    Use external Logger instance.

    - `logger : Logger`

- Logger option 2

    Use default logger setting with some options.

    - `log_io : IO` (Default: `STDOUT`)

        Change logging output from STDOUT. It must be writable.

    - `log_level : Logger::Severity` (Default: `Logger::Severity::INFO`)

        Set log level for SMTP session.

        - `Logger::Severity::DEBUG` : logging all SMTP commands and responses.
        - `Logger::Severity::ERROR` : logging only events stem from some errors.
        - `EMail::Client::NO_LOGGING`(`Logger::Severity::UNKOWN`) : no events will be logged.

    - `log_progname : String` (Default: `"crystal-email"`)

        Set logger progname.

    - `log_formatter : Logger::Formatter` (Default: `EMail::Client::LOG_FORMATTER`)

        Set log formatter.

**NOTE: You can specify only one of Logger option 1 or 2.**


```crystal
# example with option arguments

on_failed = EMail::Client::OnFailedProc.new do |mail, command_history|
  puts mail.data
  puts ""
  puts command_history.join("\n")
end

File.open("./sendamil.log", "w") do |log_file|
  logger = Logger.new(log_file)
  logger.level = Logger::Severity::DEBUG

  EMail.send("your.mx.server.name", 587,
             client_name: "MailBot",
             helo_domain: "your.host.fqdn",
             on_failed:   on_failed,
             use_tls: true,
             auth: {"your_id", "your_password"},
             logger: logger) do

    # same as above
  end
end
```

This will output to `./sendmail.log` file :

```text
I, [2018-01-25 20:52:13 +09:00 #12741]  INFO -- : [MailBot] Start TCP session to your.mx.server.name:587
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- CONN 220 unknown ESMTP
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> EHLO your.host.fqdn
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- EHLO 250 your.mx.server.name / PIPELINING / SIZE 51380224 / ETRN / STARTTLS / ENHANCEDSTATUSCODES / 8BITMIME / DSN
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> STARTTLS
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- STARTTLS 220 2.0.0 Ready to start TLS
I, [2018-01-25 20:52:13 +09:00 #12741]  INFO -- : [MailBot] Start TLS session
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> EHLO your.host.fqdn
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- EHLO 250 your.mx.server.name / PIPELINING / SIZE 51380224 / ETRN / AUTH LOGIN PLAIN / ENHANCEDSTATUSCODES / 8BITMIME / DSN
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> AUTH PLAIN AHlvdXJfaWQAeW91cl9wYXNzd29yZA==
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- AUTH 235 2.7.0 Authentication successful
I, [2018-01-25 20:52:13 +09:00 #12741]  INFO -- : [MailBot] Authentication success with your_id / ********
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> MAIL FROM:<enverope_from@your.mail>
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- MAIL 250 2.1.0 Ok
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> RCPT TO:<to@some.domain>
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- RCPT 250 2.1.5 Ok
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> RCPT TO:<cc@some.domain>
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- RCPT 250 2.1.5 Ok
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> RCPT TO:<bcc@some.domain>
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- RCPT 250 2.1.5 Ok
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> DATA
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- DATA 354 End data with <CR><LF>.<CR><LF>
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> Sending mail data
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- DATA 250 2.0.0 Ok: queued as 856D46004CC6
I, [2018-01-25 20:52:13 +09:00 #12741]  INFO -- : [MailBot] Successfully sent a message from <enverope_from@your.mail> to 3 recipient(s)
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] --> QUIT
D, [2018-01-25 20:52:13 +09:00 #12741] DEBUG -- : [MailBot] <-- QUIT 221 2.0.0 Bye
I, [2018-01-25 20:52:13 +09:00 #12741]  INFO -- : [MailBot] Close session to your.mx.server.name:587
```

### `EMail::Message` object(default receiver of the block for `EMail.send`)

Optionally, you can add mailbox name to `#from`, `#to`, `#cc`, `#bcc` or `#reply_to`.

```crystal
from "your@mail.addr", "Your Name"
```

For attachment files and message resources, you can designate another file name for the recipient.

```crystal
attach "attachment.txt", file_name: "other_name.txt"
```

You can designate mime type of the file explicitly. By default, it will be inferred from the extension of that file: e.g. ".txt" => "text/plain"

```crystal
attach "attachment", mime_type: "text/plain"
```

You can use readable `IO` object instead of the file path. In this case, the `file_name` argument is required. (The `mime_type` argument is also acceptable.)

```
attach some_io, file_name: "other_name.txt"
```

UTF-8 string can be used as follows:

- mail message
- part of header body(when it can be multibyte)
- name of attachment files or message resources

```crystal
subject "メールサブジェクト"
from "your@mail.addr", "山田　太郎"
to "to@mail.addr", "山田　花子"
message <<-EOM
  こんにちは
  EOM
attach "写真.jpg"
```

Attachment files and message resources are always encoded by Base64.

Email messages(text plain message and html message) will be encoded when they have non-ascii characters or lines that is longer than 998 bytes.

## `EMail::Sender`(Concurrent sending)

By using `EMail::Sender` object, you can concurrently send multiple messages by multiple connections.

```crystal
# Concurrent sending example

rcpt_list = ["a@some.domain", "b@some.domain", "c@some.domain", "d@some.domain"]

# create email sender object
sender = EMail::Sender.new("your.mx.server.name", 25)

# start sending emails with concurrently 3 connections
sender.start(number_of_connections: 3) do
  rcpts_list.each do |rcpt_to|
    mail = EMail::Message.new
    mail.from    "your@mail.addr"
    mail.to      rcpt_to
    mail.subject "Concurrent email sending"
    mail.message "message to #{rcpt_to}"
    # enqueue the email to sender
    enqueue mail
  end
end
```

You can:
- set same options as `EMail.send` to `EMail::Sender.new`.
- give `Array(EMail::Message)` or `EMail::Message` object to `EMail::Sender#enqueue`.
- set 2 arguments to `EMail::Sender#start`.
    - 1st one is `number_of_connections` that specify how many connections are used to send messages concurrently.(Default: `1`)
    - 2nd one is `messages_per_connection` that specify how many messages are sent by one connection.(Default: `10`)

**Note: Setting too large `number_of_connections` or `messages_per_connection` will place heavy loads on the SMTP server or occupy its resources. When the SMTP server is not yours, you should choice these parameters very carefully.**

## TODO

- [x] ~~support AUTH LOGIN~~
- [ ] support AUTH CRAM-MD5
- [x] ~~support HTML email~~
- [ ] performance tuning

## Contributors

- [arcage](https://github.com/arcage) ʕ·ᴥ·ʔAKJ - creator, maintainer

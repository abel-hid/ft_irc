# ft_irc — An IRC Server in C++98

An **Internet Relay Chat (IRC) server** written from scratch in **C++98** as part of the 42 / 1337 curriculum.
It accepts many clients at once over TCP, using a single `poll()` loop with non-blocking sockets. It handles authentication, nicknames, channels, private messages, and channel-operator commands.
It works with real IRC clients such as **irssi**, **WeeChat**, **HexChat** or **LimeChat**, and with plain `nc`.

As a bonus, it also includes a **bot** (`Bot`). The bot connects to the server like a normal client and answers commands (`time`, `help`, `level <login>`).

> Made by **abel-hid** & **ylamsiah** 

---

## Table of Contents

1. [What is IRC?](#what-is-irc)
2. [Features](#features)
3. [Project Structure](#project-structure)
4. [Build](#build)
5. [Usage](#usage)
6. [Connecting a Client](#connecting-a-client)
7. [Registration Flow](#registration-flow)
8. [Supported Commands](#supported-commands)
9. [Channel Modes](#channel-modes)
10. [Bonus: The Bot](#bonus-the-bot)
11. [How It Works (Architecture)](#how-it-works-architecture)
12. [Numeric Replies](#numeric-replies)
13. [Testing Tips](#testing-tips)
14. [Resources](#resources)

---

## What is IRC?

IRC is a text-based chat protocol, defined in RFC 1459 and later RFC 2810–2813. Clients connect to a server over TCP and send line-based commands that end with `\r\n`, for example:

```
NICK alice
JOIN #general
PRIVMSG #general :hello everyone!
```

The server relays messages between users, either directly or through **channels** (chat rooms whose names start with `#` or `&`).
This project implements a **single server**. There is no server-to-server networking.

---

## Features

- **Multiple clients at once**: one process, no `fork()`, no threads.
- **Non-blocking I/O**: every socket is set to `O_NONBLOCK` with `fcntl()`.
- **One `poll()` loop**: accepting connections and reading data all go through a single `poll()` call.
- **Password protection**: clients must send the right `PASS` before they can register.
- **Buffering of partial messages**: if a command arrives in several TCP chunks, the pieces are stored per client until the full line (`\r\n` / `\n`) arrives.
- **Channels**: create and join channels, with optional keys, invite-only mode, user limits and topic protection.
- **Channel operators**: `KICK`, `INVITE`, `TOPIC`, `MODE` (`i`, `t`, `k`, `o`, `l`).
- **Several targets in one command**: `JOIN #a,#b key1,key2`, `PRIVMSG alice,#chan :hi`, `PART #a,#b`, `KICK #a u1,u2`.
- **Standard numeric replies** (`001`, `332`, `353`, `366`, `401`, `403`, `461`, `464`, `482`, …), so real IRC clients understand the server.
- **Clean shutdown**: `Ctrl+C` (`SIGINT`) stops the server, and `SIGPIPE` is ignored so writing to a client that already disconnected cannot crash it.
- **Bonus bot**: a separate program that logs in as `Bot` and answers questions.

---

## Project Structure

```
42-IRC/
├── Makefile
├── includes/
│   ├── Server.hpp        # Server class, all command handlers, helpers
│   ├── Client.hpp        # Client state (fd, nick, user, password, buffer…)
│   └── Channels.hpp      # Channel class + IRC numeric reply codes
├── src/
│   ├── main.cpp          # Argument parsing, signals, server start
│   ├── Creat_Server.cpp  # socket/bind/listen, accept, recv, poll loop
│   ├── Server.cpp        # Server getters/setters/checkers, client cleanup
│   ├── Client.cpp        # Client class implementation
│   ├── Channels.cpp      # Channel class implementation
│   ├── methodes.cpp      # Command dispatcher (commandsIrc)
│   ├── utils.cpp         # Port/password validation, prefix stripping
│   ├── pass.cpp          # PASS
│   ├── nick.cpp          # NICK (registration + nick change)
│   ├── user.cpp          # USER + welcome messages
│   ├── join.cpp          # JOIN
│   ├── part.cpp          # PART
│   ├── privmsg.cpp       # PRIVMSG (to users and channels)
│   ├── topic.cpp         # TOPIC
│   ├── kick.cpp          # KICK
│   ├── invite.cpp        # INVITE + channel name validation
│   ├── mode.cpp          # MODE (+/- i t k o l)
│   ├── quit.cpp          # QUIT
│   └── Bot.cpp           # Server side of the bot: sends bot replies to users
└── bonus/
    ├── Bot.hpp           # Bot class
    ├── Bot.cpp           # Bot login + receive loop
    ├── botCmd.cpp        # Bot command handling, reads bot.txt
    └── Bonus.cpp         # Bot main()
```

---

## Build

You need a Unix-like system (Linux or macOS) and a C++ compiler (`c++` / `g++` / `clang++`).
The code is compiled with `-Wall -Wextra -Werror -std=c++98`.

| Command            | Description                                  |
|--------------------|----------------------------------------------|
| `make`             | Build the server → `ircserv`                 |
| `make bonus`       | Build the bot → `Bot`                        |
| `make clean`       | Remove server object files (`OBJ_DER/`)      |
| `make fclean`      | `clean` + remove `ircserv` and `Bot`         |
| `make re`          | Rebuild the server from scratch              |
| `make clean_bonus` | Remove bot object files (`OBJ_DER_BONUS/`)   |
| `make re_bonus`    | Rebuild the bot from scratch                 |

---

## Usage

```bash
./ircserv <port> <password>
```

| Argument     | Rules                                                     |
|--------------|-----------------------------------------------------------|
| `<port>`     | Digits only, between **1024** and **65535**               |
| `<password>` | Any non-empty string. Clients must send it with `PASS`.   |

Example:

```bash
./ircserv 6667 secret
```

The server prints a banner and then logs every connection and every command it receives:

```
[INFO] Server is running on port [6667]
[INFO] New client connected with ip address [127.0.0.1] From port [53422]
[CMD] NICK From [127.0.0.1] From port [53422]
```

Press `Ctrl+C` to stop the server.

---

## Connecting a Client

### With netcat

```bash
nc -C localhost 6667        # -C sends \r\n line endings (Linux)
```

Then type:

```
PASS secret
NICK alice
USER alice 0 * :Alice Liddell
JOIN #general
PRIVMSG #general :Hello!
```

### With irssi

```
/connect localhost 6667 secret alice
/join #general
```

### With HexChat / LimeChat

Add a new network with server `localhost/6667`, set the server password to `secret`, and connect.

---

## Registration Flow

A client must **register** before it can use any other command:

1. `PASS <password>`: must come **first** and must match the server password. A wrong password returns `464 Password incorrect`.
2. `NICK <nickname>`: allowed characters are letters, digits and `- _ [ ] \ ` ^ { }`. A nickname that is already taken returns `433`.
3. `USER <username> <hostname> <servername> :<real name>`

When all three are accepted, the server sends the welcome replies (`001`, `003`).
Any other command sent before registration gets `451 You have not registered`.
Sending `PASS` or `USER` again after registering gets `462 You may not reregister`.

---

## Supported Commands

Command names are case-insensitive (`join` is the same as `JOIN`).

| Command   | Syntax                                         | Description |
|-----------|------------------------------------------------|-------------|
| `PASS`    | `PASS <password>`                              | Authenticate with the server password. |
| `NICK`    | `NICK <nickname>`                              | Set the nickname, or change it later. A change is announced to everyone in your channels. |
| `USER`    | `USER <user> <host> <server> :<realname>`      | Set username and real name. Completes registration. |
| `JOIN`    | `JOIN <#chan>{,<#chan>} [<key>{,<key>}]`       | Join (or create) one or more channels. The creator becomes a channel operator. |
| `PART`    | `PART <#chan>{,<#chan>} [:<reason>]`           | Leave one or more channels. |
| `PRIVMSG` | `PRIVMSG <target>{,<target>} :<text>`          | Send a message to users and/or channels. |
| `TOPIC`   | `TOPIC <#chan> [:<new topic>]`                 | Show or set the channel topic (only operators can set it when `+t` is on). |
| `KICK`    | `KICK <#chan> <user>{,<user>} [:<reason>]`     | Remove a user from a channel *(operator only)*. |
| `INVITE`  | `INVITE <nick> <#chan>`                        | Invite a user to a channel *(operator only)*. |
| `MODE`    | `MODE <#chan> [(+\|-)<modes> [params…]]`       | Show or change channel modes *(operator only to change)*. |
| `QUIT`    | `QUIT [:<message>]`                            | Disconnect. Members of your channels receive a `QUIT` message. |
| `PONG`    | `PONG <token>`                                 | Accepted and ignored (keeps clients happy). |

**Channel names** must start with `#` or `&`, contain no spaces, commas or `^G` (BELL), and be at most 50 characters long.

**Empty channels** are deleted automatically when their last member leaves or disconnects.

---

## Channel Modes

Set or remove modes with `MODE <#chan> +<flags>` / `-<flags>`. Several flags can go in one command, e.g. `MODE #chan +itk secret`.

| Mode | With parameter          | Meaning |
|------|-------------------------|---------|
| `i`  | –                       | **Invite-only**: users can join only if an operator invited them. |
| `t`  | –                       | **Topic lock**: only operators can change the topic. |
| `k`  | `+k <key>` / `-k`       | **Channel key**: users need the password to `JOIN`. |
| `o`  | `+o <nick>` / `-o <nick>` | Give or take **operator** status. |
| `l`  | `+l <count>` / `-l`     | **User limit**: maximum number of members. |

`MODE <#chan>` with no flags shows the modes that are currently active.

---

## Bonus: The Bot

The bot is a **separate program** that connects to the server like any other client, using the nickname **`Bot`**.

### Running it

```bash
make bonus
./ircserv 6667 secret          # terminal 1: start the server first
./Bot 6667 secret              # terminal 2: start the bot
```

The bot connects to `localhost`, sends `PASS`, `USER` and `NICK`, and then waits for messages.
If the password is wrong, it prints an error and exits.

> ⚠️ The bot reads student data from **`bot.txt`** in the **current working directory**. The file has one `login:level` per line, for example `aallali:22.23`. Make sure this file exists before you run `level` queries.

### Bot commands

Type these as a registered user, either directly or with `PRIVMSG Bot :<command>`:

| Command          | Reply |
|------------------|-------|
| `time`           | 🕒 The server's current date and time. |
| `help`           | The list of available IRC commands and their parameters. |
| `level <login>`  | The student's level from `bot.txt`, or "not found". |

### How the bot round-trip works

1. The user sends `time`, `help` or `level …`. The server sees it is a bot command and forwards it to the client named `Bot`.
2. The bot builds an answer and sends it back to the server as a line that starts with `🤖:`.
3. The server recognises the `🤖:` prefix (in `Server::comdBotBot`, [src/Bot.cpp](src/Bot.cpp)), formats a friendly reply, and delivers it to the user who asked.

---

## How It Works (Architecture)

### 1. Startup — [src/main.cpp](src/main.cpp), [src/utils.cpp](src/utils.cpp)
- `parsingInput()` checks the port and password.
- It installs a `SIGINT` handler (sets `close_ser = true`) and ignores `SIGPIPE`.
- It creates the `Server`, sets the port and password, then calls `initServer()` and `runServer()`.

### 2. Socket setup — `Server::initServer()` in [src/Creat_Server.cpp](src/Creat_Server.cpp)
```
socket(AF_INET, SOCK_STREAM) → setsockopt(SO_REUSEADDR) → fcntl(O_NONBLOCK)
→ bind(INADDR_ANY:port) → listen()
```

### 3. Event loop — `Server::runServer()`
- A `std::vector<pollfd>` starts with the listening socket.
- Each time through the loop, `poll()` checks every fd:
  - **Listening socket ready** → `accept_client()` accepts the connection, makes it non-blocking, creates a `Client`, stores its IP and port, and adds its fd to the poll list.
  - **Client socket ready** → `receve_msg(fd)`.
- The loop runs until `Ctrl+C`.

### 4. Reading and buffering — `Server::receve_msg()`
- `recv()` reads up to 4096 bytes.
- `0` or an error means the client disconnected: it is removed from every channel, freed, and its socket is closed.
- If no newline has arrived yet, the data is stored in the client's buffer (`Client::str`).
- Once a newline arrives, the buffered data and the new data are joined. A leading `:prefix` is removed (`update_str`), and the line goes to `execute_command()`.

### 5. Dispatching — `Server::commandsIrc()` in [src/methodes.cpp](src/methodes.cpp)
- The line is split into words, and the command name is converted to upper case.
- **Not registered yet** → only `PASS`, `NICK` and `USER` are allowed.
- **Registered** → the command goes to the matching handler (`JOIN`, `PRIVMSG`, `MODE`, `INVITE`, …). Anything else gets `421 Unknown command`.

### 6. Data model
- **`Server`** owns `std::map<int, Client*> clients` (keyed by fd) and `std::map<std::string, Channel*> channels` (keyed by name).
- **`Client`** stores the fd, nickname, username, real name, hostname, IP, the password it sent, its registration state, and its partial-message buffer.
- **`Channel`** stores its name, topic, key, user limit, the flags `inviteOnly` and `topicRestriction`, and three `std::set`s: `users`, `operators` and `invitedUsers`.

### 7. Cleanup
- `remove_client_from_channels()` sends a `QUIT` to the client's channel members, removes the client from the user and operator lists, and deletes channels that end up empty.
- `~Server()` frees every remaining `Client` and `Channel`.

---

## Numeric Replies

These are the main numeric codes the server sends (defined in [includes/Channels.hpp](includes/Channels.hpp)):

| Code | Name                    | When |
|------|-------------------------|------|
| 001  | RPL_WELCOME             | Registration succeeded |
| 331  | RPL_NOTOPIC             | Channel has no topic |
| 332  | RPL_TOPIC               | Channel topic |
| 353  | RPL_NAMREPLY            | Channel member list |
| 366  | RPL_ENDOFNAMES          | End of member list |
| 401  | ERR_NOSUCHNICK          | Target nick doesn't exist |
| 403  | ERR_NOSUCHCHANNEL       | Channel doesn't exist |
| 404  | ERR_CANNOTSENDTOCHAN    | You are not in the channel |
| 411  | ERR_NORECIPIENT         | `PRIVMSG` with no target |
| 412  | ERR_NOTEXTTOSEND        | `PRIVMSG` with no text |
| 421  | ERR_UNKNOWNCOMMAND      | Unknown command |
| 431/432 | ERR_NONICKNAMEGIVEN / ERR_ERRONEUSNICKNAME | Missing or invalid nickname |
| 433  | ERR_NICKNAMEINUSE       | Nickname already taken |
| 442  | ERR_NOTONCHANNEL        | You are not on that channel |
| 443  | ERR_USERONCHANNEL       | User is already on the channel |
| 451  | ERR_NOTREGISTERED       | Command sent before registering |
| 461  | ERR_NEEDMOREPARAMS      | Missing parameters |
| 462  | ERR_ALREADYREGISTRED    | `PASS`/`USER` sent again |
| 464  | ERR_PASSWDMISMATCH      | Wrong password |
| 471/473/475 | Channel full / invite-only / wrong key | `JOIN` refused |
| 476  | ERR_BADCHANMASK         | Invalid channel name |
| 482  | ERR_CHANOPRIVSNEEDED    | You are not a channel operator |

---

## Testing Tips

**Partial messages.** Test that a command split across several packets is put back together. In `nc`, press `Ctrl+D` to send what you have typed so far without a newline:

```bash
nc localhost 6667
# type "PA", press Ctrl+D, type "SS sec", press Ctrl+D, type "ret", press Enter
# → the server receives one full line: "PASS secret"
```

**Many clients.** Open several terminals or clients, join the same channel, and check that messages, `KICK`, `MODE` and `TOPIC` reach everyone.

**Abrupt disconnect.** Kill a client (`Ctrl+C` on `nc`). The server must keep running, and the other members of its channels should receive a `QUIT`.

**Operator checks.** Try `KICK`, `INVITE`, `MODE` and `TOPIC` (with `+t`) as a user who is not an operator. You should get `482`.

---

## Resources

- [RFC 1459 — Internet Relay Chat Protocol](https://datatracker.ietf.org/doc/html/rfc1459)
- [RFC 2812 — IRC: Client Protocol](https://datatracker.ietf.org/doc/html/rfc2812)
- [Modern IRC documentation](https://modern.ircdocs.horse/)
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)

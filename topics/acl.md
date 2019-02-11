# ACL

The Redis ACL, short for Access Control List, is the feature that allows certain
connections to be limited in the commands that can be executed and the keys
that can be accessed. The way it works is that, after connecting, a client
requires to authenticate providing a username and a valid password: if
the authentication stage succeeded, the connection is associated with a given
user and the limits the user has. Redis can be configured so that new
connections are already authenticated with a "default" user (this is the
default configuration), so configuring the default user has, as a side effect,
the ability to provide only a specific subset of functionalities to connections
that are not explicitly authenticated.

In the default configuration, Redis 6 (the first version to have ACLs) works
exactly like older versions of Redis, that is, every new connection is
capable of calling every possible command and accessing every key, so the
ACL feature is backward compatible with old clients and applications. Also
the old way to configure a password, using the **requirepass** configuration
directive, still works as expected, but now what it does is just to
set a password for the default user.

The Redis `AUTH` command was extended in Redis 6, so now it is possible to
use it in the two-arguments form:

    AUTH <username> <password>

When it is used according to the old form, that is:

    AUTH <password>

What happens is that the username used to authenticate is "default", so
just specifying the password implies that we want to authenticate against
the default user. This provides perfect backward compatibility with the past.

## When ACLs are useful

Before using ACLs you may want to ask yourself what's the goal you want to
accomplish by implementing this layer of protection. Normally there are
two main goals that are well served by ACLs:

1. You want to improve security by restricting the access to commands and keys, so that untrusted clients have no access and trusted clients have just the minimum access level to the database in order to perform the work needed. For instance certain clients may just be able to execute read only commands.
2. You want to improve operational safety, so that processes or humans accessing Redis are not allowed, because of software errors or manual mistakes, to damage the data or the configuration. For instance there is no reason for a worker that fetches delayed jobs from Redis to be able to call the `FLUSHALL` command.

Another typical usage of ACLs is related to managed Redis instances. Redis is
often provided as a managed service both by internal company teams that handle
the Redis infrastructure for the other internal customers they have, or is
provided in a software-as-a-service setup by cloud providers. In both such
setups we want to be sure that configuration commands are excluded for the
customers. The way this was accomplished in the past, via command renaming, was
a trick that allowed us to survive without ACLs for a long time, but is not
ideal.

## Configuring ACLs using the ACL command

ACLs are defined using a DSL (domain specific language) that describes what
a given user is able to do or not. Such rules are always implemented from the
first to the last, left-to-right, because sometimes the order of the rules is
important to understand what the user is really able to do.

By default there is a single user defined, that is called *default*. We
can use the `ACL LIST` command in order to check the currently active ACLs
and verify what the configuration of a freshly stared and unconfigured Redis
instance is:

    > ACL LIST
    1) "user default on nopass ~* +@all"

The command above reports the list of users in the same format that is
used in the Redis configuration files, by translating the current ACLs set
for the users back into their description.

The first two words in each line are "user" followed by the username. The
next words are ACL rules that describe different things. We'll show in
details how the rules work, but for now it is enough to say that the default
user is configured to be active (on), to require no password (nopass), to
access every possible key (`~*`) and be able to call every possible command
(+@all).

Also, in the special case of the default user, having the *nopass* rule means
that new connections are automatically authenticated with the default user
without any explicit `AUTH` call needed.

## ACL rules

The following is the list of the valid ACL rules. Certain rules are just
single words that are used in order to activate or remove a flag, or to
perform a given change to the user ACL. Other rules are char prefixes that
are concatenated with command or cagetories names, or key patterns, and
so forth.

* `on`: Enable the user: it is possible to authenticate as this user.
* `off`: Disable the user: it's no longer possible to authenticate with this user, however the already authenticated connections will still work. Note that if the default user is flagged as *off*, new connections will start not authenticated and will require the user to send `AUTH` or `HELLO` with the AUTH option in order to authenticate in some way, regardless of the default user configuration.
* `+<command>`: Add the command to the list of commands the user can call.
* `-<command>`: Remove the command to the list of commands the user can call.
* `+@<category>`: Add all the commands in such category to be called by the user, with valid categories being like @admin, @set, @sortedset, ... and so forth, see the full list by calling the `ACL CAT` command. The special category @all means all the commands, both the ones currently present in the server, and the ones that will be loaded in the future via modules.
* `-@<category>`: Like `+@<category>` but removes the commands from the list of commands the client can call.
* `+<command>|subcommand`: Allow a specific subcommand of an otherwise disabled command. Note that this form is not allowed as negative like `-DEBUG|SEGFAULT`, but only additive starting with "+". This ACL will cause an error if the command is already active as a whole.
* `allcommands`: Alias for +@all. Note that it implies the ability to execute all the future commands loaded via the modules system.
* `nocommands`: Alias for -@all.
`~<pattern>`: Add a pattern of keys that can be mentioned as part of commands. For instance `~*` allows all the keys. The pattern is a glob-style pattern like the one of KEYS.  It is possible to specify multiple patterns.
* `allkeys`: Alias for `~*`.
* `resetkeys`: Flush the list of allowed keys patterns. For instance the ACL `~foo:* ~bar:* resetkeys ~objects:*`, will result in the client only be able to access keys matching the pattern `objects:*`.
* `><password>`: Add this passowrd to the list of valid passwords for the user. For example `>mypass` will add "mypass" to the list of valid passwords.  This directive clears the *nopass* flag (see later). Every user can have any number of passwords.
* `<<password>`: Remove this password from the list of valid passwords. Emits an error in case the password you are trying to remove is actually not set.
* `nopass`: All the set passwords of the user are removed, and the user is flagged as requiring no password: it means that every password will work against this user. If this directive is used for the default user, every new connection will be immediately authenticated with the default user without any explicit AUTH command required. Note that the *resetpass* directive will clear this condition.
* `resetpass`: Flush the list of allowed passwords. Moreover removes the *nopass* status. After *resetpass* the user has no associated passwords and there is no way to authenticate without adding some password (or setting it as *nopass* later).
* `reset` Performs the following actions: resetpass, resetkeys, off, -@all. The user returns to the same state it has immediately after its creation.

TODO list:

* Make sure to specify that modules commands are ignored when adding/removing categories.
* Document cost of keys matching with some benchmark.
* Document how +@all also includes module commands and every future command.
* Document how ACL SAVE is not included in CONFIG REWRITE.
* Document backward compatibility with requirepass and single argument AUTH.
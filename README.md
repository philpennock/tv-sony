tv-sony
=======

Here is a `tv` command, for remote control of a television.

This uses the network interfaces provided by Sony Bravia TVs, from 2013
onwards.  Support for other makes of TV is unknown, but I'm willing to take
patches for anything "sufficiently similar".

This tool lets you control the TV from the command-line, and with a REPL, and
with a curses mode to take keystrokes.  That means:

1. You can script control of the TV by just invoking a command to send
   instructions
2. The Read-Eval-Print-Loop idiom is available; you have full readline history
   and editing, and context-aware tab-completion built in.  The command can
   tab-complete its own commands, all remote control buttons known by the TV,
   and the names of various apps installed on the TV.
3. You can use cursor keys (and/or vi keys) and space/enter for basic control
   without needing to type commands and hit enter.

It's written in Python, developed against Python 3.8.


## Usage

```sh
$ tv --help
[ output of normal command-line flags ]
$ tv :help
[ lots of information about available commands]
```

```sh
$ tv on mute hdmi 4 volume 24 unmute
```

The `tv` command figures out that there are four commands there.  Each one is
sent to the TV half a second apart.

```sh
$ tv on mute key Down key Down key Right key Confirm unmute
$ tv on mute key [ Down Down Right Confirm ] unmute
```

Two ways of writing the same sequence: the `key` command takes one keyname to
send, but if that key-name is `[` then all parameters up until a matching `]`
(or the end) are taken to be keys to send.

```sh
$ tv
tv> :help
tv> key
tv key> Le<tab>
tv key> Left
tv key> Up Up Down Down Left Right Left Right Green Blue Confirm
tv key> <Ctrl-D>
tv> :keypad
 [remote-control here]
 q  (to quit)
tv> <Ctrl-D>
$
```

A flow where you enter the REPL and can use history navigation, tab
completion, etc.  After sending various key presses, we leave the remote
button mode and instead ask for a keypad, where we can use cursor keys and
space/enter to control things, before typing `q` to quit (as noted by the
on-screen help), and then exiting.

(The list-of-keys mode speeds up slightly and only waits a quarter of a second
between each key; these delays were chosen arbitrarily).

```sh
$ tv :keypad
```

Dive straight into the keypad control mode.


## Setup

At present, the only authentication mode supported is to use a PIN, which the
TV calls a "Pre-Shared Key".  You will need to enable this in the TV menus
first.

Where this is in the menus changes with TV operating system updates.  Loosely,
head into the settings / cog, go to Network, and look for IP control and
pre-shared setup.  The TV may mention something about this being for A/V
professionals.

When you set a pre-shared key, **beware**: the digits on your remote control
_do not enter ASCII digits_: they enter some other set of Unicode digits.
This can be a pain to debug.  Use the on-screen keyboard to enter a key,
instead of the remote control's digit buttons.

```sh
$ tv --host 192.168.1.2 --pin 1234 :write myname
```

A hostname can be used if you have working DNS at home, but otherwise you will
need the TV's IP address.  The parameter to `:write` becomes the name for this
TV and will be automatically set as the default TV.


## Installation

At present, this is a single script file.  It has grown organically and is a
bit unwieldy, but has the advantage that you can just copy it, mark it
executable, and run it.

You will need two Python packages from outside of the standard library:

1. `requests` : because it's the sanest way to manage HTTP in Python
2. `cryptography` : because the TV implements a custom crypto system and we
   need to mess around at the `hazmat` layer to try to talk to it.

If anybody ever shows interest in this command and it starts to grow then I
will probably split it into modules and make it `pip` installable.  For now,
you have the simple version which is from some idle time code tinkering.


## Known Problems

### Text Fields

I haven't yet managed to get text field reading and writing working.

My TV does not support HTTPS, and the way Sony get around the fact that some
text fields might contain passwords, is to offer a custom encryption scheme.
Please note: nothing here is my choice, I have no influence over what's
required, I am trying to interoperate with a scheme designed by Sony.

The TV has an RSA-2048 key and the client generates keys for AES-128/CBC,
updates the IV for each command, and encrypts the key:IV to the TV's RSA key
and sends that along side text encrypted with the TV.  A response will be
encrypted with the same key (and it's not clear how the IV is changed, if at
all, to prevent attacks from re-using the IV for send and reply).

I haven't gotten it working.  But the code is still in there.

### TV PIN/PSK Storage

I don't view this secret as particularly valuable, nor the secrets storage
layer available on Linux as particularly useful (especially when compared to
that offered by macOS), so the pre-shared key is stored unencrypted in a file
on disk, `~/.config/tv/$TVNAME.pin`

This might change but to the best of my knowledge, this secret can't do
anything dangerous and does not warrant more complexity.  If it leaks and is a
problem then you can change the secret in your TV.


## Technical Details

Unlike certain other companies, Sony provide online technical documentation
for their APIs: <https://pro-bravia.sony.net/develop/index.html>.

This is excellent and in most places detailed and sufficient.  This support
for device owners and professionals to work with Sony is great to see.

There are three different APIs supported by the TV, and this command uses two
of them.  The remote control system is available while the TV is "turned off",
as long as it has power, so the small sub-system for this is always-on.

There's a REST API, which allows for various queries, text-field
manipulations, changing inputs and so forth.

And there's a SOAP XML API, "IRCC-IP", "InfraRed Compatible Control over
Internet Protocol".  This is how a remote control is emulated.

The complete list of remote control _keys_ is provided by the TV and we cache
it.  We match in case-insensitive fashion.  We only hardcode some alias names
which will be tried if the TV does not provide a matching key name.

Some functions, such as changing the TV's input source, can be done with
either API.


## Files

The `tv` command is loosely XDG compliant.  That is, we use the same defaults
and the same environment controls, but we don't walk lists of paths, we just
use the first path in any override environment variables.

Thus the default locations are:

* `~/.config/tv/` : where we keep configuration.  This includes the hostname
  and PIN, and which TV is the current default.  You can also create a
  `readline.conf` file in that directory and it will be passed to the Python
  bindings to your system readline/libedit/whatever.
* `~/.cache/tv/` : where we keep copies of data pulled from the TV (RSA keys,
  list of installed apps) and keep our readline history.
* `~/.local/share/tv/` : locally generated data which should persist is stored
   here; at present, that's client RSA keys, which are not (yet?) used.

## Misc

The names "Sony" and "Bravia" are trademarks, used here descriptively without
explicit permission.

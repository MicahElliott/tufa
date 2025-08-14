# Tufa: Recipes for DIY 2FA etc CLI

Light-weight Do-It-Yourself Two-Factor Authenticator (2FA, from spreadsheet),
Password-Manager (PWM), etc, recipes for terminal-based workflows.

It's a "tufa" because it's also a "two-fer-one": PWM and 2FA all-in-one flow.
And that's pretty OK to do, as explained below.

Install a couple tiny tools with your package-manager and you get a simple
and secure 2FA CLI (and PWM) setup. All in a few minutes.

With this, you don't need to continuously "run" a heavy PWM client or browser
plugin or any other app. This is an altogether minimal setup for any login. It
even shell-completes the site entries registered, and puts the username,
password, and TOTP onto separate clipboards, all in one completed command.

## Demo

Note that [`rbw`](https://github.com/doy/rbw) is a CLI Bitwarden PWM client in
use and it is assumed you're using [Bitwarden](https://bitwarden.com/).

```shell
% export TUFA_SECRETS_FILE=~/.sec/tufa.tsv.gpg

% tufa <TAB>
First unlock with: rbw unlock

% rbw unlock
password: *** (promted for master password)

% tufa <TAB>
-- site
dropbox
heroku
yahoo
...

% tufa dropbox
Using Bitwarden to fetch login/password.
«password copied to primary»
Username: alicerobertson
URI: https://www.dropbox.com

Using GPG to unlock and look up Tufa code.
...
You have 34 seconds remaining for TOTP code.
«totp copied to clipboard»
«username copied to clipboard»
```

Notice that no password or code is ever visible. The current setup uses
`xclip` to store the password to your system clipboard, and so is never
visible to anyone, but is middle-click pasteable. Then it stores username and
TOTP code to your other clipboard which is then easily selected by your
clipboard manager ([parcellite](https://github.com/rickyrockrat/parcellite) in
my case) via hotkey. Because there was only a few seconds left in the 30s totp
window, it grabbed the subsequent one.

## The Recipe

You maintain a plaintext TSV ("spreadsheet", if you like) containing sites
you've got using 2FA. It looks like this:

```tsv
kid	kname	site	login	namespace	kcode
1	heroku	github.com	you@example.com	code	DEADBEEF...
2	hotmail	google.com	you@example.com	email	ABC123...
3	dropbox	drobox.com	justyou	backup	35ft xyz4 ...
...
```

(Really, `kname` and `kcode` are the only fields needed.)

That TSV is encrypted via `gpg` in our case. If you use
[Emacs](https://vxlabs.com/2021/03/21/gnupg-pinentry-via-the-emacs-minibuffer/)
or [Vim](https://github.com/jamessan/vim-gnupg), you get editing
(adding/deleting/changing entries) for free, which really simplifies the
tooling. Probably other editors purport to do this too. (You could use
[sops](https://github.com/getsops/sops) or
[age](https://github.com/FiloSottile/age) instead of `gpg`.)

To encrypt such a file manually (without editor magic):

```shell
% gpg --symmetric --pinentry-mode loopback mykeys.tsv     # encrypt to mykeys.tsv.gpg
% gpg --decrypt   --pinentry-mode loopback mykeys.tsv.gpg # decrypt (what tufa does)
```

A TSV has the nice benefit that you can import it to a DB like SQLite if you
want to do fancier things with it (eg, encrypted sync with Turso?). You can
also easily read/edit (and even print!) the TSV, and **save it somewhere safe
as a backup**.

You should know that you can generate the same TOTP from any authenticator app
if it has the same _secret key_ registered for a site. So whether you're used
to Authy or Ente or Okta, know that you can redundantly use the same secret an
any of those and get the same 6-digit code.

So the trick is to simply send any one of those 32-character (usually, though
they vary in length) secrets through any authenticator. The tiniest (but
powerful) CLI I can find for this is `oathtool` (90kb). So it's just a matter of:

```shell
% echo 'AbCd9876...' | oathtool -b --totp -d6 -
122333
```

And `tufa` does just that, after grabbing the desired secret out of the TSV.

Note that we're not using Bitwarden's 2FA add-on feature. Enough experts say
that your PWM should not be combined with your authenticator, since breaching
that one exposes full access. So let's separate them. (However, we should all
upgrade to use their paid version at $10/year in order to support them.)

Note that you're seeing a PWM and 2FA authenticator running in the same tool
(`tufa`), on the same device. This would usually be a bad idea. But since they
are otherwise disconnected, and protected in totally different ways, we should
be OK. Might be best to have `gpg` prompt you on each use.

A practical backup/redundancy process for adding your 2FA secrets might
involve always creating accounts/entries on your desktop (in your TSV), and
then pasting them into your phone app, like Ente. And you can always get them
into Ente later using `qrencode`/`timg`.

## Installation

### Fedora/RedHat etc

``` shell
% sudo dnf install rbw timg qrencode oathtool pwgen
```

### MacOS

```shell
% brew install oath-toolkit pinentry-mac gpg pwgen
```

## Extra Goodies

If you use Bitwarden to manage passwords, you can install `rbw` CLI to avoid
using a bulky client app. But if you only want a CLI 2FA solution (and no PWM),
it's mostly just:

```shell
% grep $'\t'$SOMESITE$'\t' $TUFA_SECRETS_FILE | cut -f6 | oathtool -b --totp -d6 -
```

If you want to generate QR-codes (eg, for importing into Ente or some other,
and this even works in Emacs terminals like eat):

```shell
% qrencode -o foo.png "otpauth://totp/$SITE:$EMAIL?algorithm=SHA1&digits=6&issuer=$SITE&period=30&secret=$SECRET"
% timg foo.png
```

Zsh completion of entries is as simple as putting the provided `_tufa`
completion file on your `$fpath`.

You can generate passwords with the provided `pwg` script. It simply calls
`pwgen` with some appropriate parameters (length 20), and puts it onto your
clipboard. And you can pepper it from there.

## Other Solutions

[totp-cli](https://github.com/yitsushi/totp-cli) is a really nice single CLI
that is a lot like `tufa`. I used it and would've adopted it. But it's 2500
lines of Go plus deps (5MB), and I usually prefer tiny shell scripts/recipes
over new tools. Its dump/export feature is YAML-based instead of Tufa's TSV,
but is the same idea.

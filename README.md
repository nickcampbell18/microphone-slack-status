# microphone-slack-status

> Set your Slack status when your OSX microphone is engaged 🟡

This repository contains a simple Ruby script which modifies your Slack status whenever
your microphone is engaged on OSX.

I wrote this because I often use a variety of video-conferencing apps (e.g. Microsoft
Teams, Slack, Zoom, webrtc apps like Whereby, etc), as well as pairing apps like Tuple.
Rather than joining each of those apps explicitly to Slack (if they even support that),
this app instead listens on the OSX system log for when the microphone is engaged, and
triggers a webhook which updates your Slack status.

> In recent versions of OSX, you can tell when you're in this state because a little
> orange dot appears in the control center in the top right of the menubar:
> https://lowtechguys.com/yellowdot/

You can customise the behaviour per-app, so when you start using Tuple you can use a Slack
status like `:keyboard: Pairing`, or use the generic `meeting` mode.

## Usage

I haven't bundled this as a gem because it's OSX-only and probably only works on very
recent versions of OSX. YMMV 🤷‍♂️.

First, clone this repo and switch to it:

```sh
git clone https://github.com/nickcampbell18/microphone-slack-status.git
```

Next, you'll need to setup webhooks for updating your status. I used the awesome
https://www.statushook.cool/ - you can register single-purpose webhook URLs for specific
statuses. You should setup a `meeting` webhook (or modify the Ruby file to pick your own
default name), and also a `clear` one (this is done using `/statushook add emoji clear`).

Once you've configured a few URLs using their `/statushook` commands in Slack, you need to
store them locally them, creating a file called `config.yml`:

```yaml
---
:hooks:
  :meeting: https://api.statushook.cool/v1/prod/webhook/fire?id=...
  :pairing: https://api.statushook.cool/v1/prod/webhook/fire?id=...
  :clear: https://api.statushook.cool/v1/prod/webhook/fire?id=...
  ...

:apps:
  Tuple.app: pairing
```

Next, remember to set the executable bit (`chmod +x ./microphone-slack-status`). Optionally,
you can put the app in your $PATH if you want to use it from other folders.

Then, run it:

```sh
$ ./microphone-slack-status
D, [2022-03-10T17:24:55.424943 #95706] DEBUG -- : Loaded config: {:hooks=>{:pairing=>"https://api.statushook.cool/v1/prod/webhook/fire?id=...", :meeting=>"https://api.statushook.cool/v1/prod/webhook/fire?id=...", :clear=>"https://api.statushook.cool/v1/prod/webhook/fire?id=..."}, :apps=>{"Tuple.app"=>"pairing"}}
```

With the config above, when launching an app called `Tuple.app`, the URL associated with
the name `:pairing` will receive a HTTP GET. Anything else will send a HTTP GET to the
default `:meeting` webhook. When the microphone is disabled, the `:clear` webhook is fired.

You must leave the application running to trigger the webhooks - you could probably set it
up as a system service if you wanted this to persist after reboots!

## Deeper customisation

Take a look at the source - the design is deliberately simple, as it's just glueing
several tools together. It should be fairly easy to extend it for your use case (e.g. if
you want to talk to the Slack API directly, or talk to Twitter instead, etc).

## How does it work?

The key thing here is figuring out the best way to get microphone status. Some websites
pointed to using the `ioreg` command-line tool, which gives a lot of information about the
audio devices. On my machine, with a few different input & output devices, it would tell
me devices were active (`IOAudioEngineState" = 1`) but that didn't clarify if they were in use.

So, I did some poking around and came across `/usr/bin/log` which seems to capture a
variety of OSX system log events. After some trial and error I realised that the
`coreaudiod` system emits events when the microphone is used, and the most useful event
can be found like this:

```
$ /usr/bin/log stream --predicate 'sender == "CoreAudio" && eventMessage contains "running: "'
Filtering the log data using "sender == "CoreAudio" AND composedMessage CONTAINS "running: ""

Timestamp                       Thread     Type        Activity             PID    TTL
2022-03-10 17:28:02.681896+0000 0xbc9f67   Default     0x0                  204    0    coreaudiod: (CoreAudio)  SystemStatusWrapper::PublishRecordingClientInfo: Report client 39644 running: yes
2022-03-10 17:28:03.199540+0000 0xbc9f67   Default     0x0                  204    0    coreaudiod: (CoreAudio)  SystemStatusWrapper::PublishRecordingClientInfo: Report client 39644 running: no
```

In this case, `39644` is the PID of my app (Tuple), and `running: yes` means it has
engaged the microphone. So, we just need to parse the new lines emitted from this process
and react appropriately.

# Contributing & license

I welcome contributions to stability (less buggy) or applicability (works on a wider set
of platforms). This project is MIT licensed - feel free to use it however you like!

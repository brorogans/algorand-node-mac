# Running an Algorand Node on Mac

This guide details how to get a node set up locally on a Mac

## Installation

As per the [official guide](https://developer.algorand.org/docs/run-a-node/setup/install/)

First make a destination folder. If you want this to run on an external NVMe drive install it there.

```shell
mkdir ~/node
cd ~/node
```

Install the updater script

```shell
curl https://raw.githubusercontent.com/algorand/go-algorand/rel/stable/cmd/updater/update.sh -O
```

Make it executable

```shell
chmod 744 update.sh
```

Run it

```shell
./update.sh -i -c stable -p ~/node -d ~/node/data -n
```

Add the Algorand path to your shell `// sudo nano ~/.zsh_config etc`

```shell
export ALGORAND_DATA="$HOME/node/data"
export PATH="$HOME/node:$PATH"
```

## Running a node

Open a terminal window and run

> [!IMPORTANT]
> If you want to run as a service, [set that up first](#install-as-a-service) then come back here and ignore the first step (`goal node start`)

```shell
goal node start
```

Open another window and run

```shell
goal node status -w 1000
```

Open a third terminal window and enter

```shell
goal node catchup $(curl https://algorand-catchpoints.s3.us-east-2.amazonaws.com/channel/mainnet/latest.catchpoint)
```

You should now see the output in the first window update to be something like

```shell
Last committed block: 45386623
Sync Time: 25.9s
Catchpoint: 45420000#35C7MMFM46N7R3CW54LUWWZV4ELUVKUPTBP3M4KRJANXRT4TQMVA
Catchpoint total accounts: 21656096
Catchpoint accounts processed: 798208
Catchpoint accounts verified: 0
Catchpoint total KVs: 2195387
Catchpoint KVs processed: 0
Catchpoint KVs verified: 0
Genesis ID: mainnet-v1.0
Genesis hash: wGHE2Pwdvd7S12BL5FaOP20EGYesN73ktiC1qzkkit8=
```

It should take about 15 minutes to update. It's got to download ~1GB data to catch up.

When `Sync Time: 0.0` your node is ready

## Install as a service

Lets make a background-service that will auto restart the node on failure and check for updates once an hour.

> [!TIP]
> Make sure to replace `***username***` with your mac username.
> This is because we have to use absolute paths in plist files
> If unsure, cd to `~/node` and type `pwd`

Back in the terminal run:

`sudo nano ~/Library/LaunchAgents/com.algorand.daemon.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.algorand.daemon</string>
        <key>ProgramArguments</key>
        <array>
            <string>/bin/bash</string>
            <string>-c</string>
            <string>echo '{"shared_server":true}' > /Users/***username***/node/data/system.json && /Users/***username***/node/algod -d /Users/***username***/node/data & sleep 2 && chmod 644 /Users/***username***/node/data/algod.net /Users/***username***/node/data/algod.token && /Users/***username***/node/kmd start -d /Users/***username***/node/data/kmd-v0.5</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <dict>
            <key>SuccessfulExit</key>
            <false/>
            <key>Crashed</key>
            <true/>
        </dict>
        <key>ThrottleInterval</key>
        <integer>5</integer>
        <key>StandardErrorPath</key>
        <string>/Users/***username***/node/servicelogs/daemon.err</string>
        <key>StandardOutPath</key>
        <string>/Users/***username***/node/servicelogs/daemon.out</string>
        <key>WorkingDirectory</key>
        <string>/Users/***username***/node</string>
        <key>UserName</key>
        <string>***username***</string>
        <key>EnvironmentVariables</key>
        <dict>
            <key>PATH</key>
            <string>/Users/***username***/node:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
        </dict>
    </dict>
</plist>


```

> [!IMPORTANT]
> This updater script is untested

and now :

`sudo nano ~/Library/LaunchAgents/com.algorand.updater.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.algorand.updater</string>
        <key>ProgramArguments</key>
        <array>
            <string>/bin/bash</string>
            <string>-c</string>
            <string>exec /Users/***username***/node/update.sh 2>> /Users/***username***/node/servicelogs/updater.err</string>
        </array>
        <key>StartInterval</key>
        <integer>3600</integer>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <dict>
            <key>Crashed</key>
            <true/>
        </dict>
        <key>ThrottleInterval</key>
        <integer>5</integer>
        <key>StandardErrorPath</key>
        <string>/Users/***username***/node/servicelogs/updater.err</string>
        <key>StandardOutPath</key>
        <string>/Users/***username***/node/servicelogs/updater.out</string>
        <key>WorkingDirectory</key>
        <string>/Users/***username***/node</string>
        <key>EnvironmentVariables</key>
        <dict>
            <key>PATH</key>
            <string>/Users/***username***/node:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
            <key>HOME</key>
            <string>/Users/***username***</string>
        </dict>
    </dict>
</plist>

```

Now load the services

```shell
launchctl load ~/Library/LaunchAgents/com.algorand.updater.plist
launchctl load ~/Library/LaunchAgents/com.algorand.daemon.plist
```

And to check that they're running (especially after login)

```shell
launchctl list | grep algorand
```

If you want to unload/disable

```shell
launchctl unload ~/Library/LaunchAgents/com.algorand.updater.plist
launchctl unload ~/Library/LaunchAgents/com.algorand.daemon.plist
```

If there are any errors, they will be put in `~/node/servicelogs`

## Useful settings

By default the node runs on port :8080. This isn't great if you do other development, so I changed it.

> [!TIP]
> Restart your service after making config changes

`sudo nano ~/node/data/config.json`

Type `control + w` to search type `EndpointAddress` then `enter` and then change endpoint address to:

```json
{
  "EndpointAddress": "127.0.0.1:4001"
}
```

Also, there are a lot of commands to remember, so I set alias's for each in my bash config (I use zsh, you may need to do this for your .bash_profile instead)

`nano ~/.zshrc`

```shell
alias algostat="goal node status"
alias algostart="goal node start"
alias algostop="goal node stop"
alias algorestart="goal node stop && goal node start"
alias algowatch="goal node status -w 1000"
alias algocatchup="goal node catchup $(curl https://algorand-catchpoints.s3.us-east-2.amazonaws.com/channel/mainnet/latest.catchpoint)"
alias algoserviceup="launchctl load ~/Library/LaunchAgents/com.algorand.updater.plist && launchctl load ~/Library/LaunchAgents/com.algorand.daemon.plist"
alias algoservicedown="launchctl unload ~/Library/LaunchAgents/com.algorand.updater.plist && launchctl unload ~/Library/LaunchAgents/com.algorand.daemon.plist"
alias algoservicerestart="algoservicedown && sleep 2 && algoserviceup && algoservice"
alias algoservice="launchctl list | grep algorand"
alias algonode="cd ~/node"
```

then restart your bash profile

`exec zsh`

## Participation Keys

This step is difficult in the terminal if you're using a ledger, so I prefer to just use [alloctrl](https://github.com/AlgoNode/alloctrl/)

With node installed, run from your `~/node` folder

`npx alloctrl`

Run through the environment steps and select your data folder. It will detect your custom port automatically.

Open the window at [http://127.0.0.1:3333](http://127.0.0.1:3333)

Connect with Defly and create participation keys.

It should be straightforward.

## Telemetry

Thanks to the kind folks over at [nodely](https://nodely.io/docs/telemetry/quickstart/) we have a way to monitor our OG nodes.

Translating their steps for MAC is as follows

`diagcfg telemetry endpoint -e https://tel.4160.nodely.io`

anonymize node name in telemery (do not leak hostname)

`diagcfg telemetry name -n "anon"`

Or you can make your name public by prefixing it with @

`diagcfg telemetry name -n "@MyPublicNodeName"`

Now you will see some output like

```shell
Telemetry logging: Name = @username, Guid = 123456789-f12f-4ff4-8f8d-a55h4tc415o, URI = https://tel.4160.nodely.io
```

Copy the GUID and visit

https://g.nodely.io/d/telemetry/node-telemetry?var-GUID=***your-GUID-here***

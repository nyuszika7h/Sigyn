# Sigyn #

A Limnoria's plugin to handle various spam and abuses with network hammer
You must install python-dnspython.

This version is being modified to work with Atheme IRC Services and InspIRCd.

## Commands ##

    addpattern <limit> <life> <pattern> : add a permanent pattern, triggers kill & kline if called more than <limit> during <life> in seconds, use 0 as limit for immediate action
    addregexpattern <limit> <life> <pattern> : add a permanent regular expression /pattern/, triggers kill & kline if called more than <limit> during <life> in seconds, use 0 as limit for immediate action
    lspattern [--deep] <id|pattern> : search patterns inside database, use pattern's id for full information, and --deep to search on disabled pattern
    editpattern <id> <limit> <life> [<comment>] : change some values of a pattern and add a optional comment
    togglepattern <id> <boolean> : enable or disable a pattern
    checkpattern <text> : checks text against permanent patterns
    lstmp <channel> : list computed pattern for a given <channel>
    rmtmp <channel> : remove computed pattern for a given <channel>
    addtmp <channel> <text> : add a temporary pattern for a given <channel>
    addglobaltmp <text> : add a temporary pattern to all channels
    defcon [<channel>]: put the bot in a agressive mode where limits are lowered and ignores are lifted
    oper : tell the bot to oper
    state [<channel>] : debug informations about internal state, for the whole ircd or a channel
    vacuum : optimize the database
    resync : synchronize user's presence ( after a plugin reload )

## Behaviour ##

Sigyn is coming from https://en.wikipedia.org/wiki/Sigyn

The plugin works with TimeoutQueue ( https://github.com/ProgVal/Limnoria/blob/master/src/utils/structures.py#L308 ) : 
Queues are filled on abuses, when the length of a queue exceeds limits, kill and kline are triggered.

Limits of queues may be lowered under conditions ( same abuse repeated by various users in a channel, network abuse, etc )

Sigyn is able to detect and extract spam pattern, and use them as lethal pattern of a period of time.

After being correctly configured, Sigyn can handle attacks without human hands, most settings can be tweaked per channel.

Default values in config.py must be modified to fits your needs.

## Configuration ##

You should take a look at tips here :

https://github.com/ncoevoet/ChanTracker#other-tips

### General ###

First measure:

    defaultcapability remove protected

User with #channel,protected capability are ignored, so you must remove from all users this capability which is given by default.

You should tell to bot to logs his actions in a channel:
    
    config supybot.plugins.Sigyn.logChannel #network-secret-channel

You could tell it to use notice on that channel:

    config supybot.plugins.Sigyn.useNotice True
    
In order to resolve hosts, python-dnspython is used, you can change the max duration of a resolve here:

    config supybot.plugins.Sigyn.resolverTimeout 3
    
You must set operatorNick and operatorPassword, bot will try to oper if both filled.
    
    config supybot.plugins.Sigyn.operatorNick
    config supybot.plugins.Sigyn.operatorPassword
    
On some detections bot will never take actions but will alert in logChannel, you can define the interval between two alert for the same problem:

    config supybot.plugins.Sigyn.alertPeriod 900
    
Lags and netsplits could have lot of impact on some protections ( flood, low flood detections ) due to burst of messages received at same time.
This is why the bot do a remote server request at regular interval to check the quality of the network.
If lags is too high, some protections are disabled ( note for your need you will probably need to change few bits on plugin.py ( do017, do015 ))

    config help supybot.plugins.Sigyn.lagInterval 
    config help supybot.plugins.Sigyn.lagPermit
    config help supybot.plugins.Sigyn.netsplitDuration

You should also change supybot.plugins.Sigyn.lagInterval to some minutes at least as it's also used for internal state cleanup.

As bot is opered, it can look at server's notices.

So it can alert in logChannel when some kline affects more than x users ( with some limitations due to server's notices & cloaks )

    config help supybot.plugins.sigyn.alertOnWideKline

To enable kline and kill on abuse, you must enable it and set a klineDuration > 0, before doing that, you should test and tweaks detection settings :

    config supybot.plugins.Sigyn.enable True --> otherwise Sigyn will only announce them in logChannel
    config supybot.plugins.Sigyn.klineDuration --> in minutes

### Detections ###

All those settings can be either global or customized per channel :

    config something
    config channel #mychannel something

#### Protections ####

To prevent Sigyn to monitor a particular channel:

    config channel #mychannel supybot.plugins.Sigyn.ignoreChannel True
    
To prevent Sigyn to monitor a particular user in a channel:

    register useraccount password
    admin capability add useraccount #mychannel,protected
    hostmask add useraccount *!*@something
    
You can also exempt an user for some protections :

    register useraccount password
    admin capability add useraccount #mychannel,-flood
    admin capability add useraccount #mychannel,-ctcp
    admin capability add useraccount #mychannel,-lowFlood
    hostmask add useraccount *!*@something

List of anti-capabilities you can use: flood, lowFlood, broken, ctcp, repeat, lowRepeat, massRepeat, lowMassRepeat, nick, hilight, cycle

You can tell Sigyn to be more laxist against someone who is in the channel long time enough:

    config supybot.plugins.Sigyn.ignoreDuration
    config channel #mychannel supybot.plugins.Sigyn.ignoreDuration

But as everything can happen ... 

    config help supybot.plugins.Sigyn.bypassIgnorePermit
    config help supybot.plugins.Sigyn.bypassIgnoreLife

Few are detailed here, take a look at config.py for the full list.

#### Flood ####

For most detections you have to deal with 2 or 3 values at least, let's see with flood detection.

    config supybot.plugins.Sigyn.floodPermit <max number of message allowed>
    config supybot.plugins.Sigyn.floodLife <during x seconds>
    config help supybot.plugins.Sigyn.floodMinimum ( empty messages and digit messages bypass the minimun )
    
    config channel #mychannel supybot.plugins.Sigyn.floodPermit
    config channel #mychannel supybot.plugins.Sigyn.floodLife
    
Because some irc clients throttles messages, there is another set of flood detection you could use:

    config supybot.plugins.Sigyn.lowFloodPermit <max number of message allowed>
    config supybot.plugins.Sigyn.lowFloodLife <during x seconds>

#### Repeat ####

Sigyn can create temporary lethal pattern, there is two way to create pattern, one from a single message where pattern are repeated and the other from multiples messages from various client.

For minimum computed pattern length is defined here :

    config supybot.plugins.Sigyn.computedPattern
    
Those patterns will stay active during (in seconds) :

    config supybot.plugins.Sigyn.computedPatternLife

Note : each time a pattern is triggered it remains active for 'computedPatternLife' seconds again

##### Single user repeat #####

Let's see the easy case, someone alone repeat same message over and over.

    config supybot.plugins.Sigyn.repeatPermit
    config supybot.plugins.Sigyn.repeatLife
    config supybot.plugins.Sigyn.repeatPercent ( 1.00 will never work, because similarity must be > at repeatPercent )

If the user raise 2/3 of the limit, bot will try to compute pattern, with this pattern creation settings.
    
    config supybot.plugins.Sigyn.repeatCount <number of occurences of the pattern in a message>
    config supybot.plugins.Sigyn.repeatPattern <minimal length of the pattern, if it occurs more than repeatCount but it is still smaller than computedPattern>

##### Repeat from various users #####

    config supybot.plugins.Sigyn.massRepeatPermit
    config supybot.plugins.Sigyn.massRepeatLife
    config supybot.plugins.Sigyn.massRepeatPercent 
    config help supybot.plugins.Sigyn.massRepeatMinimum  (1.00 will never work, because similarity must be > at massRepeatPercent )

The main difference between both : single repeat detection could create small pattern ( so dangerous for legit users ) while massRepeat try to make the largest possible pattern. 

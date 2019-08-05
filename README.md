# GetTickCount-as-alternative-for-timers
Southclaw’s Guide to the ‘tickcount()’ Function, and how it can be used as an alternative for unnecessary timers!


# Using Timers
Say for example you have a specific action that can only be performed once every so many seconds, I see a lot of people (including Southclaw a year ago) doing something like this:
```pawn
new bool:IsPlayerAllowed[MAX_PLAYERS];

OnPlayerInteractWithServer(playerid)
/* This can be any sort of input event a player makes such as:
 *  Entering a command
 *  Picking up a pickup
 *  Entering a checkpoint
 *  Pressing a button
 *  Entering an area
 *  Using a dialog
 */
{
    if(IsPlayerAllowed[playerid]) // This only works when the player is allowed to
    {
        DoTheThingThePlayerRequested();
        IsPlayerAllowed[playerid] = false; // Disallow the player
        SetTimerEx("AllowPlayer", 10000, false, "d", playerid);

        return 1;
    }
    else
    {
        SendClientMessage(playerid, -1, "You are not allowed to do that yet!");

        return 0;
    }
}

public AllowPlayer(playerid)
{
    IsPlayerAllowed[playerid] = true;
    SendClientMessage(playerid, -1, "You are allowed to do that thing you did 10 seconds ago again now.");
}
```
Now this is all well and good, it works, the player won’t be able to do that thing again for 10 seconds after he uses it.

Take another example here, this is a stopwatch that measures how long it takes for a player to do a simple point to point race:

```pawn
new
    gPlayerStopWatchTimerID[MAX_PLAYERS],
    gPlayerStopWatchTime[MAX_PLAYERS];

StartPlayerRace(playerid)
{
    gPlayerStopWatchTimerID[playerid] = SetTimerEx("StopWatch", 1000, true, "d", playerid);
}

public StopWatch(playerid)
{
    gPlayerStopWatchTime[playerid]++;
}

OnPlayerFinishRace(playerid)
{
    new str[128];

    format(str, 128, "You took %d seconds to do that", gPlayerStopWatchTime[playerid]);
    SendClientMessage(playerid, -1, str);

    KillTimer(gPlayerStopWatchTimerID[playerid]);
}
```

These two examples are simple, but that’s the general idea I see a lot in filterscripts, people’s examples, even released gamemodes. There is a much better way of doing it, which is more way accurate and can give stopwatch timings down to the millisecond, which is perfect for racing servers!

# Using GetTickCount

GetTickCount() is a function that gives you the time in milliseconds since the server machine started up, as in the actual computer system that you’re running the server process on.

If you call this function at two different times, and subtract the first time from the second you suddenly have an interval between those two events in milliseconds! Take a look at this example:
```pawn
new gPlayerAllowedTick[MAX_PLAYERS];

OnPlayerInteractWithServer(playerid)
{
    if(tickcount() - gPlayerAllowedTick[playerid] > 10000)
    // This only works when the current tick minus the last tick is above 10000.
    // In other words, it only works when the interval between the actions is over 10 seconds.
    {
        DoTheThingThePlayerRequested();
        gPlayerAllowedTick[playerid] = tickcount(); // Update the tick count with the latest time.

        return 1;
    }
    else
    {
        SendClientMessage(playerid, -1, "You are not allowed to do that yet!");

        return 0;
    }
}
```

There’s a lot less code there, no need for a public function or a timer. If you really want to, you can put the remaining time in the error message:

(I’m using SendFormatMessage, which isn’t a native function, in this example)

```pawn
SendFormatMessage(playerid, -1, "You are not allowed to do that yet! You can again in %d ms", 10000 - (tickcount() - gPlayerAllowedTick[playerid]));
```

That’s a very basic example, it would be better to convert that MS value into a string of “minuteseconds.milliseconds” but I’ll post that code at the end.

Hopefully you can see how powerful this is to get intervals between events, let’s look at another example

```pawn
new
    gPlayerStopWatchTick[MAX_PLAYERS];

StartPlayerRace(playerid)
{
    gPlayerStopWatchTick[playerid] = tickcount();
}

OnPlayerFinishRace(playerid)
{
    new
        interval,
        str[128];

    interval = tickcount() - gPlayerStopWatchTick[playerid];

    format(str, 128, "You took %d milliseconds to do that", interval);
    SendClientMessage(playerid, -1, str);
}
```

In this example, the tick count is saved to the player variable when he starts the race. When he finishes it, the current tick (of when he finished) has that initial tick (The smaller value) subtracted from it and thus leaves us with the amount of milliseconds in between the start and the end of the race.

# Breakdown
Now lets break the code down a bit.
```pawn
new
    gPlayerStopWatchTick[MAX_PLAYERS];
 ```

This is a global variable, we need to use this so we can save the tick-count and retrieve the value at another point in time (in other words, use it in another function, later on)
```pawn
StartPlayerRace(playerid)
{
    gPlayerStopWatchTick[playerid] = tickcount();
}
```

This is when the player starts the race, the tick count of now is recorded, if this happens is 1 minute after the server started, the value of that variable will be 60,000 because it is 60 seconds and each second has a thousand milliseconds.
Okay, we now have that player’s variable set at 60,000, now he finishes the race 1 minute 40 seconds later:


```pawn
OnPlayerFinishRace(playerid)
{
    new
        interval,
        str[128];

    interval = tickcount() - gPlayerStopWatchTick[playerid];

    format(str, 128, "You took %d milliseconds to do that", interval);
    SendClientMessage(playerid, -1, str);
}
```

Here is where the calculation of the interval happens, well, I say calculation, it’s just subtracting two values!

tickcount() returns the current tick count, so it will be bigger than the initial tick count which means you subtract the initialtick count from the current tick count to get your interval between the two measures.

So, as we said the player finishes the race 1 minute and 40 seconds later (100 seconds, or 100,000 milliseconds), tickcount will return 160,000.
Subtract the initial value (Which is 60,000) from the new value (Which is 160,000) and you get 100,000 milliseconds, which is 1 minute 40 seconds, which is the time it took the player to do the race!

***NOTE: If you use a time machine of any sort or possess the ability to manipulate the passage of the time space continuum, this can cause major problems with systems that rely on GetTickCount().***


# Recap + Notes
So! We learned that:

tickcount returns the amount of time in milliseconds since the computer system that the server is running on started.
And we can use that by calling it at two intervals, saving the first to a variable and comparing the two values can give you an accurate interval in milliseconds between those two events.
Time travel is not safe and probably requires another include library that supports it.
Last of all, you don’t want to be telling your players time values in milliseconds! What if they take an hour to complete a race?

It’s best to use a function that takes the milliseconds and converts it to a readable format, for instance, the earlier example the player took 100,000 milliseconds to do the race, if you told the player he took that long, it would take longer to read that 100,000 and figure out what it means in human-readable time.


[So what we can do is use a nice simple function that splits up the milliseconds into “hours:minutes:seconds.milliseconds”, click here to go to my post about the MsToString function!](https://southclaws.wordpress.com/2013/08/18/mstostring-snippet/)

You can use this function like this:
```pawn
OnPlayerFinishRace(playerid)
{
    new
        interval,
        str[128];

    interval = tickcount() - gPlayerStopWatchTick[playerid];

    format(str, 128, "You took %s to finish the race", MsToString(interval, "%1m:%1s.%1d"));
    SendClientMessage(playerid, -1, str);
}
```

For example, if you did a race in 34,583 milliseconds you would see:

1. “You took 00:34.583 to do that”

But if you did it in 465,583 milliseconds it would say:
1. “You took 07:45.583 to do that”

I hope this helped! I wrote it because I’ve helped a few people out recently who didn’t know how to use GetTickCount() as an alternative for timers or for getting intervals etc.

If I missed anything out or made a mistake let me know and I will fix it! And if there’s anything you think I should add to the guide, also let me know.

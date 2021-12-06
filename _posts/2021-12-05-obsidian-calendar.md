---
layout: post
title: "Import Google Calendar agenda to your Obsidian daily note"
date: 2021-12-05
---
I came across [this blog post](https://benenewton.medium.com/how-i-automatically-import-my-meetings-into-my-daily-note-template-3d9537b17dea) by Ben Newton recently where they explained how they were able to add their Outlook calendar agenda to their daily note in Obsidian. At my current company we use Google Calendar so I wanted to see if something similar could be achieved with that, and discovered that this is indeed possible. 

#### gcalcli
The [galcli tool](https://github.com/insanum/gcalcli) is the bit that makes this whole thing work. The official setup instructions for gcalcli are a little out of date and not fully complete, so I recommend following [this guide](https://github.com/aristosv/google_auth) for the Google API setup part. Once you've got your Client ID and Secret you can either pass them directly to gcalcli or set them in the gcalclirc file. 

After authenticating gcalcli you can try to run `gcalcli agenda` to see what it returns. The default output is a little too highly formatted for it to be useful when pulled into Obsidian so I chose to go with the tab-separated output by appending the `--tsv` option to the end of my command. There also isn't any built-in option to just return the current days agenda so we need to use the `date` command in the terminal to figure that out. In the end my gcalcli command looks like this (Note: I am on macOS, if you want to do this on Linux you'll need to change the `date` command options):

```
gcalcli --lineart ascii  --nocolor agenda --details end $(date '+%A') $(date -v+1d "+%A") --tsv
```

```
2021-12-06      10:30   2021-12-06      11:30   DNB
2021-12-06      11:30   2021-12-06      12:30   Lunch
2021-12-06      13:00   2021-12-06      15:00   Focus time
```

This is closer but I really don't need the date in there as that will be implied from the date of my note. To remove these two columns I installed [tsv-utils](https://github.com/eBay/tsv-utils#obtaining-and-installation). I only specifically need the `tsv-select`  binary so after downloading those tools I just popped that into `/usr/local/bin` so it was in my $PATH. With `tsv-select` installed I can pipe the output from `gcalcli` to it and remove the unwanted columns:

```
gcalcli --lineart ascii  --nocolor agenda --details end $(date '+%A') $(date -v+1d "+%A") --tsv | tsv-select --exclude 1,3
```

```
10:30   11:30   DNB
11:30   12:30   Lunch
13:00   15:00   Focus time
```

#### Obsidian and Templater
Now that we have a way to programmatically get our Google Calendar agenda from the command line - we can use the same technique that Ben used in the post that inspired this one. You will need to install the [Templater plugin](https://github.com/SilentVoid13/Templater) for Obsidian. Once installed and enabled, go to its settings and scroll down to the User System Command Functions section. Enable it and add a new function called `dailyAgenda` with the following command:

```shell
/opt/homebrew/bin/gcalcli --lineart ascii  --nocolor agenda --details end $(date '+%A') $(date -v+1d "+%A") --tsv | /usr/local/bin/tsv-select --exclude 1,3
```

![Templater Configuration](/images/obsidian_templater.png)

Finally, just add the following to your daily note template:

```
### Agenda
<% tp.user.dailyAgenda() %>
```

Now when you use Templater to generate a new daily note - your Google Calendar agenda for that day will be imported as well. 

![Daily Agenda](/images/obsidian_calendar.png)

Thanks to [Ben Newton's blog](https://benenewton.medium.com) for the idea to begin with.

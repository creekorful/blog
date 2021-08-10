+++ 
title = "Parakeet: an IRC log renderer"
date = "2021-08-09"
author = "Aloïs Micard"
authorTwitter = "" #do not include @ cover = ""
tags = ["My Projects", "Golang"]
keywords = ["", ""]
description = "Generate beautiful HTML static files from IRC logs."
showFullContent = false 
+++

I have started using a lot IRC lately since I became a [Debian Developer](https://wiki.debian.org/DebianDeveloper) (Debian
use a lot IRC and mailing lists to communicate). 

For those who don't know what IRC is: it's basically the ancestor of modern chat messaging
like [Discord](https://discord.com/). IRC is a quite old piece of technology, coming from 1988.

One big drawbacks of IRC is that there's no such thing as a history, you need to be connected to receive the messages, 
everything that happens when your offline will be missed.

Because of such limitation, people started using [Bouncer](https://en.wikipedia.org/wiki/BNC_(software)#IRC) or self-hosted
IRC client such as [The lounge](https://thelounge.chat/). These clients are always connected to the IRC channels in order
to allow the user to read the messages sent while he is away.

I'm personally using a self-hosted **The lounge** on my dedicated server. The lounge is storing the logs history on the
filesystem under `/var/opt/thelounge/logs/{username}/{server}/*.log` files.

The log files looks like this:

```
2021-03-29T13:16:36.853Z] *** creekorful (~creekorful@0002a854.user.oftc.net) joined
[2021-03-29T14:09:34.402Z] *** fvcr (~francisco@2002c280.user.oftc.net) quit (Server closed connection)
[2021-03-29T14:09:45.237Z] *** fvcr (~francisco@2002c280.user.oftc.net) joined
[2021-03-29T15:07:10.843Z] *** Maxi[m] (~m189934ma@00027b5d.user.oftc.net) quit (Server closed connection)
[2021-03-29T15:07:15.415Z] *** Maxi[m] (~m189934ma@00027b5d.user.oftc.net) joined
[2021-03-31T11:44:01.184Z] <b342> If a pipeline in my namespace failed to build, does it mean it that only me get the e-mail for the failed build or the project I forked from too ?
[2021-03-31T11:53:03.597Z] <Myon> should be only you
[2021-03-31T11:53:49.045Z] <Myon> plus I think it's only the person triggering the pipeline, not the whole project
[2021-03-31T12:22:27.901Z] *** sergiodj (~sergiodj@00014bc9.user.oftc.net) quit (Server closed connection)
[2021-03-31T12:23:42.039Z] *** sergiodj (~sergiodj@00014bc9.user.oftc.net) joined
[2021-03-31T14:35:02.708Z] <b342> Thanks Myon
```

I was wondering: *maybe a could render these log file into something more visual?*

# Parakeet

I built [Parakeet](https://github.com/creekorful/parakeet) (in Golang) to solve this issue.
Using it is a simple as: `./parakeet -input irc-log.log -output irc-log.html`

## How does it work?

### Parsing the log file

The tool first parse the provided log file, and extract the messages from it, while excluding all login / logout events.

```go
for {
    line, err = rd.ReadString('\n')
    if err != nil {
        break
    }

    line = strings.TrimSuffix(line, "\n")

    // Only keep 'message' line i.e which contains something like '] <username>'
    if !strings.Contains(line, "] <") || !strings.Contains(line, ">") {
        continue
    }

    // Approximate line parsing
    date := line[1:strings.Index(line, "] <")]
    username := line[strings.Index(line, "<")+1 : strings.Index(line, ">")]
    content := line[strings.Index(line, "> ")+2:]

    t, err := time.Parse(time.RFC3339, date)
    if err != nil {
        break
    }

    ch.Messages = append(ch.Messages, Message{
        Time:    t,
        Sender:  username,
        Content: content,
    })
}
```

### Rendering the messages

Thanks to the go [html/template](https://pkg.go.dev/html/template) package, it's fairly easy to generate HTML.
The template file looks like this:

```
<html lang="en">
<head>
    <title>{{ .Name }}</title>
    <meta charset="UTF-8">
</head>
<body>
<table>
    {{ range .Messages }}
        <div>
            [{{ .Time.Format "2006-01-02 15:04:05" }}] {{ .Sender | colorUsername }} {{ .Content | applyUrl }}
        </div>
    {{ end }}
</table>
</body>
</html>
```

#### Assign color to the username

Here's a little trick I've done to assign a color to each username, making the log files easier to read.

```go
type Context struct {
	Colors []string
	Users  map[string]string
}

func (c *Context) colorUsername(s string) template.HTML {
	// check if username color is not yet applied
	color, exist := c.Users[s]
	if !exist {
		// pick-up new random color for the username
		color = c.Colors[rand.Intn(len(c.Colors)-1)]
		c.Users[s] = color
	}

	return template.HTML(fmt.Sprintf("&lt;<span style=\"color: %s; font-weight: bold;\">%s</span>&gt;", color, s))
}
```

#### Make links clickable

And here's how I've made the links clickable, by making them HTML anchor.
I've used [github.com/mvdan/xurls](https://github.com/mvdan/xurls) to find the URLs.

```go
func (c *Context) applyURL(s string) template.HTML {
	rxStrict := xurls.Strict()
	return template.HTML(rxStrict.ReplaceAllStringFunc(s, func(s string) string {
		return fmt.Sprintf("<a href=\"%s\">%s</a>", s, s)
	}))
}
```

# Conclusion

I'm using this tool to mirror the logs for the channel I'm using. 
The results are available [here](https://irc-logs.creekorful.org/) and updated daily.

Happy hacking!
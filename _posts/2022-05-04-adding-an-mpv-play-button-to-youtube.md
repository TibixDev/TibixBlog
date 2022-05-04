---
title: "How I added an mpv play button to YouTube"
date: 2022-05-04
---

# 0x01 Intro

Hello world!
So, this is my first blog post. The style, structure, look, and overall feel of everything is a work in progress. Expect things to change. Also, if you have any feedback please feel free to email me at `fuloptibi03@gmail.com`.

Oftentimes I find myself copying video URL-s into my terminal to play them using mpv. mpv is basically a really lightweight cross-platform media player. I use it whenever I don't want to leave my browser open because I am doing some intensive task where I can really use the spare RAM and CPU. I also use it when I really want to highlight something amongst my *70+ open tabs*, but it is also my go-to player for movies and anime, you should seriously give it a try.

This task of copying video URL-s has become extremely repetitive over the months I've been doing it, so I decided to combat this problem by trying to do something useful with my passion for programming. Thus, I decided I'd add an mpv play button to YouTube.

# 0x02 How?

Actually, this was my first question when I started thinking about this problem: "*how?*". The only solution I could come up with for having some interaction between the browser and the "outside apps" was to make a custom protocol handler/URI scheme. The notion "custom protocol handler" might sound a bit intimidating at first, but it's actually quite simple when we break it down. So, your usual link starts with `https://`, which standards for `Hypertext Transfer Protocol Secure`. Notice how it has the keyword "protocol" in it? The HTTPS protocol is automatically opened and handled in your browser, as expected. Well, it turns out there are many other protocols, which all serve different purposes. For example, the `mailto://address@mail.com` protocol as you could've guessed opens up your mail client with the email address `address@mail.com` in the receiver field, or `calc://` which opens up the calculator in Windows. What's really cool is that you can also register your own protocols that have their own logic. This is exactly what we're going to leverage here to get our browser to open up mpv. If you feel like you would like to read up more on this topic, you can read [this](https://ldapwiki.com/wiki/Custom%20URI%20scheme) article.

As a fun fact, the first time I did a deep-dive into protocols was when I was doing penetration testing for a random game back in 2019. It turns out protocols are not only useful for the users, but also for the attackers. I've stumbled upon many interesting writeups back then, such as this [Origin Remote Code Execution Vulnerability](https://zero.lol/posts/2019-05-22-fun-with-uri-handlers/). I think that this topic in in of itself is so long and intriguing that it could use a separate blog post, so in the future I **might** publish a separate blog post about protocol handler vulnerabilities.

# 0x03 How a protocol-handler-sandwich is made

So, we want to register our own protocol handler. Well, how do we do that? This is the part where it actually gets complicated. Sadly there's no universal method. Windows, Mac, and Linux all have different ways of registering protocol handlers. In this blog post we'll be focusing on the Linux side of things since that's what I'm using on a daily basis, but it's not impossible that at a later date Windows will be included in this blog post as a late update, or at least in the [repository](https://github.com/TibixDev/MPVPlay) of the project if someone (or me) gets enough motivation to work on it.

Registering custom protocol handlers in Linux is actually not that complicated if your distribution uses `xdg-open` for handling them. Mine (Manjaro) does, and so should most other popular distros, so ideally this will work for most people. You need to make a `.desktop` file under `~/.local/share/applications/`. It's interesting, but these `.desktop` files are also used like traditional shortcuts in Linux, so your protocol handler be visible under apps by default.

You need to create a file like so:
```
[Desktop Entry]
Type=Application
Name=Example Scheme Handler
Exec=example.sh %u
StartupNotify=false
MimeType=x-scheme-handler/example;
```

Now that you've made this file you need to register it for it to be able to handle calls, which you can do this using the following command: `xdg-mime default example-opener.desktop x-scheme-handler/example`. You should also run `update-desktop-database`, though theoretically it should happen automatically. This will bind the `example-opener.desktop` file to the `example://` protocol. From now on if everything went well, whenever you open a `example://argument-here` link, the protocol handler with launch `example.sh` (**from the same directory, the executor should exist there**) with the arguments provided. 

I used this exact method to make my custom protocol handler, with some small modifications such as adding an Icon, but other than this it's exactly the same.

# 0x04 Obscure bugs always lurk in the darkness

We've arrived at the point where we need to implement the logic of our protocol hander. This is where things went south for me, *really* quickly. So, the argument can be easily obtained with `%u`. I thought I'd have an extremely easy task next. I'd just use `mpv %u` as the `Exec` parameter, mpv and yt-dlp should do the rest automatically and I'd have videos playing in mpv in no time. Well, that's where I was very wrong. What I'm about to cover here only spans a few paragraphs, however it took me multiple days to figure out the issues that followed. And trust me, there were many.

First off, `mpv %u` did nothing. I eventually figured out, that the argument supplied by Chromium was `mpv://URL`, not just simply `URL`. This breaks everything, obviously, so I had to change it. This is where I started doing useless stuff that ate up alot of time. Instead executing mpv directly, I executed Python. I took the argument, stripped the `mpv://` part, and launched mpv from Python, but strangely nothing happened. I even added all kinds of quotes and safeguards to make sure everything was in place. The script was absurdly long at this point, I logged every single step to make sure it was correct, yet still nothing happened. I even logged the execution into files and they were either empty or just contained the correct command and arguments, both in Bash and Python. In the end I decided to abandon Python alltogether and concluded that perhaps protocol handlers have some kind of limitation on execution, because almost nothing I'd try to launch worked. Mpv, Cellulose, VLC, nothing seemed to launch. If I typed in the exact same command as the one which got executed in the terminal myself, it worked just fine. For a while I had a hunch that perhaps `xdg-open` limits what I can execute, but in the end I saw that some programs still launched, such as the terminal from that very same `Exec` parameter. This was strange.

After alot of thinking, I had a brilliant idea: if I couldn't see anything appearing, why don't I directly make the terminal appear as a separate process with that command, this way at least I can see what the error is? I did just that. I launched Konsole with the exact same command as before, and I briefly got two windows. One with mpv "opening" disappeared, and the Konsole window that remained. I got really curious. If there's a sandbox, this Konsole window should be sandboxed, so I should see if something fails to launch like previously. I typed in `mpv` and of course nothing appeared, **but** this time I saw something I've never seen before: `mpv: symbol lookup error: mpv: undefined symbol: vkCreateWaylandSurfaceKHR`. This was even more bizarre than the sandboxing idea.

How come some random Vulkan call fails when it works fine in my other terminal opened from KDE? The error had almost no online footprint, until I found [this](https://www.reddit.com/r/mpv/comments/ry9sgi/comment/hs3o3pw/?utm_source=share&utm_medium=web2x&context=3) Reddit comment. Strange, the poster tries to open a video from Downloads and gets the same error. Funnily enough just when I was making this blog post, I also found [this](https://ttys3.dev/post/archlinux-mpv-undefined-symbol-vkcreatewaylandsurfacekhr/) other blog post detailing the exact same issue I was having but in Chinese but also providing a perfect solution (*Where were you in my search results when I was suffering for hours?*). So where does `vkCreateWaylandSurfaceKHR` come from? Well, I don't exactly remember the process of how I found this out, but apparently it comes from a shared library called `libvulkan.so`. It works fine in a regular terminal, so that means my `libvulkan.so` isn't broken, and my GPU is also able to support it.

And here is when I just went 🤨. Could it be that Chromium provides its own `libvulkan.so`? If so, we need to use the one provided by the system and that should make this problem go away. And that's precisely what happened. Chromium ships a broken copy of `libvulkan.so` without symbols and we need to provide our own. Linux has a nifty feature for this called `LD_PRELOAD` where we can just preload our own libraries. So, to make this problem go away we just need to prepend our command with `export LD_PRELOAD=/usr/lib/libvulkan.so.1;`. **Aaand it worked like magic!** MPV opens and the video plays just fine with the correct URL provided. By the way, if you are wondering how you can test protocol handlers without fiddling in the browser you can just use `xdg-open 'example://parameter`.

At this point I also decided that I wanted the entire `Exec` to be a single line of Bash without involving Python or Node or anything alike, so I also needed a way to strip the initial `mpv://` substring. Luckily I got a bunch of help from StackOverflow and also my [friend](https://github.com/Levev) who helped me massively with debugging and fixing this entire mess (props to him).

So the final `Exec` command at that time was, *lo and behold*: `Exec=sh -c 'export LD_PRELOAD=/usr/lib/libvulkan.so.1; konsole -e /bin/bash --rcfile <(arg=%u; mpv "https:${arg:11}")'`. Amazing, isn't it? The `sh -c` is needed because for some reason nothing appears or executes without it. `konsole -e binary --rcfile args` is just executing a binary with the specified `args`, in this case `/bin/bash` which is the Bash interpreter. The latter part simply explained takes the string `https:` and adds the argument to it, but without the first 11 characters. which contain the `mpv://` part, and the URL. I will explain why I added the `https:` part myself later. This command already looks terrible, but it's about to get alot worse, so buckle up. 

# 0x05 There's always that other browser

Now we'll skip a bit ahead in time to a point where the project was done, but nobody had tested it in Firefox. We skip here because this is the last part that still relates to Bash and I feel like it'll flow better. I tested the protocol handler in Firefox because some of my friends use that, aaaand it didn't work. Time to investigate!

I first learned that different browsers submit URL arguments differently, some use `https//`, others use `https//:`. MPV won't accept both, only the latter one, so I had to deal with that because Chromium prefers the first variant. You can see traces of this in the previous command above. With Chromium it was easy enough, I just took the part starting with `//` which comes after the 11-th character, and manually prepended it the string `https:`. Well, Firefox was different. It had the correct `https://` by default, so modifying the string was unnecessary. On the other hand though, Firefox had quotes around the entire argument which Chromium didn't. I needed to account for both in one line of Bash.

In the end, this is the final command: `bash -c 'export LD_PRELOAD=/usr/lib/libvulkan.so.1; arg=$0; arg=${arg/"https//"/"https://"}; arg=$(echo $arg | tr -d "'"'"'"); mpv "${arg:6}"' "%u"`. It's similar in some ways to the previous one. We no longer execute Konsole since it's unnecessary, we can just execute the Bash interpreter directly. Again, we preload the Vulkan library because we need to, then comes this part: `arg=${arg/"https//"/"https://"};`. This just replaces `https//` with `https://` when it can, otherwise does nothing. Pretty weird syntax but it works. This is much cleaner than my previous approach and will always work. The next part (`arg=$(echo $arg | tr -d "'"'"'");`) removes the single quotes around a string. See the issue is that the `Exec` already includes a single quote, so including another one before the end would break the command. The way to *escape* a single quote in Bash is actually quite crazy: `"'"'"'"`. Yes, **this** abomination is how you escape a single quote. Thanks to that one person on StackOverflow who came up with this, I sadly can't find the post anymore. The last part is not really different from our previous attempt, it's just that now we remove less characters because we only need `mpv://` removed. The last `%u` part only supplies the argument to Bash. Aaaand, that's it. We have a fully functioning YouTube Video URL opener protocol which opens mpv handler. That was a mouthful.

# 0x06 The button

Now that I had a perfectly working protocol handler, it was time to work on actual button which was originally mentioned in the article title aswell. *(It took a while to get to this point didn't?)*

I concluded that the best way to get a button to consistently to appear was by using a userscript. An addon would've worked too but I felt like it was unnecessary. So, what's a userscript? A userscript is basically exactly what it says it is, a script by made users. It injects into any site to add extra functionality or adjust things. This was exactly what I needed. To run userscripts you need a userscript addon, after trying a bunch of them, my choice ended up being [ViolentMonkey](https://violentmonkey.github.io/) since it's open source and works with most major browsers.

A userscript is exactly like an ordinary JS file except it has a special header. I think this is way easier to understand if I give you an example, so here's the header I did for my userscript:

```js
// ==UserScript==
// @name         YouTube MPV Player
// @version      0.1
// @description  This little script opens any YouTube video in MPV with a simple button click
// @author       TibixDev
// @match        https://www.youtube.com/*
// @icon         https://www.google.com/s2/favicons?sz=64&domain=youtube.com
// @grant        GM_addStyle
// ==/UserScript==
```

As you can see it has some metadata but also fields like `match` which specify where the script should activate. As for me this was on YouTube, since I wanted it to activate even if someone didn't start directly with a video URL but normal YouTube. `@grant` is just there for legacy reasons but I might remove it later if I test it with more userscript extensions and they work without it.

So now with this done I just needed to write some ordinary JS. Pacing will get faster here because the length of the code also significantly increases, but it's just DOM operations which you probably know the basics of if you're reading this. Significant portions of the code will still be explained. I decided to place the button in the row where the dislike and like buttons are, because it seemed like there was enough space there and it wouldn't look too weird. To find this specific container we need to search the DOM elements based on the classes, ID, and children they might have. YouTube has an extremely convoluted list of classes and *many* elements that share the exact same class tree. You might come screaming at me to just use the ID, but believe it or not, those also have duplications. How? I don't know. These classlists, elements, and ID-s also change after some loading, and you need to wait before adding any elements because otherwise they might end up in the *abyss*. Initially only the row appears with the space, so I also had to make a timeout to wait for the load to complete. Have I mentioned that the layout differents on an element-level *just from having a different screen-size*? Well, now you know. It's kind of a nightmare.

The query selector I ended up with for the just **detecting** the button container is the following: `#info>#menu-container>#menu>ytd-menu-renderer>#top-level-buttons-computed>ytd-toggle-button-renderer`. This is different from the button container query selector **for placing** our button which is `#info>#menu-container>#menu>ytd-menu-renderer>#top-level-buttons-computed`. I wish I was joking but I am not. So why are these two different? Well, the first one has an extra `>ytd-toggle-button-renderer` because otherwise it'd find the container way before the rendering is finished, and then elements would end up in the shadow realm. You can still see them in the DOM tree but they are nowhere to be seen. The `ytd-toggle-button-renderer` class is only present in an element if the rendering is finished. It's a cheap trick but it works surprisingly well.

After this all that remains is toggling the aforementioned detection if the url contains `v=` which is for video URL-s, there's an automated interval where this automatically happens and stops if a button is already present. We just create a stylized button element and append it to our container. The mpv URI is also really easy to assemble: `"mpv://" + document.location.href`. With those out of the way, we are ready to add the button! Here's the final code:

```js
// ==UserScript==
// @name         YouTube MPV Player
// @version      0.1
// @description  This little script opens any YouTube video in MPV with a simple button click
// @author       TibixDev
// @match        https://www.youtube.com/*
// @icon         https://www.google.com/s2/favicons?sz=64&domain=youtube.com
// @grant        GM_addStyle
// ==/UserScript==

(async function () {
    'use strict';
    console.log("[YTMPV] YouTube MPV player script loaded");

    function setButtonInterval() {
        return setInterval(() => {
            if (document.querySelector("#info>#menu-container>#menu>ytd-menu-renderer>#top-level-buttons-computed>ytd-toggle-button-renderer")) {
                console.log("[YTMPV] Menu container found, executing...");
                addMpvButton()
            }
        }, 1000);
    }

    let waitForButtons = null;

    let location = window.location.href;;
    if (location.match(/^https:\/\/www\.youtube\.com\/watch\?v=([^&]*)/)) {
        waitForButtons = setButtonInterval();
    }

    let waitForUrlChange = setInterval(() => {
        if (location !== window.location.href && window.location.href.includes("watch?v=") && !waitForButtons) {
            console.log("[YTMPV] Video URL detected, toggling waitForButtons...");
            waitForButtons = setButtonInterval();
            location = window.location.href;
        }
    }, 2000);

    function addMpvButton() {
        clearInterval(waitForButtons);
        waitForButtons = null;
        const ytButtons = document.querySelector("#info>#menu-container>#menu>ytd-menu-renderer>#top-level-buttons-computed");
        const ytButton = document.createElement("button");
        ytButton.id = "mpv-button";
        const mpvBtnStyle = `
        #mpv-button {
            color: white;
            cursor: pointer;
            background-color: #043565;
            border-radius: 10px;
            margin-left: 10px;
            margin-right: 10px;
            padding-left: 10px;
            padding-right: 10px;
            font-size: var(--ytd-tab-system-font-size);
            font-weight: var(--ytd-tab-system-font-weight);
            font-family: Roboto, Arial, sans-serif;
            border: 0;
            transition: all 0.2s ease-in-out;
        }
    
        #mpv-button:hover {
            background-color: #0a6dab;
        }`
    
        const styleElem = document.createElement("style");
        if (styleElem.styleSheet) {
            styleElem.styleSheet.cssText = mpvBtnStyle;
        } else {
            styleElem.appendChild(document.createTextNode(mpvBtnStyle));
        }
        document.getElementsByTagName('head')[0].appendChild(styleElem);
        ytButton.textContent = "▶ MPV";
        ytButton.addEventListener("click", () => {
            document.querySelector("video").pause();
            document.location = "mpv://" + document.location.href;
            ytButton.textContent = "⌛ Opening...";
            ytButton.style.cssText = "background-color: #0a6dab;";
            setTimeout(() => {
                ytButton.textContent = "▶ MPV";
                ytButton.style.cssText = "";
            }, 3000);
        });
    
        ytButtons.appendChild(ytButton);
        console.log("[YTMPV] MPV button added");
        console.log(ytButton, ytButtons);
    }
})();
```

This yields us a beautiful button which is not exactly centered but it'll do:
![The Button Bar](https://i.imgur.com/qfdjvNV.png)

Clicking the button shows a little waiting animation upon which mpv will start and the play the video accordingly. We did it!
![mpv playing a video](https://i.imgur.com/cXjxsfQ.png)

If you're interested in trying out this little project yourself, or contribute to it, feel see [this](https://github.com/TibixDev/MPVPlay) repository.

# 0x07 Conclusion
All in all, this project was really fun to work on, albeit a bit annoying at times. I guess the stereotype of a programmer preferring to take a week to automate a task that takes 10 seconds at most is true afterall.

If you're still reading this, thank you for coming along on this journey. I hope you enjoyed it!

*Last edit: 2022/05/04*
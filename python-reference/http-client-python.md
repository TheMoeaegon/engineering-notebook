## Why HTTP?

### Communicating on the Web

Instagram would be pretty terrible if you had to manually copy your photos to your friend's phone when you wanted to share them. Modern applications need to be able to communicate information _between devices_ over the internet.
- Gmail doesn't just store your emails in variables on your computer, it stores them on computers in their data centers
- You don't lose your Slack messages if you drop your computer in a lake, those messages exist on Slack's server.

#### How Does Web Communication Work?

When two computers communicate with each other, they need to use the same rules. An English speaker can't communicate verbally with a Japanese speaker, similarly, two computers need to speak the same language to communicate.

This "language" that computers use is called a protocol. Network protocol is a set of rules or a specific language that devices use to talk to each other. Just like human interactions, different situations require different rules, languages, and levels of security. The most popular protocol for web communication is [HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview), which stands for Hypertext Transfer Protocol.

### HTTP

HTTP, or Hypertext Transfer Protocol, is a protocol designed to transfer information between computers.

There are other protocols for communicating over the internet, but HTTP is the most popular and is particularly great for websites and web applications. Each time you visit a website, your browser is making an HTTP request to that website's server. The server responds with all the text, images, and styling information that your browser needs to render its pretty website!

#### Why Web use HTTP instead of other communication protocol Like TCP?

**Reddit-answer:** HTTP is needed so that communicating systems can understand each other. TCP just provides "envelopes" that can transfer bytes around the network. An application protocol assigns structure and meaning to the contents of the envelopes. If you speak English and I send you a letter written in French, you'll still get the letter, but you won't understand it.

To that point, HTTP builds a lot more functionality over TCP. TCP just makes sure the packet was received. Even if the letter is received by the recipient, the recipient may not understand it or may have trouble acting upon it.

What happens when the recipient gets the letter, can understand it, but is unable to act upon it? Maybe you are asking your friend to mail you a book, but he lent it temporary to a friend. Or maybe he doesn't have it anymore or you asked for an incorrect title

HTTP can let you say things like 302, the resource moved, or various ways of saying 4xx client error or 5xx server errors. It also has various ways of saying 2xx I did it, or 1xx I'm still working on it.

You'd have to reimplement a whole status, notification, and state system yourself if you are just strictly working with raw packets.

To expand on this, TCP doesn't know anything about the data that it passes around. When you receive data through a TCP connection you're essentially just getting a bunch of ones and zeroes and there's no way to know just by looking at it if it's supposed to be an image, text, a file, or whatever. Also TCP tells you nothing about who sends the data and when. The only way around this is if there is an agreement/shared convention (i.e. a protocol) between the applications running on the connected machines about what will happen.

#### HTTP URLs

A URL, or Uniform Resource Locator, is the address of another computer, or "server" on the internet. Part of the URL specifies where to reach the server, and part of it tells the server what information we want.

![[Pasted image 20261104184621.png]]

Put simply, a URL represents a piece of information on some computer somewhere. We can get access to it by making a request, and reading the response that the server replies with.

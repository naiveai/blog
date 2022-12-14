# Protocols As Practice Projects

As a developer, whether you're a hobbyist or in a career, you are (or should be) always looking to improve your skills. But it can be difficult to find a way to do that while fitting into a busy schedule and ensuring that the learning time you put in translates to verifiably better skills, especially if you're concerned about building a resume. There are a lot of suggestions and speculation about this topic out in the blogosphere, but I didn't find a system that really worked for me for a while, and I've finally nailed down exactly why.

## The Internet's Suggestions

There are a few broad categories of suggestions you'll find, which we'll talk about in turn.

### Toy Projects

Generally speaking, these refer to simple beginner projects that can often be finished in a few dozen lines of code. These tend to be in the frontend world: you've got counter apps, todo apps, shopping list apps, etc.

Most of them involve simple CRUD operations that tend to test your basic familiarity with the language and toolset. These can be useful for getting started, and even for more advanced stages if you commit to refining their user interface and/or adding complicated features like decentralization, authentication, etc - the trappings of a "real-world" app.

However, I find that they can be very difficult to get excited about and thus most never get to the stage where they push past your comfort zone and become truly useful for learning. It's also easy to get trapped in a circle of fake productivity where you're remaking the same simple apps in a thousand different frameworks or methodologies.

### Cloning Projects

Another common option is cloning a popular existing app within your domain, such as Twitter or Tetris or some such. This does get more juices flowing and necessarily forces you to think about more practical concerns. But these projects can often be so large that it's difficult to translate them into a scope that makes sense for you; and the large number of existing projects which by definition are identical or better than your goal can greatly drag on your long-term motivation, and tempt even the most honest person to copy significant parts wholesale from others because it's so much easier.

More importantly, I think these kinds of projects ultimately only develop a pretty narrow set of skills, related to refining the user interface or connecting various APIs together or such. Important skills, no doubt, but there's a lot more that needs to be mastered by a good developer.

### Just Solve Your Problems, Bro

Last and very much least, there are the extremely helpful suggestions to simply find a problem you have that needs solving, come up with a way to solve it with tech (an extremely healthy impulse that definitely didn't ruin Silicon Valley), somehow reduce its scope to be manageable with your skill level, reduce the scope again because you definitely didn't reduce it enough the first time (2 weeks in, of course!), let yourself get your hopes up of turning this into a magical unicorn startup despite all evidence to the contrary, then give up on it at some point weeks in.

Yeah, or you could just *not*.

## A Better Way: Implement a Protocol

I honestly think pragmatic protocol pursuits are the perfect programming practice project. They provide a solution to nearly every problem I've had with the above ideas.

Firstly and most importantly, they come with a formal specification, which reduces the amount of upfront work that needs to be done translating vague thoughts in your head into a scope and list of tasks. They also very often have ways to reduce the scope built-in, with protocol versions with backward compatibility or extra features that are optional. For instance, the [BitTorrent protocol](https://www.bittorrent.org/beps/bep_0003.html) has many different extensions which may be used in modern torrents, but the base protocol requires none of them. Or you can choose not to implement legacy options that other modern implementers support from those talking to them but do not use themselves when talking to others.

Increasing scope is easy as well - building custom CLIs/UIs for your library, incorporating extra extensions, optimizing performance, multithreading, or for the truly harebrained among us, thinking up and implementing your own custom extensions.

Learning to read specifications is an incredibly important skill for a developer. Specifications, in practice, aren't nearly as specific as we'd like them to be. But public specifications for important protocols have been refined to a knife's edge to be as simple to read for as wide a variety of people as possible, and they're good training wheels for the fundamental skill of translating requirements to code. This isn't to say that they aren't technically involved, but rather that they're designed to be precise, oftentimes frustratingly so, which makes them look less accessible than they are.

They're also motivating because each step along the way is relatively simple to define and, gradually, to execute; and reaching the end goal means being able to communicate or otherwise interoperate with other systems using that same protocol, which is an incredibly exciting thing to be able to do with your homegrown code. For me, the moment when the BitTorrent client I had implemented with Jeff Zhang ([matey](https://github.com/naiveai/matey)) downloaded its first torrent successfully was ecstatic, and I'll never forget it.

It's also possible to borrow code from other projects, but it's not as easy to copy stuff wholesale if you already have some stuff you need to integrate that code with. Some more niche protocols may not have implementations in the programming language you're working with, which means you still need to study their approach and translate it if you need help.

Protocols are precise, but they also aren't insulated from the messy reality of the world. Once your implementation gets past a basic level, it is naturally going to slam straight into non-conforming peers, hacky workarounds, and (often but not always) the unreliability of networks. Frustrating, but necessary for developing your skills - and by the time you've gotten to this stage, you're often very close, so your motivation can keep you pushing through.

Lastly, they force you to overcome an intimidation factor that I think is very important to a developer's maturity. If you're like me, you're likely coming into this convinced that you're not cut out to implement highly detailed and long specifications for low-level protocols. But most of us have never actually read one of them cover to cover. I hope this article serves as a nudge for you to do so, picking from one of the many important protocols to implement:

*   [TCP](https://www.ietf.org/rfc/rfc9293.html)
    
*   [BitTorrent](https://www.bittorrent.org/beps/bep_0003.html)
    
*   [SMTP](https://www.rfc-editor.org/rfc/rfc3207), for mail
    
*   [WebSocket](https://www.rfc-editor.org/rfc/rfc6455.html)
    
*   A parser for [ISO 8601 dates as used on the Internet](https://www.rfc-editor.org/rfc/rfc3339)
    
*   [DEFLATE compression](https://www.rfc-editor.org/rfc/rfc1951)
    

You don't have to commit to working on them to read them, especially some of the shorter ones toward the end of the list. They're not as scary as they look, and they can provide invaluable experience. Happy protocoling!

*Cover image courtesy of* [*XKCD*](https://xkcd.com/927/)*.*
### What do I do?
I'm a .NET Developer that posts semi-frequently on [Twitter](https://twitter.com/KStrubeG) and [Mastodon](https://hachyderm.io/@KristofferStrube) about the open source projects that I work on, on [GitHub](https://github.com/KristofferStrube). I have previously worked professionally with developing/testing Blazor applications as my primary job, but I recently transitioned into a new job that is more heavily architecture-focused. Therefore I work a lot with Blazor in my spare time to get my curiosity satisfied. I graduated in the summer of 2022 with a Master's degree in Computer Science from Aarhus University. So it should be no surprise that I enjoy technically complex problems especially if they have a focus on modeling systems or if they need to work with large amounts of data. 

### Projects
The ones of you who know me from Twitter/Mastodon probably do so in the context of two of my projects that I have shared a lot about.

The first one is my [Blazor Wrapper for the File System Access API](https://github.com/KristofferStrube/Blazor.FileSystemAccess). This is a wrapper for a somewhat simple API, but also a very useful API as it enables the consumers of the API to interact with their local file system from the browser. This helps us move towards something a lot of Blazor developers want: Application-like websites. This is not the first wrapper that I have made for Blazor *(that is [Blazor.Popper](https://github.com/KristofferStrube/Blazor.Popper))*, but it is the project that I have learned the most from. It has also sprouted a handful of other API wrappers that I have worked on to facilitate rich interactions with the API like [Blazor.FileSystem](https://github.com/KristofferStrube/Blazor.FileSystem), [Blazor.FileAPI](https://github.com/KristofferStrube/Blazor.FileAPI), and [Blazor.Streams](https://github.com/KristofferStrube/Blazor.Streams). Through my work with these I have built myself a good standard for how to wrap browser API's in Blazor and have begun to wrap some of the lower level APIs like the DOM API.

<video width="500" autoplay muted controls loop>
<source src="{site}/videos/Zip_and_IndexedDB.mp4" type="video/mp4">
A video showing some of the functionality of the File System Access API wrapper
</video>

*This video is an example of some of the functionality that the File System Access API wrapper supports. This was also one of the first times I had a sample submitted by someone in a PR.*

The second project some might know me from is my [Blazor SVG Editor](https://github.com/KristofferStrube/Blazor.SVGEditor). This is a much more creative and mathematically challenging project that I have worked on for 2 years. With it, I can edit SVGs simply by dragging anchors around the screen and seeing the resulting SVG code in real-time. I often get back to this project when I feel like making something visual or if I really need to be able to make that specific kind of SVG be that a gradient or an animated stroke offset. This also crosses over into my wrapper libraries every so often like when I made a partial wrapper for the SVG Animation API.

![The Simpson character Bart being drawn in the SVG Editor.](https://github.com/KristofferStrube/Blazor.SVGEditor/raw/main/docs/showcase.gif?raw=true)

*This is an example of an SVG I created 1½ years ago, which actually landed me my first paid gig outside the borders of Denmark.* 

### This platform
When I started writing this I figured this would be the nerdy part, but I see that I have already geeked out a lot, so let's keep it simple. This blog is a static site, generated from markdown files and some configuration in JSON. It is a generator that I have made myself called [StaticBlog.NET](http://StaticBlog.NET), but that is not the important part. The important part is that all the content I will write here is in markdown which can be utilized in a huge variety of ways or be consumed by other blogging platforms. This reminds me of a [comment to a post on Twitter](https://twitter.com/terrajobst/status/1622031997454671872) by [Immo Landwerth](https://twitter.com/terrajobst):

> "This is one of the reasons why folks like me decided that we don’t blog straight via WordPress — we use markdown in GitHub and merging there stages it in Wordpress. If they kill that again, we have a bunch of static markdown files we can easily serve."

This idea of having a structure that I can easily move is important to me and will be a key focus for me when continuing the development of the generator and editor for this blog.

### Future posts
I intend to make some posts here every so often. At least one post every month. I know that setting goals for creative work can be bad a idea, but I thrive under these kinds of hard measurable goals. Some of my first posts will be about my existing open-source projects, the work I have recently done on them, and the work that follows in the near future.
I have also made a list of some topics that I would like to make some posts about in the near future so that I don't forget:

- .EditorConfig files for .NET and C#
- Performance optimizations in .NET 8
- ActivityPub: The open social network standard
- Concurrency with SignalR and Blazor WASM
- The forgotten Typed SignalR Clients introduced in .NET 7
- StaticBlog.NET: A minimal Static Site Generator written in .NET edited with Blazor WASM.

Feel free to reach out if any of the topics above sounds especially interesting to you. Then I might do them first.
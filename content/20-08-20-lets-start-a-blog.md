+++
title = "Let's start a blog!"
date = 2020-08-20
+++

After thinking about starting a programming blog for way too long, I've finally gotten around to actually starting one! Although this will probably just end up being a platform for me to shout my thoughts into a void. But that's fine in and of itself. Almost therapeutic.

For the first post, lets just talk about the technology being used, why, and where we go from here.

## Choosing Technology

I really just wanted a simple static site generator that renders Markdown. Something so that I can focus more on writing blog posts and less on maintaining a website. The good thing is that quality static site generators are everywhere nowadays. Lets talk about a few SSGs that I _didn't_ end up choosing.

### [Hugo](https://gohugo.io/)

Hugo was probably my second choice. It's simple, fast, and static. The big reason I didn't end up using it is because it uses a not-so-popular template format built into the language. Maybe I'll revisit Hugo in the future.

### [Next.js](https://nextjs.org/) / [Gatsby](http://gatsbyjs.org/)

Both of these were rejected for basically the same reason: I don't want to depend on React. It's not that I dislike React - on the contrary it's currently my favorite way to write front-end code - but React just seems too heavy for what I want.

### [Jekyll](http://jekyllrb.com/)

The OG SSG. This probably would have also been a fine choice. It's well documented, well supported, and reliable. But it's not fast, not that speed is all that important to me right now. But I just felt like it would be better to use something a little more modern rather than of using Jekyll itself. Nothing but respect.

## Used Technology

In the end, I decided on using:

### [Zola](https://www.getzola.org/)

From the best I can understand, Zola is very similar to Hugo except that it uses Rust instead of Go, with the biggest difference being the templating engine. Zola's Tera templating engine is much more similar to other templating engines used by other popular SSGs like Jekyll and Django, which is the biggest reason I chose Zola over Hugo.

The biggest drawback of Zola seems to be using TOML instead of YAML for metadata. Since I'm starting out from scratch, TOML vs YAML doesn't really matter to me too much, and writing a converter from one to the other seems like it could be a fun task in and of itself.

And for hosting:

### [Netlify](https://www.netlify.com/)

I don't expect traffic to be high (actually I expect it to be essentially zero), so Netlify should be more than enough for my needs right now. Plus, I already have an account. Let's not think about this too hard right now.

## From Here

There are definitely some things I already know I want to change:

### Template

I appreciate using a pre-built template to get me up and running, but there are already things I don't like about it. I already patched the CSS because my title was too large and overran the div. I plan on probably forking my current template eventually and editing it to my liking. Low priority.

### Favicon

Rather, lack thereof. It's probably as simple as putting a file into the `public` folder, but I don't even have an icon. I'll need to dust off my Illustrator skills. Low priority.

### Page generator

Rather than manually creating posts every time, it would be nice to automate it with a command: `zola new [dir] "title"` or something which would create a pre-annotated `.md` file in the provided posts directory. Low priority.

For now, I'll probably just be thinking about my next post. Until next time!

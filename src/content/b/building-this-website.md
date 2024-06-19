---
title: "[EN] Building this cursed website"
description: "Help"
pubDate: "Jun 10 2024"
category: "web   - "
---

# Building this website

Imagine - if you will - the most curious thing at all. A man,
a beast perhaps, absolute unit in scottish vernacular, who has two wolves inside him.

One wolf says: ***"You avoid everything bloated and mainstream, and have no issues using
exploring and using cursed ancient tools."***

The other says: ***"Try the cool new thing, bro. B-)"***

These are two streams that rarely meet, but once upon a time, I do in fact get an idea for
a really good project that can serve as an outlet to my tendencies.

And so, after many years of experimentation, I once again have a website.

This is the third one. We can devide the main eras by which domain I predominantly used:
- **magnusi.tech**
- **mag.wiki**
- **lho.sh**

As you can see, there is a trend for shorter and shorter domain names. Am I getting lazy
typing a URL?

Each of these has been an absolute technological marvel.

## magnusi.tech

This is the oldest one, and it ran 2014-2016-ish. The most initial version predated my usage
of Rust, but very soon, I wanted things to be interactive.

Sadly, I had to let go of the domain as it was too expensive for my teenage self.

The **magnusi.tech** website ran on an old Raspberry Pi stuck into an outlet in my room. Here
comes the fun part - no ethernet access in our room, and the distribution I was using at the time (some
mad individuals published 32-bit Arch Linux), did not come with `wpa_supplicant` pre-installed.i
Didn't have WiFi either, I used a dongle.

This meant that I had to get things right by doing manual installation. That required several
laps of doing the following:
- Mount the SD card in my laptop
- Install a package (starting with `wpa_supplicant`) or adjust a config (I didn't feel like
  wrangling `systemd` at the time, so I made an `@reboot` job for cron, which I new was running)
- Try to run it
- Wait for it to fail
- Mount the SD card again and read logs

There were instances of missing libraries, and such, for the things I needed to run. But eventually,
I got it working, and had my very own "home server".

A server, that I had to cover with something because the constantly blinking LEDs made
it difficult to sleep.

My first website was an unnecessarily dynamic one. It is a case of ancient Rust. This is the
latest version of that implementation that I found:

```rust
#![feature(label_break_value)]

extern crate ansi_term;
extern crate comrak;
extern crate light_pencil;

#[macro_use]
extern crate lazy_static;

use std::collections::HashMap;
use std::fs::read_dir;
use std::io;
use std::sync::{Mutex, RwLock};
use std::time::SystemTime;

use ansi_term::Colour::{Blue, Green};
use light_pencil::{Pencil, Request};

mod util;

use util::{markdown_page, stylus};

// site w/ sidebar
macro_rules! md {
    ($lit:expr) => {
        Box::new(move |_: &mut Request| {
            markdown_page($lit, TEMPLATE)
        })
    };
}

// site w/out sidebar
macro_rules! raw_md {
    ($lit:expr) => {
        Box::new(move |_: &mut Request| {
            markdown_page($lit, RAW_TEMPLATE)
        })
    };
}

// stylus stylesheet
macro_rules! styl {
    ($lit:expr) => {
        Box::new(move |_: &mut Request| stylus($lit))
    };
}

static RAW_TEMPLATE: &'static str =
    include_str!("../raw_template.html");
static TEMPLATE: &'static str = include_str!("../template.html");

lazy_static! {
    static ref STYLUS_CACHE: Mutex<HashMap<String, (String, SystemTime)>> =
        Mutex::new(HashMap::new());
    static ref PAGE_CACHE: Mutex<HashMap<String, (String, SystemTime)>> =
        Mutex::new(HashMap::new());
    static ref NAME_CACHE: RwLock<Vec<String>> =
        RwLock::new(vec![]);
}

fn main() {
    let mut app = Pencil::new("web");

    app.get("/", "index", md!("index"));

    app.before_request(|request| {
        println!(
            " {} {} from {}",
            Green.bold().paint(request.method.to_string()),
            request.url,
            Blue.bold().paint(request.remote_addr.to_string())
        );

        'dir_loop: for (name, ext) in read_dir("web")
            .unwrap()
            .map(io::Result::unwrap)
            .filter(|x| x.file_type().unwrap().is_file())
            .map(|x| x.path())
            .map(|x| {
                (
                    x.file_stem().map(|s| s.to_owned()),
                    x.extension().map(|s| s.to_owned()),
                )
            })
            .filter(|(s, e)| s.is_some() && e.is_some())
            .map(|(f, e)| {
                (
                    f.unwrap().to_str().map(|s| s.to_owned()),
                    e.unwrap().to_str().map(|s| s.to_owned()),
                )
            })
            .map(|(f, e)| (f.unwrap(), e.unwrap()))
        {
            'read_scope: {
                let list = NAME_CACHE.read().unwrap();

                if list.contains(&name) {
                    continue 'dir_loop;
                }
            }

            'write_scope: {
                let mut list = NAME_CACHE.write().unwrap();
                let name_clone = name.clone();

                match ext.as_ref() {
                    "md" => request.app.get(
                        "/".to_string() + &name,
                        &name,
                        md!(&name_clone),
                    ),
                    "rmd" => request.app.get(
                        "/".to_string() + &name,
                        &name,
                        raw_md!(&name_clone),
                    ),
                    "styl" => request.app.get(
                        "/".to_string() + &name + ".css",
                        &(name.clone() + ".styl"),
                        styl!(&name_clone),
                    ),
                    _ => (),
                }
                list.push(name);
            }
        }
        None
    });
    app.enable_static_file_handling();

    app.run("0.0.0.0:80");
}
```

For the sake of completeness, this is how the `stylus` and `markdown` functions looked like:

```rs
use std::io::Read;
use std::process::Command;
use std::fs::{File, metadata};

use light_pencil::{Response, PencilResult};
use comrak::{ComrakOptions, markdown_to_html};

use PAGE_CACHE;
use STYLUS_CACHE;
use RAW_TEMPLATE;

pub fn markdown_page(name: &str, template: &str) -> PencilResult
{
	let mut page_cache = PAGE_CACHE.lock().unwrap();
	let ext = if template == RAW_TEMPLATE { "rmd" } else { "md" };
	let body = if page_cache.contains_key(&format!("web/{}.{}", name, ext))
	{
		let p = format!("web/{}.{}", name, ext);
		let metadata = match metadata(&p) { Ok(m) => m, _ => { return Ok(Response::from("404")); } };
		let date = match metadata.modified() { Ok(d) => d, _ => { return Ok(Response::from("404")); } };

		let entry = match page_cache.get_mut(&p)
		{
			Some(e) => e,
			None => unreachable!(),
		};

		if entry.1 == date
			{entry.0.clone()}
		else
		{
			let mut f = match File::open(&p) { Ok(f) => f, _ => { return Ok(Response::from("404")); } };
			let mut tmp_str = String::new();
			match f.read_to_string(&mut tmp_str)
			{
				Ok(_) =>
				{
					entry.0 = markdown_to_html(&tmp_str, &ComrakOptions::default());
					entry.1 = date;
					entry.0.clone()
				},
				Err(_) => { return Ok(Response::from("404")); }
			}
		}
	}
	else
	{
		let p = format!("web/{}.{}", name, ext);
		let metadata = match metadata(&p)  { Ok(m) => m, _ => { return Ok(Response::from("404")); } };
		let date = match metadata.modified() { Ok(d) => d, _ => { return Ok(Response::from("404")); } };

		let mut f = match File::open(&p) { Ok(f) => f, _ => { return Ok(Response::from("404")); } };
		let mut tmp_str = String::new();

		if f.read_to_string(&mut tmp_str).is_err()
			{return Ok(Response::from("404"));}

		let s = markdown_to_html(&tmp_str, &ComrakOptions::default());
		page_cache.insert(p, (s.clone(), date.clone()));
		s
	};
	let contents = template.replace("<contents/>", body.as_ref());
	Ok(Response::from(contents))
}

pub fn stylus(name: &str) -> PencilResult {
	let mut stylus_cache = STYLUS_CACHE.lock().unwrap();
	let p = format!("web/{}.styl", name);
	let body = if stylus_cache.contains_key(&format!("web/{}.styl", name))
	{
		let metadata = match metadata(&p) { Ok(m) => m, _ => { return Ok(Response::from("404")); } };
		let date = match metadata.modified() { Ok(d) => d, _ => { return Ok(Response::from("404")); } };

		let entry = match stylus_cache.get_mut(&p)
		{
			Some(e) => e,
			None => unreachable!(),
		};

		if entry.1 == date
			{entry.0.clone()}
		else
		{
			let mut f = match File::open(&p) { Ok(f) => f, _ => { return Ok(Response::from("404")); } };
			let mut tmp_str = String::new();
			match f.read_to_string(&mut tmp_str)
			{
				Ok(_) =>
				{
					entry.0 = std::str::from_utf8(&Command::new("stylus")
							.arg(p).arg("-p").output()
							.expect("can't start stylus, is it installed?").stdout)
						.expect("couldn't convert stylus to CSS").to_string();
					entry.1 = date;
					entry.0.clone()
				},
				Err(_) => { return Ok(Response::from("404")); }
			}
		}
	}
	else
	{
		let metadata = match metadata(&p)  { Ok(m) => m, _ => { return Ok(Response::from("404")); } };
		let date = match metadata.modified() { Ok(d) => d, _ => { return Ok(Response::from("404")); } };

		let mut f = match File::open(&p) { Ok(f) => f, _ => { return Ok(Response::from("404")); } };
		let mut tmp_str = String::new();

		if f.read_to_string(&mut tmp_str).is_err()
			{return Ok(Response::from("404"));}

		let s = std::str::from_utf8(&Command::new("stylus")
				.arg(&p).arg("-p").output()
				.expect("can't start stylus, is it installed?").stdout)
			.expect("couldn't convert stylus to CSS").to_string();
		stylus_cache.insert(p, (s.clone(), date.clone()));
		s
	};
	let mut res = Response::from(body);
	res.set_content_type("text/css");
	Ok(res)
}
```

You may be wondering - why the caching? The reason is quite simple. This code originally used
the `markdown` crate. At the time of that website, this crate was extremely slow. How slow?

On my Raspberry Pi, rendering a 49-line markdown file took almost 20 seconds. Longer files,
such as the actual articles could take over a minute.

This led me to a pretty bad solution - cache the results on the first request, and be the
first client after startup to bite the bullet, and wait the long times for everything.

One important thing to note is that this implementation predates `async/await` in Rust,
and so it uses my lightweight fork of then semi-popular pencil framework, which is synchronous,
like most of the Rust frameworks from that time.

## Advancing on the autism spectrum - V2

This website was full of hacks, and raw html to make up for missing functionality. There was also
absolutely no reason for it to be a dynamic app, when all it was doing was serving static content.
As a matter of fact, there was no Javascript at all.

During this time, I started noticing the popularity of static site generators. In particular,
the generator now known as [`zola`](https://www.getzola.org) was gaining popularity.

So naturally, I chose something completely different for the V2 of my website. I found an old,
abandoned SSG called [`mdblog`](https://github.com/FuGangqiang/mdblog.rs), and I forked,
fixing many of its outstanding issues, making improvements, and, in my classic fashion, deleting
about half of it. I named my fork [`morpho`](https://github.com/luciusmagn/morpho).

The two SSGs are different enough that websites are not cross-compatible.

During this time, I also wrote a different SSG, which I called [`hyper-rat`](https://github.com/luciusmagn/hyper-rat/).

```rs
extern crate fs_extra;
extern crate pulldown_cmark;
extern crate ramhorns;
extern crate regex;
extern crate toml;
#[macro_use]
extern crate die;

use fs_extra::dir;
use pulldown_cmark::{html, Parser};
use ramhorns::{Ramhorns, Template};
use regex::{Captures, Regex};

use std::collections::HashMap;
use std::error::Error;
use std::fs::{create_dir_all, read_dir, read_to_string, write};
use std::path::{Path, PathBuf};

static TEMPLATE: &str = "{{{body}}}";

fn main() -> Result<(), Box<dyn Error>> {
    let content_regex = Regex::new(
        r#"\[\[(?P<content>(((\.\.?/)|([.a-zA-Z0-9_/\-\\]))+(\.[a-zA-Z0-9]+)?))(?P<template> +(((\.\.?/)|([.a-zA-Z0-9_/\-\\]))+(\.[a-zA-Z0-9]+)?))?\]\]"#,
    )?;
    let mut template_cache = HashMap::new();
    template_cache
        .insert("base".to_string(), Template::new(TEMPLATE)?);

    let template_files = read_dir("theme")?
        .into_iter()
        .filter_map(|x| x.ok())
        .map(|x| x.path())
        .filter(|x| x.is_file())
        .collect::<Vec<PathBuf>>();

    let mut templates = Ramhorns::from_folder("theme")?;

    create_dir_all("build")?;
    dir::copy("media", "build/", &{
        let mut c = dir::CopyOptions::new();
        c.overwrite = true;
        c
    })?;
    dir::copy("theme/static", "build/", &{
        let mut c = dir::CopyOptions::new();
        c.overwrite = true;
        c
    })?;

    template_files.iter().for_each(|path| {
        let tpl = templates
            .from_file(
                &path
                    .strip_prefix("theme")
                    .unwrap()
                    .display()
                    .to_string(),
            )
            .unwrap();

        if let Err(e) = tpl.render_to_file(
            &PathBuf::from("build")
                .join(&path.strip_prefix("theme").unwrap()),
            &(),
        ) {
            die!("failed to render to file: {}", e);
        }
    });

    let built = read_dir("build")?
        .filter_map(|x| x.ok().map(|x| x.path()))
        .filter(|x| x.is_file())
        .collect::<Vec<PathBuf>>();

    built
        .iter()
        .map(|x| (x, read_to_string(x)))
        .filter_map(|x| if let (n, Ok(s)) = x { Some((n, s)) } else { None })
        .for_each(|(path, contents)| {
            let processed = content_regex.replace_all(&contents, |caps: &Captures| {
                let path = Path::new(&caps["content"]);
                let mut files = match path {
                    p if !p.exists() => die!("path does not exist: {}", p.display()),
                    p if p.is_file() => vec![p.to_owned()],
                    p if p.is_dir() => read_dir(p)
                        .unwrap_or_else(|_| {
                            die!("could not read directory {}", p.display())
                        })
                        .filter_map(|x| x.ok().map(|x| dbg!(x.path())))
                        .filter(|x| dbg!(x.is_file()) && dbg!(x.to_str().unwrap_or_default().ends_with(".md")))
                        .inspect(|x| {dbg!(x);})
                        .collect::<Vec<PathBuf>>(),
                    p => die!("invalid path: {}", p.display()),
                };

                files.sort();

                dbg!(&files);

                let mut s = String::new();

                println!("{}", caps.len());
                for f in files {
                    let content = match read_to_string(dbg!(f)) {
                        Ok(s) => s,
                        Err(e) => die!("failed to read file {}: {}", path.display(), e),
                    };

                    let tpl_name = caps
                        .name("template")
                        .map(|x| x.as_str().trim())
                        .unwrap_or("base")
                        .to_string();

                    let (head, body);
                    let v: Vec<&str> = content.splitn(2, "\n\n").collect();

                    match v.len() {
                        1 => {
                            body = v[0].trim();
                            head = "".to_string();
                        }
                        _ => {
                            head = v[0].trim().to_string();
                            body = v[1].trim();
                        }
                    }
                    dbg!(&head);
                    dbg!(&body);

                    let body = {
                        let mut h = String::new();
                        html::push_html(&mut h, Parser::new(&body));
                        h
                    };

                    let data = match toml::from_str::<HashMap<String, String>>(&head) {
                        Ok(mut s) => {
                            s.insert("body".into(), body.into());
                            s
                        }
                        Err(_) => {
                            let mut h = HashMap::new();
                            h.insert("body".into(), body.into());
                            h
                        }
                    };

                    let tpl =
                        template_cache.entry(tpl_name.clone()).or_insert_with(|| {
                            match read_to_string(&tpl_name) {
                                Ok(s) => Template::new(s)
                                    .unwrap_or_else(|_| die!("template suck")),
                                Err(e) => die!(
                                    "failed to make template from file {}: {}",
                                    tpl_name,
                                    e
                                ),
                            }
                        });

                    s.push_str(&tpl.render(&data));
                }

                s
            });

            if let Err(e) = write(path, processed.to_string()) {
                die!("failed to write to file {}: {}", path.display(), e);
            }
        });

    Ok(())
}
```

This is its complete source-code. Once again, notice that there are some old Rust idioms,
such as `extern crate` declarations for all dependencies. These are no longer necessary
in contemporary Rust.

## This one

The current website is a complete pivot. I saw **Fireship's** video on **AstroJS** and figured I
could try setting it up with **Bun**. Well, I could. As a result, we have the most complicated file structure
so far:

```
.
├── astro.config.mjs
├── bun.lockb
├── package.json
├── public
│   ├── favicon.svg
│   └── _finsko.jpg
├── README.md
├── src
│   ├── components
│   │   ├── BaseHead.astro
│   │   ├── FormattedDate.astro
│   │   └── Header.astro
│   ├── consts.ts
│   ├── content
│   │   ├── b
│   │   │   ├── building-this-website.md
│   │   │   ├── data-recovery-cz.md
│   │   │   ├── data-recovery-en.md
│   │   │   └── _finsko.jpg
│   │   └── config.ts
│   ├── env.d.ts
│   ├── layouts
│   │   └── BlogPost.astro
│   ├── pages
│   │   ├── b
│   │   │   └── [...slug].astro
│   │   └── index.astro
│   └── styles
│       ├── font
│       ├── global.css
│       └── src
│           └── env.d.ts
├── tsconfig.json
└── yarn.lock

22 directories, 47 files
```

To me, the result is essentially the same. It outputs static HTML. However,
I did gain a live reload when editing, which is nice.

I will probably clean this up and make it open-source.

+++
title = 'Hugo 101'
date = 2024-05-04T15:50:56+03:00
cover = 'images/hugo-bg.jpg'
draft = false
+++

Hugo is an open source static site generator that is powered by Go - which makes it insanely fast to compile and run.
What is a static site? Well it's the most basic website of all, pure HTML. Hugo as a framework allows you to use templates to generate HTML files quickly. 
Let's go over the pros and cons: 
##### PROS
 - It allows for variables in the template (using Hugo's own templating engine based on Go)
 - Highly customizable templates and styles
 - At its core - it's just HTML, with batteries
 - Fairly easy to learn
 - Minimization comes included
##### CONS
 - Not popular outside the US
 - Not non-tech friendly - there are no dashboards or WYSIWYG editors for you to make blog posts


Getting Started
Here are the download links: https://gohugo.io/installation/
To get started, hugo generates a very basic site
```
$ hugo new site quickstart
$ cd quickstart
$ hugo serve
```

```
├── archetypes
│   └── default.md  <--- preconfigured files 
├── assets          <--- CSS / Javascript
├── config.toml     <--- site-wide variables
├── content         <--- your pages
├── data            <--- similar to config, but for content
├── layouts         <--- default/custom layouts
├── public          <--- build folder
├── static          <--- where your image assets go
└── themes
```


Adding CSS and JavascriptThe folder `/assets` is for CSS and Javascript you want to upload to the site, since it uses Hugo Pipes.

>Stores all the files which need be processed by Hugo Pipes. Only the files whose .Permalink or .RelPermalink are used will be published to the public directory.
<sup>[https://gohugo.io/getting-started/directory-structure/](https://gohugo.io/getting-started/directory-structure/)</sup>

I like to put my SCSS into a folder, and import it in the `/layouts/baseof/_default.html` file. 

This will mean that every page will include the whole CSS file. If you'd like to have something more specific to each page, then add the CSS to the HTML for the specific page you want. We'll get into custom pages later. 

<sub>Folder structure</sub>
```
assets
├── js
│   └── index.js
└── scss
    ├── _single.scss
    └── main.scss
```

<sub>main.scss</sub>
```
@import '_single.scss'
```





_single.scss is just a name. But it is convention to add "_" prefix for CSS partials and import them in the main. 

As a cleanliness rule, wrap your CSS partials with a wrapper class so that it doesn't leak outside the intended pages. Especially since all the pages are using the same stylesheet. <sub>_default.html</sub>
```
<!--
    JAVASCRIPT IMPORT
-->
{{ $js := resources.Get "js/index.js" }}
{{ $script := $js | resources.Minify }}
<script defer src="{{ $script.Permalink | relURL }}"></script>

<!--
    CSS IMPORT 
-->
{{ $options := (dict "targetPath" "/css/main.min.css" "outputStyle" "compressed" "enableSourceMap" true) }}
{{ $style := resources.Get "scss/main.scss" | resources.ToCSS $options }}
<link rel="stylesheet" href="{{ $style.Permalink | relURL }}">
```

Note the `resources.Minify` and `resources.ToCSS` that is included. That is the magic, that is what helps minify our Javascript and CSS into production ready code. Hugo has a few more in-built [pipes](http://https://gohugo.io/hugo-pipes/) that you can play with.



Index.html and Content Pages`/layouts/index.html` is a special page that is loaded when we enter the website on `http://domain.com/` i.e. the landing page. Although you can also move index.html to root of `/content` folder, _however_ you won't be able to use partials outside of `/layouts`.

The rest of the pages are loaded from `/content`
```
└── content
    └── about
    |   └── index.html      // <- https://example.com/about/
    ├── posts
    |   ├── firstpost.html  // <- https://example.com/posts/firstpost/
    |   ├── happy
    |   |   └── ness.html   // <- https://example.com/posts/happy/ness/
    |   └── secondpost.html // <- https://example.com/posts/secondpost/
    └── quote
        ├── first.html      // <- https://example.com/quote/first/
        └── second.html     // <- https://example.com/quote/second/
```


# Defaults
<b>/layouts/_default/baseof.html</b> 
is the basic template for every page, and it usually contains the `<head>` and `<style>` declarations. 

```
<main>
    {{ block "main" . }}{{ end }}
</main>
```

`{{ block "main" . }}` 
is where your other pages slot in. In other pages you'll likely see a `{{ define "main" }}`.


<b>/layouts/_default/single.html</b> 
is the template for html pages in `/content/**/*.html`.

<b>/layouts/partials</b> 
contains single partial HTML that can be inserted into other layout pages.



# Templating HTML
Templating uses something akin to Go templating handlebars. There are several functions you can use, but most common will be:<b>Partials</b>
These are used to send in HTML partials you've made from /layouts/partials/\<filename>.html. A dictionary (object) of values can be sent into a partial by using the `dict` function
```
{{ partial "<filename>" (dict "context" . <key> <value> <key> <value>) }}
```

```
{{ partial "contact-button" (dict "context" . 
    "text" "Request A Demo" 
    "color" "#ffee33") 
}}
```

<b>Range</b>
A for-each loop. Ranges require an `{{ end }}`. 
```
{{ range (where .Site.RegularPages "Section" "blog") }}
{{ .Title }}
{{ end }}
```

<b>If</b>
If blocks. Also requires an end.
```
{{ if eq $variable true }}
    <p>This will show if true
{{ end }}
```




FrontmatterFrontmatter such as titles and section are listed on the top of the html page within `/content`. Certain frontmatter params are used for structuring the footer and navbar such as `section` and `subsection`.

You can even use HTML in your Frontmatter, as long as you import it correctly in your template.

<sub>layouts/software/single.html</sub>
```
---
header:
    title: Software Development
    subtitle: The cake is a lie
bgImageUrl: images/content/people.png
text: "Lorem ipsum dolor sit amet consectetur, adipisicing elit. Exercitationem, placeat ipsa ipsum quaerat optio labore impedit. Non commodi sit nihil magnam temporibus magni laboriosam, velit tempore amet iure ut mollitia."
longDesc: 
    <ul>
        <li>Why you can use HTML</li>
        <li>Different ways to use Hugo</li>
        <li>Quality over Quantity</li>
    </ul>
imgs:
    - imgUrl: images/content/meeting-rules.jpg
    - imgUrl: images/content/profile-search.jpg
    - imgUrl: images/content/reporting.png
---
```


Using HTML from FrontmatterBy piping into safeHTML, we can use HTML in our frontmatter
```
{{ if .Params.longDesc }}<p>{{ .Params.longDesc | safeHTML }}</p>{{ end }}
```



Global VariablesAny variable that wish to be saved globally can be set in `config.yaml` under the `params` key. 

You can access the parameter in html using `{{ $.Site.Params.[parameter key]}}`.

<sub>config.yaml</sub>
```
params:
    gtagKey: U-123456
```
<sub>_baseof.html</sub>
```
<script async src="https://www.googletagmanager.com/gtag/js?id={{ $.Site.Params.gtagKey }}"></script>
```


You can access data files via `$.Site.Data.<filename>`

<sub>/data/testimonials.yaml</sub>
```
- 
    icon: la-handshake
    text: lorem ipsum
    position: CEO
    company: Investment Bank
- 
    icon: la-user-tie
    text: lorem ipsum
    position: Director
    company: Chase Bank
```

<sub>/layout/partials/testimonials-section.html</sub>
```
{{ range $.Site.Data.testimonials }}   
<li>
    <img class="la {{ .icon }}"/> <h6>{{ .position }} | {{ .company }}</h6>
    <h4>"{{ .text }}"</h4>
</li>
```

---


Conclusions
Hugo, is easy to learn, quick to develop, flexible enough for customization, yet structured enough for maintenance. 
I like it a lot and would recommend it for any static sites that does not require an editor.
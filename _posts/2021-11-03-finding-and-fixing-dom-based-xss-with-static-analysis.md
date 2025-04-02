---
title: "Finding and Fixing DOM-based XSS with Static Analysis"
date: 2021-11-03
categories: 
  - "firefox-internals"
  - "hardening"
---

Despite [all](https://www.w3.org/TR/CSP1/) [the](https://www.w3.org/TR/CSP2/) [efforts](https://www.w3.org/TR/CSP3/) of fixing Cross-Site Scripting (XSS) on the web, it continuously ranks as one of the [most dangerous security issues in software](https://cwe.mitre.org/top25/archive/2021/2021_cwe_top25.html).

In particular, DOM-based XSS is gaining increasing relevance: DOM-based XSS is a form of XSS where the vulnerability resides completely in the client-side code (e.g., in JavaScript). Indeed, more and more web applications implement all of their UI code using fronted web technologies: Single Page Applications (SPAs) are more prone to this vulnerability, mainly because they are more JavaScript-heavy than other web applications. An XSS in Electron applications, however, has the potential to cause _even more_ danger due to the system-level APIs available in the Electron framework (e.g., reading local files and executing programs).

The following article will take a deeper look into [Mozilla's eslint-based tooling to detect and prevent DOM-based XSS](https://github.com/mozilla/eslint-plugin-no-unsanitized) and how it might be useful for your existing web applications. The eslint plugin was developed as part of [our mitigations against injection attacks in the Firefox browser](https://blog.mozilla.org/attack-and-defense/2020/07/07/hardening-firefox-against-injection-attacks-the-technical-details/), for which the user interface is also written in HTML, JavaScript and CSS.

# Background: Real world example of DOM-based XSS

Let's take a moment to look at typical sources of DOM-based XSS first. Imagine a bit of JavaScript (JS) code like here:

```js
let html = `
  <div class="image-box">
    <img class="image"
         src="${imageUrl}"/>
  </div>`;
// (...)
main.innerHTML = html;
```

You first will notice the variable called `html`, which constructs a bit of HTML using a Javascript [template string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals). It also features an inclusion of another variable - `imageUrl` - to be used in the `src` attribute. The full html string is then assigned to `main.innerHTML`.

If we assume that the `imageUrl` variable is controlled by an attacker - then they might easily break out of the `src` attribute syntax and enter arbitrary HTML of their choosing to launch an XSS attack.

This example demonstrates how easy it is to accidentally implement a DOM XSS vulnerability: The application was expecting an image URL, but also accepts all sorts of strings, which are then parsed into HTML and JavaScripts. This is enough to enable XSS attacks.

If we want to avoid these bugs we need to find all instances in which the application parses a string into HTML and then determine if it can be controlled from the outside (e.g., form input, URL parameters, etc).

To do so efficiently, we are required to inspect various patterns in source code. First, let's look at all assignments to `innerHTML` or `outerHTML`. In order not to miss other sources of XSS, we also need to inspect calls to the following functions: `insertAdjacentHTML()`, `document.write()`, `document.writeln()`.

When first trying this ourselves, we at Mozilla used text search with tools like `grep` or `ripgrep`, but it did not turn out successful: Even a complicated search pattern gave us thousands of results and contained many false positives (e.g., assignments of safe, hardcoded strings). We knew we needed something that is more syntax-aware.

# Linting and Static Analysis

Static Analysis is just another way to say that we want to inspect source code automatically. Our static analysis method builds on existing JavaScript linting with [eslint](https://eslint.org), which supports robust JS source code parsing and also supports new JavaScript syntax extensions. Furthermore, the provided plugin API helps us build an automated check with relatively little new code. However, there are some limitations:

## Caveats

Since we are scanning the JavaScript _source code_, there are some things we can _not_ easily do:

- Static Analysis has almost no visibility into a variable's content (i.e., harmful, harmless, attacker controlled, hardcoded).
- In JavaScript, the source code does not tell us a variable's type (e.g., Number, String, Array, Object)
- Static Analysis is easily fooled by minification, bundling or obfuscation.

At Mozilla, we managed to accept these limitations because we can build on our existing engineering practices:

- All proposed patches are going through code review.
- The repository contains all relevant JavaScript source code (e.g., third-party libraries are vendored in).

The latter point is sometimes hard to guarantee and requires using dependencies through published and versioned libraries. Third-party JavaScript dependencies through `<script>` elements are therefore out of scope. For a cohesive security posture, the associated security risks need to be mitigated by other means (e.g., using in-browser checks at runtime like CSP). You should validate whether the following assumptions also hold true for your project.

## How Static Analysis works

To explain the implementation of our eslint plugin, let's take a look at how JavaScript can be parsed and understood by eslint: A common representation is the so-called Abstract Syntax Tree (AST). Let's take a look at the AST for a simplified version of our vulnerability from above:

`foo.innerHTML = evil`:

```
AssignmentExpression (operator: =)
|-- left: MemberExpression
| |-- object: Identifier "foo"
| `-- property: Identifier "innerHTML"
`-- right
`-- Identifier "evil"
```

Indeed, the whole line is seen as an assignment, with a left and a right side. The right side is a variable (Identifier) and the left side `foo.innerHTML` is accessing the property of an object (MemberExpression).

Now let's look at the AST representation for a case, where XSS is not possible, which just assigns an empty string: `foo.innerHTML = ""`.

```
AssignmentExpression (operator: =)
|-- left: MemberExpression
| |-- object: Identifier "foo"
| `-- property: Identifier "innerHTML"
`-- right
`-- Literal ""
```

Did you spot the difference? Again the assignment has a left and right side. But in this case, the right node is of type _Literal_ (i.e., a hardcoded string).

We can use exactly these kinds of differences to understand the basics of our linter plugin: When looking at assignments, all hardcoded strings are considered trustworthy and do not need further static analysis. But only if all patches are subject to code review, before being committed to the source code. Naturally, the plugin has many more syntax expressions to take into account.

While bearing in mind, that the abstract syntax tree can _not_ tell us anything about a variable despite its name, we probably want to allow some other things: In our case, we configured our linter runtime (not the plugin itself) to skip files if they are in the `test/` folder, as we do not expect test code to be running on our users' systems.

We also need to take false positives into account. False positives are incorrect detection of code, in which the content of the variable is known to be safe through other means. Here, we recommend our developers to use a trusted Sanitizer library that will always return XSS-safe HTML. Essentially, we allow all code on the right side of the assignment as long as it is wrapped in a function call to a known sanitizer like so:

```js
foo.innerHTML = DOMPurify.purify(evil);
```

We currently recommend using [DOMPurify](https://github.com/cure53/DOMPurify/) as your sanitizer and our linter allows such calls in our default configuration. In parallel, we are also actively working on specifying and implementing a secure [Sanitizer API](https://wicg.github.io/sanitizer-api/) for the web platform. Either way, as long as our sanitizer function is well implemented, the input data doesn't have to be.

With all these techniques and decisions in mind, we ended up developing an eslint plugin called [**`eslint-plugin-no-unsanitized`**](https://github.com/mozilla/eslint-plugin-no-unsanitized), which also contains checks for other typical XSS-related source code fragments like `document.write()` and is fully configurable in terms of which sanitizers you want to allow.

# Evaluation & Integration

When we first tried finding XSS in the Firefox UI automatically, we used `grep` and spotted thousands of potential vulnerabilities. With the eslint plugin, we reduced this number to 34 findings! This reduction enabled us to start a focused manual code audit and resulted in finding only two critical security bugs. Imagine trying to identify those two bugs by going through thousands of potential findings manually.

Eventually, we fully integrated [eslint-plugin-no-unsanitized](https://github.com/mozilla/eslint-plugin-no-unsanitized) into our CI systems by choosing an iterative approach:

- We enabled the linter over time and directory by directory.
- We skip test files.
- We also had to allow some exceptions for code that violates the linter but was not actually insecure (validated through code audit).

An important note here is that allowing linter violations incurs a risk that needs to be temporary. It's still useful to tolerate exceptions _during_ the migration to the linter plugin, but not after. We've also experienced that developers misunderstand the purpose of the linter and try to design their own path of evading these checks. Our lesson: By controlling the path for exceptions and escalations, we were able to understand and adopt the tool to find workable solutions for all developers and their use cases.

Once all code has been integrated, it should be on the security & analysis teams to get the number of exceptions down to zero. With all those bugs fixed and most linter violations resolved, we are running the plugin against all newly submitted Firefox code and have a pretty good handle on XSS issues in our codebase.

# Conclusion: You can fix DOM-XSS

Fixing DOM-based XSS across a whole codebase is not easy, but we believe this overview will serve as a useful guide: As a first step, we can highly recommend just using the [eslint plugin no-unsanitized](https://github.com/mozilla/eslint-plugin-no-unsanitized) as it is and running it against your source code. A dry-run will already tell you whether the topic of DOM-based XSS is a problem at all. Our integration section showed how you can integrate the linter gradually, based on risk or feasibility. But we also want to note that source code analysis is not a silver bullet: There are notable caveats and it is useful to complement static analysis with enforcement in the browser and at runtime. But, eventually you will be able to get rid of a lot of DOM-XSS.

_This is a summary of my presentation of the same title, delivered at Sekurak Mega Hacking Party (June 2021) and JSCamp Barcelona (July 2021). Feel free to [reach out](https://frederik-braun.com/contact.html), if you want me to talk about web security at your event_.

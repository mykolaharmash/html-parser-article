Use cases:
Syntax highlighting
Parsing a structure, like matching parentheses
Custom template language
Components server side rendering
Custom renderer

How browser gets html and which role charset plays.
Why it's important to explicitly specify charset.
How browser parses partially delivered html.
Tokenizer and parser, why tokenizer is actually more important.
Algorithm of tokenizing.
Tokenizing a stream of characters.

Links:
Other parsers implemented in JS (html-parser2, parse5)
WHATWG Specification for html tokenizer and parser


Web developers, including myself, tend to forget how many stuff are resolved for us by the browsers. Fonts, media, networking, layout, rendering — to name few. But it's even easier to start taking for granted very subtle but fundamental features like parsing HTML and CSS or executing JavaScript. I personally never bothered to understand how parsing of HTML, for example, actually works.

Couple of month ago I started my another pet-project for which I needed to render custom elements in HTML on a server-side. And as any normal developer I've been carried away a little and, naturally, wrote my own HTML parser. You've been there, right? 😀

Here I want to share my findings and just general knowledge. It might be very useful for those who staring with web development. So, let's start.

Besides obvious purpose — to render this shiny page to you, HTML parser can be used in a lot more scenarios.

* Highlighiting a syntax in your editor.
* Analysing structure, like highlighing matching brackets or validation against some standard.
* Working with artificial templates like in VueJS, Angular or React's JSX.
* Server-side rendering
* And some crazy stuff, like custom custom renderers.

All of those have the same basis. The same which browser has. 

At this point lets assume that we know nothing. What happens between entering URL and watching that funny kitten video insightful conference talk is complete magic. Browser — just a blackbox with input (URL) and output (Page).

Input is URL, ok. Using this URL browser will go and get address of a server and will ask this server for information. In a simple case, information on a server is a file with our hand-written piece of HTML. 

Encoding

File is a chunk of data, zeros and ones in different combinations which in general do not make sense, unless someone who reads those 0s and 1s knows that author of the file was using combination 00111100 for a symbol "<" or 01010110 for a letter "V". Basically, reader has a table of different combinations and corresponding characters. This is where encoding comes into play.

This table of characters (encoding) needs to be passed along with a file itself from author to reader. Fortunately, all kinds of encodings nowadays are standard and every computer has them by default, so there is no need to send whole tables, we just need a table-name. UTF-8 is one of a popular encodings.

When you are typing symbols into HTML file, actual 0s and 1s are generated according to encoding currently set in your editor.

// Image with typing symbols

File is ready and can be sent via network to a user who entered URL in a browser. This is a responsibility of HTTP server, program which going to receive requested URL, read the file and send it bit by bit along with additional data (headers), like encoding name. HTTP header used for sending encoding name is "Content-Type" and it looks like this "Content-Type: text/html; charset=utf-8".

It's very important to always send encoding name to browser, otherwise it will try to guess and chances are pretty big that it will be wrong guess, as a result it will use wrong characters table and render your 0s and 1s as a completely different symbols.

// Image with sending bytes to a browser

https://jsfiddle.net/azya3htz/10/
https://jsfiddle.net/e82o2pnv/

Tokenization

Now bits of data start to hit our browser. Thanks to a known encoding it can make sense out those bits and convert them back to a human readable symbols. Next step is to figure out which of those symbols are custom text and which are special HTML-grammar. Such process is called tokenization, as its purpose is to assign tokens, meaningful for a browser, to a parts of input string.

Let's take a piece of HTML.

<div class="block">
	<p>
		Custom Text
	</p>
	<button class="send-something">Send</button>
</div>

Small piece, but lot of tokens: open tags, closing tags, attributes, text. All of those needs to be found.

// Image with assigning tokens

Result of tokenization will be an array of tokens. Plain array. At this point browser does not tries to build any structure or establish hierarchy between elements, just looks for a known grammar.

[
	{ type: 'tag-open-start', content: '<div' },
	{ type: 'attribute-key', content: 'class' },
	{ type: 'attribute-assignment', content: '=' },
	{ type: 'attribute-value-wrapper', content: '"' },
	{ type: 'attribute-value', content: 'block' },
	{ type: 'attribute-value-wrapper', content: '"' },
	{ type: 'tag-open-end', content: '>' },
	…
]

From my experience, tokenization is the most important and most difficult part of parsing. From the amount and quality of generated tokens depends quality of later AST, which is used for rendering and analysis.

Implementation of tokenizer is not trivial. To be honest, anything that have to handle human-written text is not trivial. People make mistakes, don't follow rules or standards. And this needs to be taken into account, no way around. Finding tokens in a code that we have above it's actually pretty easy, but in reality, you going to need to handle mess like this:

---

just a text, go along

<div  class=' so far ok' data-value=57 >
	more text
	here
</div  >

<!--
	comment here-->

<div "-oops" class  =   "it's a class"  ="no-key-at-all" total="mess>here">

  tag here < but not really

  <custom-component ="again-no-key" disabled 
  ></custom-component>

  <br>
  <img />

  <span>finally, normal tag</span>

  yeah, I don't feel like closing tags today anymore…

---

Ok, maybe It's a bit exaggerated and I hope people don't really write this, but you get the point. Browser needs to handle a lot of edge cases.

Tokenizer implementation is based on a concept called state-machine. A thing that hold a state, takes an input and switches to another state based on the input.

I'm going to illustrate implementation I've come up with and then explain most important details. Just to clear — I'm sure browsers have much more optimal algorithms as they are written by much smarter folks than myself 🤪. My implementation is just a one of possible solutions.

// Image with a state machine and caret going through characters

On a left side you can see the state-machine. It takes input of characters one by one. For every character it decides if it needs to switch to another state or perform some action like generating a new token.

Problem with HTML is that you can not make decision based on one character, at least, not in all cases. For example if the input is symbol "<" it can be a start of a tag or it can be just a custom text with this symbol. To decide you need the next character, if it's a letter than you have a tag, if it's a space — you have a custom text. 

So, you have to have a buffer where to collect characters until you sure what they are. Lets call it a decision buffer. On the image above you can see it marked with a __color__ color.

Every input character goes into a decision buffer. If the state-machine cannot decide what to do with a character, it leaves it in the buffer and waits for another one. Once there are enough characters to make a decision, state-machine will react and then clear the buffer.

State-machine can decide that characters inside the buffer are not any kind of grammar and they do not signal to switch to some another state. In this case those symbols go into a content buffer, on the image it's __color__. Content buffer accumulates characters for a future token. Ones state-machine decides that it needs to switch to another state, it looks if there is something in the content buffer and generates a new token with that content.

It worth noting that every state of the state-machine can switch only to certain other states. Meaning the same character for one state has completely different meaning for another. This way tokenizer can make a difference between symbols "<div" in text, which would mean start of a new tag, and the same in attribute value 'class="<div"', which is normal valid attribute and you don't want to recognize is as a tag. Being in a state of attribute value state-machine just does not have an option to switch into tag state.

// Image with a simplified schema of state-machine states connections

Streaming

One of the coolest features in a browser is to start parsing and rendering while still getting data from the network. It turned that with a state-machine approach implementation of such streaming is very easy. All you need to do is to save current state (decision buffer, content buffer and number of processed on character) when you're done with one chunk of data and start with the same state for the next chunk. That is it, no changes to the algorithm needed at all, just an intermediate storage for the state.

Syntax Highlighting

Generated array of tokens is not only material for further processing, it is very useful by its own. One popular use-case is syntax highlighting. Think about this, you already have an array with all detected keywords. All you have to do now is to go through that array and replace words in original HTML string with colored versions based on token types. 

If you are implementing syntax highlighting in a browser it means wrapping every keyword with a tag which has a class for each token.

// Image with plain text, array of tokens and highlighted text

// Link to a codepen with syntax highlighting

Parsing a Structure

Lets back on track, remember, we are trying to display a page here. File is transferred through the network and browser has detected all tokens. Next step would be to establish connections between those tokens. Meaning to find out that some attributes belong to some tag and tag also has a couple of child tags inside. This is where famous DOM being born.

This is how algorithm I came up with looks like.

// Image with a process of constructing a tree

Here you can see the same approach with a state-machine. Let's call this this state-machine The Constructor. Input in this case is a tokens generated by Tokenizer. For each token Constructor decides which state it needs to switch to.

Important detail is that Constructor has a pull of currently opened tag. You can see on the image at the __position__. The last one opened gets all the content. When Constructor finds close tag which matches last opened one it removes tag from the pull, so parent tag starts to get content. This way the tree of tags is constructed.

// Image with tree structure of tags 

Processing the Tree

At this point official parsing part is over. Browser has a tree of tags with all attributes and it can start for example traversing it looking for "img" or "script" tags to download those files specified in "src" attribute. Or it can go ahead and draw a nice rounded rectangle with overlaid text for a tag named "button".

And like with Tokenizer, parsing a structure and processing it can be done while still receiving more data. Approach here is the same.

While parsing HTML browser is doing all the same stuff for CSS and JavaScript. All the same downloading, tokenizing and tree-constructing. With a different algorithms, of course, but the high-level approach is unchanged. And in the end you have you page as a combination of those parsed trees.

Besides browser use-cases, your editor uses the tree to highlight matching tags or show an error when you have unclosed one. You can use this tree for server side rendering of custom components for example or to implement some special template language inside HTML. All kinds of crazy stuff.

That Is It From Me

Thank you for making it this far! Source code of my parser you can find on GitHub, project is called __name__ https://github.com/nik-garmash/html-parser, consider it more like case study than something for production use.

Here is couple of great implementations, well tested and proven to be reliable on real projects.

Parse5 (https://github.com/inikulin/parse5)
htmlparse2 (https://github.com/fb55/htmlparser2)

And, of course

WhatWG HTML Specification, includes Tokenization and Tree Construction. (https://html.spec.whatwg.org/multipage/parsing.html#tree-construction)

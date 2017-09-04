---
layout: post
title: How to use the OpenNLP&nbsp;1.5.0 Parser
tags:
- OpenNLP
---

I'm back to try and figure out how in the world to make use of the Open NLP Parser.  

I'm only going to warn you once: this is a long post.  Go grab a beer or a glass of wine or some coffee before starting.  It's long.  Now I've warned you twice.

<!--more-->

First, a quick refresher:

- [Getting started with OpenNLP 1.5.0 - Sentence Detection and Tokenizing]({{ site.nav.main.posts }}/2011/05/05/opennlp-1-5-0-basics-sentence-detection-and-tokenizing)
- [Part-of-Speech (POS) Tagging with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2011/06/30/part-of-speech-pos-tagging-with-opennlp-1-5-0)
  - [Part-of-Speech (POS) Tags: Penn English Treebank]({{ site.nav.main.posts }}/2011/12/28/opennlp-part-of-speech-pos-tags-penn-english-treebank)
- **How to use the OpenNLP 1.5.0 Parser** (surprise, you're reading it)
- [Making Coreference Resolution your bitch with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2012/11/04/making-coreference-resolution-with-opennlp-1-5-0-your-bitch)
- Extracting Names with the OpenNLP 1.5.0 Named Entity Finder

### Getting Started

#### model files
Only one additional model file is needed for parsing (which also seems to include noun phrase chunking).  That said, you don't need to know how to do any noun phrase chunking on your own.

- en-parser-chunking.bin

As with all of the model files, it can be found at [http://opennlp.sourceforge.net/models-1.5/](http://opennlp.sourceforge.net/models-1.5) and are identified by language and component.  There's no info provided on this one, but I'm guessing that it was also trained on the [CoNLL 2000 shared task data](http://www.cnts.ua.ac.be/conll2000/chunking/) (as is `en-chunker.bin`, which is used for noun phrase chunking).

I use maven, so these files go into `src/main/resources` and are loaded with `getResourceAsStream`, as you'll see below.

### Parsing
So what is Parsing?  The Parser page on the now-defunct OpenNLP SourceForge wiki defined the Parser as:

> TODO: Write an introduction for the parser.

The [Parsing section of the Apache OpenNLP Developer Documentation](https://opennlp.apache.org/docs/1.5.3/manual/opennlp.html#tools.parser.parsing) moved the ball foward by offering a little more info but ends with this nugget:

> TODO: Extend this section with more information about the Parse object.

What it _actually_ does is takes a sentence like this:

`The quick brown fox jumps over the lazy dog.`

and turns it into a parse tree with part-of-speech tags that looks like this:

`(TOP (NP (NP (DT The) (JJ quick) (JJ brown) (NN fox) (NNS jumps)) (PP (IN over) (NP (DT the) (JJ lazy) (NN dog)))(. .)))`

which is useful for performing coreference resolution. Coreference resolution identifies when multiple expressions in a sentence or document refer to the same thing.  I talk about that in the next post about [making coreference Resolution your bitch]({{ site.nav.main.posts }}/2012/11/04/making-coreference-resolution-with-opennlp-1-5-0-your-bitch).

#### Creating a `Parse` object

First, for some silly reason, you need to create your own `Parse` object.  Yes, before parsing you create a `Parse` object.  Strange, no?  

<a name="update2"></a>

---

_**Update:**_ As [iosu notes in the comments](#comment-1471), all of this logic to create a `Parse` object could be replaced with a simple call to **`ParserTool.parseLine(sentence, _parser, 1)`** after [initializing `_parser` as shown below](#parsing-a-parse).

However, I've noticed that the resulting parse does not have punctuation separately tokenized (i.e., in the example parse tree above, `(NN dog)` is now `(NN dog.)`) which leads to some [differences during Coreference Resolution](#update1).

---

This code uses the `_tokenizer` so before moving on make sure that you've already tackled [sentence detection and tokenization](2011-05-05-opennlp-1-5-0-basics-sentence-detection-and-tokenizing) before proceeding.

No really, go read that link.  I'm not fucking around.

Done?  OK, here's how to create your own `Parse` from an array of tokens:

<a name="update1"></a>

---

_**Update:**_ Thanks to [a comment by Jonathan Huts](#comment-1165), I've simplified the following code to use the Tokenizer's **`tokenizePos`** method, which will save you from manually creating the individual token spans.

---

```java
private Parse parseSentence(final String text) {
   final Parse p = new Parse(text,
         // a new span covering the entire text
         new Span(0, text.length()),
         // the label for the top if an incomplete node
         AbstractBottomUpParser.INC_NODE,
         // the probability of this parse...uhhh...? 
         1,
         // the token index of the head of this parse
         0);

   // make sure to initialize the _tokenizer correctly
   final Span[] spans = _tokenizer.tokenizePos(text);

   for (int idx=0; idx < spans.length; idx++) {
      final Span span = spans[idx];
      // flesh out the parse with individual token sub-parses 
      p.insert(new Parse(text,
            span,
            AbstractBottomUpParser.TOK_NODE, 
            0,
            idx));
   }

   Parse actualParse = parse(p);
}
```

Still with me?  I'm impressed.  Go get a refill on whatever you're drinking (you are drinking, right?).  We're almost done!

#### Parsing a `Parse`

Now that you've actually created a `Parse` object you can...well...parse it!  Watch the magic unfold:

```java
private Parser _parser = null;

private Parse parse(final Parse p) {
   // lazy initializer
   if (_parser == null) {
      InputStream modelIn = null;
      try {
         // Loading the parser model
         modelIn = getClass().getResourceAsStream("/en-parser-chunker.bin");
         final ParserModel parseModel = new ParserModel(modelIn);
         modelIn.close();
         
         _parser = ParserFactory.create(parseModel);
      } catch (final IOException ioe) {
         ioe.printStackTrace();
      } finally {
         if (modelIn != null) {
            try {
               modelIn.close();
            } catch (final IOException e) {} // oh well!
         }
      }
   }
   return _parser.parse(p);
}
```

That's it!  The actual parsing isn't really any different from the other OpenNLP tools, but creating that initial `Parse` object isn't exactly spelled out very clearly elsewhere.

Hope it helps, drop a comment if you have any problems or just to give a shout-out!

---

_**Next Step:**_ [Making Coreference Resolution your bitch with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2012/11/04/making-coreference-resolution-with-opennlp-1-5-0-your-bitch)

My source code and test cases can be found at [https://github.com/dpdearing/nlp](https://github.com/dpdearing/nlp)

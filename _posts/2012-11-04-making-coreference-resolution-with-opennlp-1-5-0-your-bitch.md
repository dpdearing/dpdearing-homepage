---
layout: post
title: Making Coreference Resolution your bitch with OpenNLP&nbsp;1.5.0
tags:
- OpenNLP
excerpt: First thing's first--what is coreference resolution?  Well, it's difficult ...
---

First thing's first--what is coreference resolution?

Co-reference means that multiple expressions in a sentence or document refer to the same thing.  OpenNLP contains a "linker" that analyzes the tokens of a sentences to identify which chunks of text refer to the same things (e.g., people, organizations, events).

Take, for example, the sentence **"John drove to Judy's house.  He made her dinner."**  In this example both **John** and **He** refer to the same entity/person (John); and **Judy** and **her** refer to the same, different entity/person (Judy).  Don't expect OpenNLP to get this 100% correct.  Even a simple example like this is a difficult problem.

Picking up where I left off once upon a time (and finally wrapping up this series), here are links to the old material:

- [Getting started with OpenNLP 1.5.0 - Sentence Detection and Tokenizing]({{ site.nav.main.posts }}/2011/05/05/opennlp-1-5-0-basics-sentence-detection-and-tokenizing)
- [Part-of-Speech (POS) Tagging with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2011/06/30/part-of-speech-pos-tagging-with-opennlp-1-5-0)
  - [Part-of-Speech (POS) Tags: Penn English Treebank]({{ site.nav.main.posts }}/2011/12/28/opennlp-part-of-speech-pos-tags-penn-english-treebank)
- [How to use the OpenNLP 1.5.0 Parser]({{ site.nav.main.posts }}/2011/12/04/how-to-use-the-opennlp-1-5-0-parser)
- **Making Coreference Resolution your bitch with OpenNLP 1.5.0** (surprise, you're reading it)
- Extracting Names with the OpenNLP 1.5.0 Named Entity Finder

### Getting Started
#### model files

Coreference resolution uses a folder of pre-trained model libraries.  You will need them all, which I put in my local `lib/opennlp/coref` folder.  All of the necessary model files can be found at [http://opennlp.sourceforge.net/models-1.4/english/coref/](http://opennlp.sourceforge.net/models-1.4/english/coref).  Be careful when downloading these.  On my Macbook, OSX tries to add an incorrect ".txt" extension to several of the files.

I placed these files in `lib/opennlp/coref` and pass that path directly to the `Linker` constructor, as you'll see below.

#### WordNet Dictionary
You'll also need the database files for the WordNet dictionary.  You can find them at [http://wordnet.princeton.edu/wordnet/download/current-version](http://wordnet.princeton.edu/wordnet/download/current-version), but (as [Samyak noted in the comments](http://blog.dpdearing.com/2012/11/making-coreference-resolution-with-opennlp-1-5-0-your-bitch/#comment-73245)) the WordNet 3.0 "just database files" seems to be incomplete.  Instead, get the `dict` folder from either the full source code and binaries or try the link for the WordNet 3.1 database files.

Once you have the necessary `dict` folder, specify its location as a java VM argument:

**`-DWNSEARCHDIR=path/to/wordnet/dict`**

### Coreference Linker

Initializing the coreference `Linker` object is pretty straightforward.  Point it to the model folder and specify the `LinkerMode`.  I'm not sure why, but I was only able to get it to work as expected when I used `LinkerMode.TEST`.

```java
Linker _linker = null;

try {
   // coreference resolution linker
   _linker = new DefaultLinker(
         // LinkerMode should be TEST
         //Note: I tried LinkerMode.EVAL for a long time
         // before realizing that this was the problem
         "lib/opennlp/coref", LinkerMode.TEST);
   
} catch (final IOException ioe) {
   ioe.printStackTrace();
}
```

Using the Linker to actually extract entity mentions is actually pretty tricky.  I had to dig into the OpenNLP source code to get this to work.

Below is a helper function that I created to handle the actual finding entity mentions, which makes use of the OpenNLP Parser on line 11 ([see my post on the Parser for the `parseSentence` helper method]({% post_url 2011-12-04-how-to-use-the-opennlp-1-5-0-parser %})).  First, the each sentence parse is used to identify entity mentions.  These `Mention` objects contain information about the entity references.

The `Mention`s that do not have a corresponding `Parse` must have one created and set before passing those entity mention objects into the `_linker.getEntities` method, which returns an array of `DiscourseEntity` objects.

I've highlighted the important lines. Good luck:

```java
public DiscourseEntity[] findEntityMentions(final String[] sentences,
      final String[][] tokens) {
   // tokens should correspond to sentences
   assert(sentences.length == tokens.length);
   
   // list of document mentions
   final List<Mention> document = new ArrayList<Mention>();

   for (int i=0; i < sentences.length; i++) {
      // generate the sentence parse tree
      final Parse parse = parseSentence(sentences[i]);
      
      final DefaultParse parseWrapper = new DefaultParse(parse, i);
      final Mention[] extents = _linker.getMentionFinder().getMentions(parseWrapper);

      //Note: taken from TreebankParser source...
      for (int ei=0, en=extents.length; ei<en; ei++) {
         // construct parses for mentions which don't have constituents
         if (extents[ei].getParse() == null) {
            // not sure how to get head index, but it doesn't seem to be used at this point
            final Parse snp = new Parse(parse.getText(), 
                  extents[ei].getSpan(), "NML", 1.0, 0);
            parse.insert(snp);
            // setting a new Parse for the current extent
            extents[ei].setParse(new DefaultParse(snp, i));
         }
      }
      document.addAll(Arrays.asList(extents));
   }

   if (!document.isEmpty()) {
      return _linker.getEntities(document.toArray(new Mention[0]));
   }
   return new DiscourseEntity[0];
}
```

### Example
`Pierre Vinken, 61 years old, will join the board as a nonexecutive director Nov. 29.  Mr. Vinken is chairman of Elsevier N.V., the Dutch publishing group.  Rudolph Agnew, 55 years old and former chairman of Consolidated Gold Fields PLC, was named a director of this British industrial conglomerate.`

And here are the groups of entities that OpenNLP will extract, including (for some reason) the trailing space.  Unfortunately, they aren't very accurate:
- `[this British industrial conglomerate ]`
- `[a nonexecutive director ]` , `[chairman ]` , `[former chairman ]` , `[a director ]`
- `[Consolidated Gold Fields PLC ]`
- `[55 years ]`
- `[Rudolph Agnew ]`
- `[Elsevier N.V. ]` , `[the Dutch publishing group ]`
- `[Pierre Vinken ]` , `[Mr. Vinken ]`
- `[Nov. 29 ]`
- `[the board ]`
- `[61 years ]`

<a name="update1"></a>

---

_**Update:**_ I get different results if using **`ParserTool.parseLine(..)`**--[as described here]({% post_url 2011-12-04-how-to-use-the-opennlp-1-5-0-parser %}#update2)--instead of the procedure described in [my post on the Parser]({% post_url 2011-12-04-how-to-use-the-opennlp-1-5-0-parser %}).  Entity names include additional trailing punctuation and entity resolution appears to be even worse:

- `[this British industrial conglomerate. ]`
- `[chairman ]`
- `[former chairman ]`
- `[a director ]`
- `[Consolidated Gold Fields PLC, ]`
- `[55 years ]`
- `[Rudolph Agnew, ]`
- `[Rudolph Agnew, 55 years old and former chairman of Consolidated Gold Fields PLC, ]`
- `[the Dutch publishing group. ]`
- `[Elsevier N.V. ]`
- `[Mr. Vinken ]`
- `[a nonexecutive director Nov. ]`
- `[the board ]`
- `[Pierre Vinken, 61 years ]`

---

My source code and test cases can be found at [https://github.com/dpdearing/nlp](https://github.com/dpdearing/nlp)

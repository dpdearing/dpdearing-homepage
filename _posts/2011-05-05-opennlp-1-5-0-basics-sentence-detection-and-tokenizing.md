---
layout: post
title: Getting started with OpenNLP&nbsp;1.5.0 – Sentence&nbsp;Detection and Tokenizing
tags:
- OpenNLP
excerpt: OpenNLP is a poorly-documented pain in the ass to figure out.  Here's hoping that I can help myself and others understand it just a little better.

---
OpenNLP is a poorly-documented pain in the ass to figure out.  There are [various](http://opennlp.sourceforge.net/README.html) scattered [resources](http://opennlp.apache.org/docs/1.5.3/manual/opennlp.html) you can find on the internet, none of which are particularly thorough, accurate, or up to date.

The most useful that I've found is a blog post called [Getting started with OpenNLP (Natural Language Processing)](http://danielmclaren.com/node/49), but it is several years old and refers to version 1.4.3 (1.5.x is what I'll discuss here).  That post is quite helpful, but still required digging into the source code to figure out the beast that is coreference resolution. 

Here's hoping that I can add a few posts to the conversation and help both myself and maybe others.

---

_**Update, April 2017:**_ OpenNLP has moved over to Apache and the documentation has been updated a bit, although it still appears to be lacking quite a bit.  See [https://opennlp.apache.org](https://opennlp.apache.org) 

---

Most (if not all) of the more advanced OpenNLP components rely on text that is broken into sentences and/or tokens, so I'm starting with those...

- **Getting started with OpenNLP 1.5.0 - Sentence Detection and Tokenizing** (surprise, you're reading it)
- [Part-of-Speech (POS) Tagging with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2011/06/part-of-speech-pos-tagging-with-opennlp-1-5-0)
  - [Part-of-Speech (POS) Tags: Penn English Treebank]({{ site.nav.main.posts }}/2011/12/opennlp-part-of-speech-pos-tags-penn-english-treebank)
- [How to use the OpenNLP 1.5.0 Parser]({{ site.nav.main.posts }}/2011/12/how-to-use-the-opennlp-1-5-0-parser)
- [Making Coreference Resolution your bitch with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2012/11/making-coreference-resolution-with-opennlp-1-5-0-your-bitch)
- Extracting Names with the OpenNLP 1.5.0 Named Entity Finder

### Getting Started

#### jar dependencies

You'll need three jar files to get started, all can be found in the binary downloads at [http://sourceforge.net/projects/opennlp/files/OpenNLP Tools/1.5.0/](http://sourceforge.net/projects/opennlp/files/OpenNLP%20Tools/1.5.0/). After expanding the files, you'll find `opennlp-tools.1.5.0.jar` and, under the `lib` folder, `maxent-3.0.0.jar` and `jwnl-1.3.3.jar` (for the Java WordNet Library).

---

_**Update:**_ You can look at the [Maven Dependency](http://opennlp.apache.org/maven-dependency.html) page for up-to-date information. I am still using 1.5.3 with this maven dependency:

```
<dependency>
   <groupId>org.apache.opennlp</groupId>
   <artifactId>opennlp-tools</artifactId>
   <version>1.5.3</version>
</dependency>
```
---

#### model files

You'll probably want the pre-trained model files as a starting point (rather than creating/training your own). They can be found at [http://opennlp.sourceforge.net/models-1.5/](http://opennlp.sourceforge.net/models-1.5/) and are identified by language and component. For this tutorial you'll need:

- en-sent.bin
- en-token.bin

I use maven, so I just drop these files into `src/main/resources` and load them with `getResourceAsStream`, as you'll see below.

### Sentence Detection

The Sentence Detector is actually described well in the [Apache OpenNLP Developer Documentation on Sentence Detection](https://opennlp.apache.org/documentation/1.7.2/manual/opennlp.html#tools.sentdetect.detection), so I'll just quote what's there (errors theirs, emphasis mine):

> The **OpenNLP Sentence Detector can detect that a punctuation character marks the end of a sentence or not**. In this sense a sentence is defined as the longest white space trimmed character sequence between two punctuation marks. The first and last sentence make an exception to this rule. The first non whitespace character is assumed to be the begin of a sentence, and the last non whitespace character is assumed to be a sentence end.
>
> ...
>
> The OpenNLP Sentence Detector **cannot identify sentence boundaries based on the contents of the sentence**. A prominent example is the first sentence in an article where the title is mistakenly identified to be the first part of the first sentence.

To the code!

```java
SentenceDetector _sentenceDetector = null;

InputStream modelIn = null;
try {
   // Loading sentence detection model
   modelIn = getClass().getResourceAsStream("/en-sent.bin");
   final SentenceModel sentenceModel = new SentenceModel(modelIn);
   modelIn.close();

   _sentenceDetector = new SentenceDetectorME(sentenceModel);

} catch (final IOException ioe) {
   ioe.printStackTrace();
} finally {
   if (modelIn != null) {
      try {
         modelIn.close();
      } catch (final IOException e) {} // oh well!
   }
}
```

And then actually using the `_sentenceDetector` is simple:

```java
_sentenceDetector.sentDetect(content);
```

### Tokenizing

Once again, for these simple components the [Apache OpenNLP Developer Documentation on the Tokenizer](https://opennlp.apache.org/documentation/1.7.2/manual/opennlp.html#tools.tokenizer.introduction) has a good description:

> The OpenNLP Tokenizers segment an input character sequence into tokens. Tokens are usually words, punctuation, numbers, etc.


```java
Tokenizer _tokenizer = null;

InputStream modelIn = null;
try {
   // Loading tokenizer model
   modelIn = getClass().getResourceAsStream("/en-token.bin");
   final TokenizerModel tokenModel = new TokenizerModel(modelIn);
   modelIn.close();

   _tokenizer = new TokenizerME(tokenModel);

} catch (final IOException ioe) {
   ioe.printStackTrace();
} finally {
   if (modelIn != null) {
      try {
         modelIn.close();
      } catch (final IOException e) {} // oh well!
   }
}
```

And then, once again, actually using the `_tokenizer` is simple:

```java
_tokenizer.tokenize(sentence);
```

### Example

Here are the expected results for an example taken from those wiki pages, with an added fourth sentence to show off some edge cases.

`Pierre Vinken, 61 years old, will join the board as a nonexecutive director Nov. 29. Mr. Vinken is chairman of Elsevier N.V., the Dutch publishing group. Rudolph Agnew, 55 years old and former chairman of Consolidated Gold Fields PLC, was named a director of this British industrial conglomerate. The previous contraction-less sentences don't have boundary/odd cases...this one does.`

Sentences:

1. `Pierre Vinken, 61 years old, will join the board as a nonexecutive director Nov. 29.`
2. `Mr. Vinken is chairman of Elsevier N.V., the Dutch publishing group.`
3. `Rudolph Agnew, 55 years old and former chairman of Consolidated Gold Fields PLC, was named a director of this British industrial conglomerate.`
4. `The previous contraction-less sentences don't have boundary/odd cases...this one does.`

Tokens:

1. `[Pierre] [Vinken] [,] [61] [years] [old] [,] [will] [join] [the] [board] [as] [a] [nonexecutive] [director] [Nov.] [29] [.]`
2. `[Mr.] [Vinken] [is] [chairman] [of] [Elsevier] [N.V.] [,] [the] [Dutch] [publishing] [group] [.]`
3. `[Rudolph] [Agnew] [,] [55] [years] [old] [and] [former] [chairman] [of] [Consolidated] [Gold] [Fields] [PLC] [,] [was] [named] [a] [director] [of] [this] [British] [industrial] [conglomerate] [.]`
4. `[The] [previous] [contraction-less] [sentences] [do] [n't] [have] [boundary/odd] [cases] [...this] [one] [does] [.]`


---

**_Next Step:_** [Part-of-Speech (POS) Tagging with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2011/06/part-of-speech-pos-tagging-with-opennlp-1-5-0)

My source code and test cases can be found at [https://github.com/dpdearing/nlp](https://github.com/dpdearing/nlp)

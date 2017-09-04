---
layout: post
title: Part-of-Speech (POS) Tagging with OpenNLP&nbsp;1.5.0
tags:
- OpenNLP
---
Continuing from where I left off, I'm going to quickly touch on part-of-speech tagging before moving on.  It's actually pretty straightforward once you're set up to run OpenNLP.  

<!--more-->

This all assumes that you've already done sentence detection and tokenization.  If you haven't, go back to the beginning.  Here are the links to the rest of my posts:

- [Getting started with OpenNLP 1.5.0 - Sentence Detection and Tokenizing]({{ site.nav.main.posts }}/2011/05/05/opennlp-1-5-0-basics-sentence-detection-and-tokenizing)
- **Part-of-Speech (POS) Tagging with OpenNLP 1.5.0** (surprise, you're reading it)
  - [Part-of-Speech (POS) Tags: Penn English Treebank]({{ site.nav.main.posts }}/2011/12/28/opennlp-part-of-speech-pos-tags-penn-english-treebank)
- [How to use the OpenNLP 1.5.0 Parser]({{ site.nav.main.posts }}/2011/12/04/how-to-use-the-opennlp-1-5-0-parser)
- [Making Coreference Resolution your bitch with OpenNLP 1.5.0]({{ site.nav.main.posts }}/2012/11/04/making-coreference-resolution-with-opennlp-1-5-0-your-bitch)
- Extracting Names with the OpenNLP 1.5.0 Named Entity Finder

### Getting Started

#### model files
Only one additional model file is needed for part-of-speech tagging.

- en-pos-maxent.bin

As with all of the model files, it can be found at [http://opennlp.sourceforge.net/models-1.5/](http://opennlp.sourceforge.net/models-1.5/) and are identified by language and component.  I'm using the **Maxent model** (maximum entropy) instead of the **Perceptron model** because, frankly, I'm not familiar with  what the Perceptron model is.  If you care, you can [read about OpenNLP Maxent here](http://maxent.sourceforge.net/about.html).

I use maven, so these files go into `src/main/resources` and are loaded with `getResourceAsStream`, as you'll see below.

### Part-of-Speech Tagging
The description from the [Apache OpenNLP Developer Documentation on Tagging](https://opennlp.apache.org/docs/1.5.3/manual/opennlp.html#tools.postagger.tagging):
> The Part of Speech Tagger **marks tokens with their corresponding word type based on the token itself and the context of the token**. A token can have multiple pos tags depending on the token and the context. The OpenNLP POS Tagger **uses a probability model to guess the correct pos tag** out of the tag set. To limit the possible tags for a token a tag dictionary can be used which increases the tagging and runtime performance of the tagger.

Code time:
```java
Tokenizer _tokenizer = null;

InputStream modelIn = null;
try {
   // Loading tokenizer model
   modelIn = getClass().getResourceAsStream("/en-pos-maxent.bin");
   final POSModel posModel = new POSModel(modelIn);
   modelIn.close();

   _posTagger = new POSTaggerME(posModel);

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

And then to use it:
```java
_posTagger.tag(tokens);
```
Although the tokenizer only returns an array of string tokens, the `tag` method is overloaded to accept a list of strings, an array of strings, or a single string.

### Example
Here are the expected results for an example taken from the above wiki page (and corrected).

Tokens:
1. `[Pierre] [Vinken] [,] [61] [years] [old] [,] [will] [join] [the] [board] [as] [a] [nonexecutive] [director] [Nov.] [29] [.]`
1. `[Mr.] [Vinken] [is] [chairman] [of] [Elsevier] [N.V.] [,] [the] [Dutch] [publishing] [group] [.]`

Part-of-Speech Tags:
1. `[NNP] [NNP] [,] [CD] [NNS] [JJ] [,] [MD] [VB] [DT] [NN] [IN] [DT] [JJ] [NN] [NNP] [CD] [.]`
1. `[NNP] [NNP] [VBZ] [NN] [IN] [NNP] [NNP] [,] [DT] [JJ] [NN] [NN] [.]`

---

**Update:** These are Penn English Treebank POS tags

---

**_Next Step:_** [How to use the OpenNLP 1.5.0 Parser]({{ site.nav.main.posts }}/2011/12/04/how-to-use-the-opennlp-1-5-0-parser)

My source code and test cases can be found at [https://github.com/dpdearing/nlp](https://github.com/dpdearing/nlp)

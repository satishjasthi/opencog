
# AIML in the AtomSpace

There are recurring, assorted requests to be able to script chatbot
interactions using AIML, and this directory contains the tools to
support such capability.

Please observe that using AIML is very much AGAINST the spirit of
OpenCog, and so these tools are only grudgingly provided.  The real
goal of OpenCog is to create a machine that can learn and think, rather
than run off of hard-coded, human-created rules.  Of course, we've
de-facto violeted this rule repeatedly: the link-grammar rules are
human-created and hard-coded, as are the relex and relex2logic rules.
Alas. So perhaps using AIML is no real crime.

## Status
Beta.  The code is done, its probably buggy. It attempts to support the
most commonly used features of AIML version 2.1

The mixed-mode operation of AIML with other verbal and non-verbal
subsystems is in development (alpha stage).


## Using
There are two steps to this process: The import of standard AIML markup
into the AtomSpace, and the application of the imported rules to the
current speech (text-processing) context.

The goal of the import step is to map AIML expressions to equivalent
atomspace graphs, in such a way that the OpenCog pattern matcher can
perform most or all of the same functions that most AIML interpretors
perform. The goal is NOT to re-invent an AIML interpreter in OpenCog!
... although, de-facto, that's what the end-result is: an AIML
interpreter running in the AtomSpace.

The current importer generates OpenPsi-compatible rules, so that the
OpenPsi rule engine can process these.  This should allow AIML to be
intermixed with other chat systems, as well as non-verbal behaviors.
This mixed-mode operation is in development.

## Running AIML in OpenCog
AIML files need to be converted into Atomese, using the perl script
provided in the `import` directory.  Then do this:

```
(use-modules (opencog) (opencog nlp) (opencog nlp aiml) (opencog openpsi))
(primitive-load "/tmp/aiml.scm")
(aiml-get-response-wl (string-tokenize "call me ishmael"))
```

The various sections below provide additional under-the-cover details.


## Lightning reivew of AIML
Some example sentences.

```
        Input               That Topic     Result
(R1)  Hello                  *     *    Hi there
(R2)  What are you           *     *    I am Hanson Bot
(R3)  What are you thinking  *     *    How much I like you
(R4)  Hi                     *     *    <srai>Hello</srai>
(R5)  I * you                *     *    I <star/> you too.
(R6)  What time is it        *     *    The time is <syscall>
(R7)  *                      *     *    I don't get it.
(R8)  ...                    *     *    <set topic = "animals dog"/>
(R9)  ...                    *     *    <set name="topic">animals dog</set>
(R10) ...                    *     *    <set name="topic"><star/></set>
(R11) ...                    *   "* dog"   ...
(R12) ...                    *     *    I like <get name="topic"/>
(R13) ...                    *     *    I like <bot name="some_key"/>
(R14) Is it a <set>setname</set> ?
(R15) Are you <bot name="key"/> ?
(R16) Are you <set><bot name="species"/></set> ?
```

Future, not a current part of AIML:
```
(F1)  Do you like <topic/> ?
(F2)  Do you like <get name="topic"/> ?
```

### Notes
* Notice that the pattern match of R3 takes precendence over R2.
* srai == send the result back through. ("stimulus-response AI")
* that == what the robot said last time. (just a string)
  A full history of the dialog is kept, it can be refered to ...
  (TBD XXX How ???)
* topic == setable in the result. (just a string)
  topic is just one key in a general key-value store of per-conversation
  key-value pairs.

* star and underscore (`*` `_`) are essentially the same thing; it's an AIML
  implementation detail.
* star means "match one or more (space-separated) words"
  stars are greedy.
* carat and hash (`^` `#`) mean "match zero or more words"
* There is an implicit star at the end of all sentences; it is less
  greedy).
* (R11) the wild-card match matches the second of two words in a
  2-word topic.  (TBD XXX what about N words ??)
* (R14) the wild-card match limited to one of a set.


## Pattern Recognition

Implementing AIML efficiently in the atomspace requires that the pattern
matcher be run "in reverse": rather than applying one query to a
database of thousands of facts, to find a small handful of facts that
match the query; we instead apply thousands of queries to a single
sentence, looking for the handful of queries that match the sentence.
In this sense, AIML is an example of pattern recognition, rather than
querying.  The traditional pattern-matcher link types `BindLink`,
`GetLink` and `SatisfactionLink` are not appropriate; instead, the
underlying search is performed with `DualLink`.

Given a graph (containing only constants -- that is, a "ground term")
the `DualLink` mechanism will find all graphs with variables in them
such that these graphs might be grounded by the given graph.
See `http://wiki.opencog.org/w/DualLink` for details.


### Globbing

The pattern matcher uses GlobNode to perform globbing.
See `http://wiki.opencog.org/w/GlobNode` for details.
See `glob.scm` for a simple working example.


## OpenCog equivalents
* R5 example.

The R5 sentence in the example above is converted into the following
OpenPsi fragment:

```
ImplicationLink
   AndLink
      ListLink
         Word "I"
         Glob "$star"
         Word "you"
		ListLink
         Word "I"
         Glob "$star"
         Word "you"
         Word "too"
```


### AIML tags
All AIML tags are converted into `DefinedSchema` nodes.  There are
processing routines to carry out the actions.  For example, the
&lt;person&gt; tag gets converted to
```
      (ExecutionOutput
         (DefineSchema "AIML-tag person")
         (ListLink
             (Glob "$star-1")))

```


##Design Notes

### Input
Sentences from RelEx.  That is, the normal RelEx and R2L pipeline
runs, creating a large number of assorted atoms in the atomspace
representing that sentence.  See the RelEx and R2L wiki pages for
documentation.

### Pre-processing
The script `make-token-sequence` creates a sequence of word tokens from
a given RelEx parse.  For example, the sentence "I love you" can be
tokenized as:
```
(Evaluation
   (PredicateNode "Token Sequence")
   (Parse "sentence@3e975d3a-588c-400e-a884-e36d5181bb73_parse_0")
   (List
      (Concept "I")
      (Concept "love")
      (Concept "you")
   ))
```

The above form is convenient for the AIML rule processing, as currently
designed.  Unfortunately, it erases information about specific words,
such as part-of-speech tags, tense tags, noun-number tags, etc.  Such
information could be available if we used this form instead:

```
(Evaluation
   (PredicateNode "Word Sequence")
   (Parse "sentence@3e975d3a-588c-400e-a884-e36d5181bb73_parse_0")
   (List
      (WordInstance "I@a6d7ba0a-58be-4191-9cd5-941a3d9150aa")
      (WordInstance "love@7b9d618e-7d53-49a6-ad6e-a44af91f0433")
      (WordInstance "you@cd136fa4-bebe-4e89-995c-3a8d083f05b6")
   ))
```

However, we will not be using this form at this time, primarily because
none of the AIML rules to be imported are asking for this kind of
information.

### Rule selection
The OpenPsi-based rule importer represents a typical AIML rule in the
following form:
```
(Implication
	; context
   (List (Concept "I") (Glob "$star") (Concept "you"))
	; goal
   (List (Concept "I") (Glob "$star") (Concept "you") (Concept "too"))
	; demand
	(Concept "AIML chatbot demand")
)
```

To find this rule, we need to search for it.  This can be accomplished
by saying:
```
(cog-execute!
	(Dual
		(List (Concept "I") (Concept "love") (Concept "you"))))
```
which will find the above rule (and any other rules) that match this
pattern. Alternately, there is a convenience wrapper:
```
(psi-get-dual-rules
	(List (Concept "I") (Concept "love") (Concept "you"))))
```

But first, before we can do this, we must find the current
sentence and parse in the atomspace.  This can be done by saying:
```
(Get
   (VariableList
      (TypedVariable (Variable "$sent") (VariableType "SentenceNode"))
      (TypedVariable (Variable "$parse") (VariableType "ParseNode"))
      (TypedVariable (Variable "$tok-seq") (VariableType "ListLink"))
   )
   (And
      (State (AnchorNode "*-eva-current-sent-*") (Variable "$sent"))
      (ParseLink (Variable "$parse") (Variable "$sent"))
      (Evaluation
         (PredicateNode "Token Sequence")
         (Variable "$parse")
         (Variable "$tok-seq"))
   ))
```

Running the above will return the sequence of words that have
been recently uttered.

### Rule application
Having found all of the rules, we now have to run them ... specifically,
we have to apply them to the current parse.  We can do this in one of
several ways.  These are:

* Create a NEW bindlink, which mashes together both the search for the
  current parse, and the AIML rule, and generate the resulting output,
  which then needs to be routed to the chatbot, to be voiced.

* Since we already know the current parse, we can limit the application
  of the naive AIML rule to only that parse.  This requires treating the
  rule as a kind of PutLink, i.e. of applying it only to a specified
  set.

### TODO
* openpsi duallink is returning too much -- should pre-filter the results.
* Implement the person tag (XXX atomic.aiml seems to mis-use this
   tag???)
* Implement "that" (XXX need examples of how that works, for testing).

### Misc Notes

Hints for hand-testing this code:

Pre-compile the AIML files (this can take over an hour)
(This is optional)
```
time guild compile "/tmp/aiml.scm"
```

```
(use-modules (opencog) (opencog nlp aiml))
```
A normal file load triggers a compile; avoid this as follows:
```
(primitive-load "/tmp/aiml.scm")
(use-modules (system base language))
(compile "/tmp/aiml.scm")
(load-compiled (compiled-file-name "/tmp/aiml.scm"))

```

Search for the rules:
```
;; YOU CAN DO BETTER
(psi-get-dual-rules (List (Word "you") (Word "can") (Word "do") (Word "better")))
(string-tokenize "you can do better")
(string-tokenize "you are such a winner")
```
Search for duals by hand:
```
(define sent
	(List (Word "you") (Word "can") (Word "do") (Word "better")))

(cog-execute! (DualLink sent))

(cog-incoming-set sent)
(cog-get-root sent)

(define s3 (string-tokenize "who supports Trump?"))
(define s3a (string-tokenize "who endorses Trump?"))
(define s4 (string-tokenize "who won the superbowl"))
(aiml-get-response-wl s4)
```


###Notes:

```
psi-get-dual-rules calls psi-get-member-links

(cog-execute! (car (aiml-get-response-wl s3)))

(use-modules (opencog) (opencog nlp) (opencog exec))
(cog-execute! (Map
	(Implication
		(ListLink (Word "who") (Word "supports") (Glob "$star-1"))
		(ListLink (Word "who") (Word "endorses") (Glob "$star-1")))
	(Set (ListLink (Word "who") (Word "supports") (Word "Trump")))
))

(cog-execute! (Map
	(Implication
		(ListLink (Word "who") (Word "supports") (Glob "$star-1"))
		(ExecutionOutput (DefinedSchema "AIML-tag srai")
			(ListLink
				(ListLink (Word "who") (Word "endorses") (Glob "$star-1")))))
	(Set (ListLink (Word "who") (Word "endorses") (Word "Trump")))
))

(aiml-get-response-wl
	(ListLink (Word "who") (Word "endorses") (Word "Trump")))

(psi-rule
   ;; context
   (list
      (ListLink
         (Word "who")
         (Word "endorses")
         (Glob "$star-1")
      ) ; PATEND
   ) ;TEMPATOMIC
   ;; action
   (ListLink
      (Word "The")
      (Word "crazies")
      (Word "like")
      (Glob "$star-1")
   ) ; TEMPATOMICEND
   (Concept "AIML chat goal")
   (stv 1 0.8)
   (psi-demand "AIML chat" 0.97)
) ; CATEND

(aiml-get-response-wl (string-tokenize "will you remember what"))
(aiml-get-response-wl (string-tokenize "what will you remember"))
(aiml-get-response-wl (string-tokenize "will you remember that"))
MAY I TEACH YOU
REMEMBER THAT

(aiml-get-response-wl (string-tokenize "you do not learn"))
(aiml-get-response-wl (string-tokenize "call me ishmael"))

-- non-trivial that:
THAT IS A GOOD PARTY

(use-modules (ice-9 ftw))

(define (load-all-files DIR)
	(map
		(lambda (fn) (primitive-load (string-join (list DIR fn) "/")))
		(scandir DIR (lambda (fname) (string-contains fname ".scm"))))
	#t)

(load-all-files "/home/linas/src/opencog/opencog/nlp/aiml/import/aiml-scm/")
```

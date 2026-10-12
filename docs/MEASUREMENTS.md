# What has been measured, and what has not

Every figure here comes from a run on one machine (an Apple M4 with 16 GB, shared
with other programs) and is stated with what it does not show. A figure that has
not been measured is listed as such.

## Running a model larger than memory

A mixture-of-experts model keeps its dense weights in memory and its experts
memory-mapped from disk. Qwen3 30B A3B, a 17.28 GiB file (0.93 GiB dense, 16.35 GiB
experts; 8 of 128 experts used per token), ran on the 16 GB machine with about 2 GiB
committed.

| | |
| --- | --- |
| start to healthy | 14.5 s |
| generation | 1.5-2.7 tokens/s |
| reading a long prompt | 11-13 tokens/s (a short prompt pays about 10 s fixed) |
| short question, one-sentence answer | about 22 s |
| 482-token document, one-sentence answer | about 50 s |
| capability on 24 probes | 22 of 24 (a 4B model: 20; an 8B math-tuned model: 16) |

It is for a hard question that can wait, not for a conversation. Speed depends on
how much of the 16 GiB of experts the operating system can keep cached, which is
whatever the rest of the machine leaves free: with 1.4 GiB free it fell to 0.5
tokens/s. A busy page cache competes with other programs' memory, so streaming on a
shared machine costs them something. CUDA placement has not been measured.

Things only running it showed: llama.cpp repacks CPU weights into memory by default
(a 130 s start and 4.6 GB pushed into swap); on Metal the whole mapped file is
wrapped in GPU buffers (out of memory on the first compute); and one variant of the
model cannot stop thinking, so it is not routed unless reasoning is switched on.

## Reading the same context twice

A prompt cache reuses only the longest common prefix of two prompts. The same three
documents, re-read on the 30B model: in a different order, 383 tokens and 30.5 s; in
the same order, 7 tokens and 4.7 s. The program therefore writes context in an order
chosen so that consecutive prompts share a prefix, and tells the planner which model
already holds which prefix. Over the labelled questions of two sample datasets, the
shared prefix rose from 7.9% to 15.9% and from 1.3% to 13.8% of context tokens.

Not measured: the end-to-end effect of this on a real session with a 30B model (the
machine did not have the memory free to run it), and whether answer quality is the
same when context is placed in the user turn rather than a system message.

## Parallel requests on a streamed model

Four requests at once on the streamed 30B took 149 s in total; the same four one
after another took 136 s. Each sequence picks its own experts, so the disk has to
supply the union. A streamed model therefore runs one request at a time. For a model
held fully in memory, extra slots are a setting that has not been measured.

## Choosing context

Before 2.14 the answering document was exposed for 31% of questions on one sample
playbook and ranked first for 8%. After changes to the lexical embedder and the
neighbourhood radius it ranked first for 73%. On 18 held-out questions about the
product's own documentation, the answering document was exposed for 89%, at about
700 tokens per question.

## Cost

On local models, with real answers graded against facts the answering document
states, the same number of questions were answered to within one, with 35-54% fewer
input tokens, and about twice the output tokens (the planner chose the more verbose
model, which is why answer length is now learned per model). **No saving has been
measured on a hosted provider's pricing, and none is claimed.**

## Not verified

* The Anthropic and Gemini adapters are tested against local servers that speak the
  documented wire formats, not against the live services. `topospace models ping`
  checks a key and model id against the live service.
* Routing quality across hosted vendors has not been measured; the planner's choices
  are only as good as the capabilities you configure.
* The program has not been independently security-reviewed.
* A role restricts which models a request may use; it does not split one request
  across models.
* Text chat only: tool calls and images are not translated between providers.

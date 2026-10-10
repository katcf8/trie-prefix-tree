# trie python: Build a Prefix Tree From Scratch and Use It for Autocomplete, Word Search II, and LeetCode 208

Search "trie python" and you usually want one of three things: a `Trie` class you can paste into a LeetCode answer, an explanation of why the prefix tree beats a plain `set` for prefix queries, or a working autocomplete function. This covers all three, plus the three follow-up problems that use the exact same node structure.

## What a trie actually is

A trie (pronounced "try," from re**trie**val) stores strings by sharing prefixes. Every node represents a character path, and all descendants of a node share that prefix. The root is empty. If you insert `app` and `apple`, you don't get two separate chains — you get one chain of `a → p → p`, and the `p` node gets a flag saying "a word ends here," then continues on to `l → e`.

That single design decision is what buys you everything else.

The standard Python alternative is a `set` of strings. It's faster to build, uses less memory, and answers `"apple" in words` in roughly O(1). What it cannot do is answer *"give me every word starting with `app`"* without scanning the entire set and doing a `startswith` on each entry — O(n · L) where n is the number of words. A trie answers the same query in O(L), because the walk to the prefix node only depends on the length of the prefix, not on how many words you stored.

> Rule of thumb: if you never ask a prefix question, don't build a trie. A set will outperform it on every axis except prefix search.

## Two ways to store children, and the trade-off nobody mentions

A trie node needs three things: a way to find child nodes by character, a flag marking the end of a word, and nothing else.

**Dictionary children.** `{"a": TrieNode, "b": TrieNode, …}`. Works with any alphabet — Unicode, digits, mixed case. Memory grows with the number of *actual* characters you stored, not the alphabet size. This is what most Python solutions use.

**Fixed 26-slot list.** `[None] * 26`, indexed by `ord(ch) - ord('a')`. No hashing per step, and index lookups are about as cheap as Python gets. The catch: you allocate 26 slots whether the node has one child or twenty-six. For a large word list where most nodes sit near the leaves with a single child, that's a lot of `None` objects sitting around.

| Approach | Works for non a–z | Memory per node | Typical use |
| --- | --- | --- | --- |
| `dict` children | Yes (Unicode, digits, mixed case) | Grows with real edges | LeetCode answers, general code |
| `[None] * 26` list | Lowercase English only | 26 slots, always | Performance-sensitive, fixed alphabet |
| Nested plain dicts, no node class | Yes | Less than a class per node | Speed-coding interviews |

There's a third style worth knowing because it's very short: a trie made entirely of nested dictionaries with a sentinel key marking word ends. `node.setdefault(c, {})` on insert, `'#' in node` to check for a word. It's fewer lines and no classes, which is fine for an interview whiteboard but harder to extend when you need per-node data like word frequency.

## The full implementation

Here's the version I'd write in an interview: explicit node class, dictionary children, one shared helper for walking down a prefix.

python
class TrieNode:
    __slots__ = ("children", "is_word")

    def __init__(self):
        self.children = {}
        self.is_word = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]
        node.is_word = True

    def _walk(self, s: str):
        """Return the node at the end of s, or None if the path breaks."""
        node = self.root
        for ch in s:
            node = node.children.get(ch)
            if node is None:
                return None
        return node

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and node.is_word

    def starts_with(self, prefix: str) -> bool:
        return self._walk(prefix) is not None


Three details worth noticing.

`search` and `startsWith` are the same traversal with one extra check at the end. `startsWith` stops as soon as the path exists; `search` also demands `is_word`. That difference is the entire point of the `is_word` flag — without it, `search("app")` would return `True` after inserting only `"apple"`, which is wrong.

Don't write `node.children.setdefault(ch, TrieNode())`. It looks cleaner, but `setdefault` evaluates its default argument on every call, so you construct and immediately discard a `TrieNode` at every step of every insert. Minor for small inputs, pure waste for large ones.

`__slots__` on `TrieNode` matters more than it looks. A trie allocates one node per character, so you can easily end up with hundreds of thousands of them. Without `__slots__`, each node carries a `__dict__`, and that overhead is often larger than the payload itself.

## Complexity, stated plainly

| Operation | Time | Space |
| --- | --- | --- |
| `insert(word)` | O(L) | O(L) new nodes worst case |
| `search(word)` | O(L) | O(1) |
| `starts_with(prefix)` | O(P) | O(1) |
| Collect all words under a prefix | O(P + total output characters) | O(output) |
| Whole trie | — | O(total characters inserted) |

L is word length, P is prefix length. Nothing here depends on how many words you stored, which is the property that makes tries worth the memory.

That memory is the honest downside. Each node is a Python object plus a dict, and CPython dicts are not small — an empty dict alone runs about 64 bytes before any entries. For a few thousand words this doesn't matter. For a production autocomplete over millions of entries, it does.

## Getting every word under a prefix (the autocomplete part)

This is the function people actually search for, and it's the one most tutorials leave as an exercise.

python
def words_with_prefix(self, prefix: str) -> list[str]:
    start = self._walk(prefix)
    if start is None:
        return []
    out = []
    stack = [(start, prefix)]
    while stack:
        node, so_far = stack.pop()
        if node.is_word:
            out.append(so_far)
        for ch, child in node.children.items():
            stack.append((child, so_far + ch))
    return out


An iterative stack instead of recursion, deliberately. A trie built from long strings can be deeper than Python's default recursion limit, and `sys.setrecursionlimit` is a bandage, not a fix. The recursive version using `yield from child.all_words(prefix + ch)` is elegant and shows up constantly in Stack Overflow answers, but the iterative one won't blow up on a 10,000-character word.

Sorting is a separate concern. The stack version returns words in an order determined by dict iteration, which is insertion order but not alphabetical. If you need lexicographic output — which real autocomplete does — sort the result, or push children in sorted order and use a FIFO queue.

## The three LeetCode variants that reuse this exact node

**208. Implement Trie (Prefix Tree)** — the code above, with the method named `startsWith` instead of `starts_with`. Medium, ~66% acceptance. If you can write `_walk` without thinking, you can write this problem in under ten minutes.

**211. Design Add and Search Words Data Structure** — same trie, but `search` accepts `.` as a wildcard matching any single character. The fix is a small DFS:

python
def search_wildcard(self, word: str) -> bool:
    def dfs(node, i):
        if i == len(word):
            return node.is_word
        ch = word[i]
        if ch == ".":
            return any(dfs(child, i + 1) for child in node.children.values())
        nxt = node.children.get(ch)
        return nxt is not None and dfs(nxt, i + 1)

    return dfs(self.root, 0)


Worth noting the complexity honestly: with a full alphabet and many dots, this degrades toward exponential branching. It's still the accepted answer, because the constraints keep real inputs tame.

**212. Word Search II** — build a trie from the word list, then DFS over the grid while walking the trie in parallel. The trie is what makes it work: instead of re-scanning every word at every cell, you prune the moment the current path stops matching any word prefix. Two optimizations turn a TLE into a pass — stop the DFS when you've matched a word, and delete exhausted branches as you go.

python
# inside the grid DFS, after a word match:
if node.is_word:
    found.append(word)
    node.is_word = False   # don't report duplicates
if not node.children:      # dead branch, prune it from the parent
    ...


The same structure also shows up in 648 (Replace Words), 677 (Map Sum Pairs), and 336 (Palindrome Pairs, which needs a good deal more thought). None of them require a new data structure — just extra fields on the node.

## Mistakes that cost points

Forgetting the end-of-word flag and treating "path exists" as "word exists." This is the single most common bug, and it's why `search("app")` returns `True` after inserting only `"apple"`.

Confusing prefix existence with word existence in the other direction, too: `starts_with("apple")` returning `True` for the word `"apple"` is correct — a whole word is a prefix of itself.

Recursing over deep tries. Python's default limit is 1000 frames. Word-search grids and long dictionary words get closer to that than you'd expect.

Building a fresh trie per test case when the problem gives you a fixed dictionary for many queries. Build once, query many times.

## When you shouldn't reach for a trie

If your queries are exact-match only, a `set` or `dict` wins on memory and is usually faster in CPython, because hashing a short string is fast and dict lookups are heavily optimized in C. Tries earn their keep specifically on prefix queries, ordered traversal, and "longest matching prefix" problems like IP routing tables.

If you do need a production-grade trie in Python, the standard library doesn't ship one. `marisa-trie` is a C++-backed implementation whose string data can take up to 50–100x less memory than a standard Python dict, at comparable raw lookup speed — but it's built for static, build-once-and-query dictionaries rather than heavy incremental insertion. `pygtrie` is the pure-Python option and implements the full mutable mapping interface, so a `Trie` can substitute for a `dict` in most places while adding prefix lookups like `longest_prefix()` and `shortest_prefix()`. For interview code, though, write your own — nobody wants to see an import in a whiteboard answer.

## Where this pattern fits in your prep

The trie itself is short. What trips people up in interviews isn't the node class — it's the moment a problem doesn't announce that it needs a trie. "Find all words in a board," "autocomplete," "longest common prefix" and "replace words with their roots" are all trie problems in disguise, and the recognition step is what actually gets tested.

That's the gap a pattern-based course is designed to close. AlgoMonster is built by people from Google and competitive programming backgrounds, and its structure is pattern-first rather than problem-first: the official site advertises 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, with all patterns, lessons, and problems unlocked on the paid tier. There's also an AI assistant and a timed "Speedrun" mode, both unlimited on Pro, plus a set of real questions sourced from top tech companies that get refreshed monthly. Two bonus courses — a CS foundations course and a build-an-LLM course — are included; partner-developed tracks like Quant Interview Prep are sold separately.

👉 [Start working through the trie and prefix-sum patterns on AlgoMonster](https://algo.monster?fpr=li69)

## AlgoMonster pricing, as listed on the official subscribe page

| Plan | What you get | Price | Billing | Buy |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample content — enough to judge the teaching style | $0 | — | [Create a free account](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant and Speedrun; company-sourced questions updated monthly; 12 months of access; member discount on live bootcamps; bonus Foundation and Build-an-LLM courses | $99/year ($8.25/mo), listed as 50% off a $198 list price | Billed annually | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates, one payment | $189, listed as 50% off a $378 list price | Billed once | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

A caveat worth stating plainly: AlgoMonster's pricing moves around with promotions, and third-party review sites quote a wide range of numbers that don't match the official page — you'll see everything from $89/year to $459 lifetime across blog posts. Treat the official checkout as the only reliable number and check it at the moment you buy. There's no monthly plan and no team tier; it's annual or lifetime, for individual users.

Which one makes sense depends on your timeline. If you're interviewing in the next two or three months, the annual plan costs less than a single mock interview. If you expect to switch jobs again in a couple of years — which most engineers do — the lifetime plan pays for itself after the second year of not re-subscribing.

## Quick answers

**Is "trie" pronounced "try" or "tree"?** The standard is "try," to distinguish it from a general tree. You'll hear "tree" in the wild and nobody will be confused by context.

**Does Python have a built-in trie?** No. `marisa-trie` and `pygtrie` are the usual third-party picks, both pip-installable.

**Dict or 26-array in an interview?** Dict. It handles Unicode and digits, it doesn't waste 26 slots per node, and it's shorter to write correctly under pressure. Mention the array trade-off out loud and you've covered the follow-up.

**What's the space complexity?** O(total characters inserted), worst case, and that's the honest answer — real tries compress shared prefixes and land well below the worst case, but never assume it.

The implementation is about thirty lines. The skill being tested is knowing which thirty lines to write, and then recognizing the next four problems where the same node class shows up again.

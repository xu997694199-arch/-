# Comparing placeholders during localization review

A translated interface string can read naturally and still pass the wrong values to its formatter. A player name may disappear, a score may move into a name slot, or one of two repeated variables may be lost. A small structural check can catch some of these mistakes before someone imports the translation into a game or application.

This tutorial builds a JavaScript comparison for a deliberately limited placeholder syntax. It preserves repeated tokens, allows named arguments to move, and keeps the order of unnamed printf arguments. Its output is a review finding: the formatter used by the application still defines the real contract.

## Define the supported syntax

The example accepts simple named braces such as `{name}` and numbered braces such as `{0}`. It also recognizes a subset of printf-style directives, including `%s`, `%d`, `%03d`, and `%2$s`. The examples assume that explicit positions identify arguments and that unnamed directives consume arguments in order. Confirm those assumptions against your application before using the checker.

For this tutorial, `{{name}}` means literal brace text and `%%` means a literal percent sign. Those are input conventions for this checker, not universal rules for translation files. A project using a different escape convention needs a different tokenizer.

Here are the comparisons we want:

| Source | Translation | Expected finding |
| --- | --- | --- |
| `{name}: {score}` | `{score} — {name}` | Same recognized tokens |
| `{name} met {name}` | `{name}来了` | Repeated token missing |
| `%s has %d coins` | `%d 金币属于 %s` | Unnamed argument order changed |
| `%1$s has %2$d coins` | `%2$d 金币属于 %1$s` | Explicit argument positions preserved |

A set is insufficient for the second case. Both sides contain the name `{name}`, so a comparison of unique names would accept the missing occurrence. Keep every occurrence and compare sorted arrays, or use a map from token to count.

## Preserve unnamed argument positions

Sorting every directive introduces another problem. The arrays `['%s', '%d']` and `['%d', '%s']` become equal after sorting, even though the consumer will receive arguments in a different order.

Attach an ordinal to each unnamed printf directive before sorting. The first source then produces `#0:%s` and `#1:%d`; the reordered translation produces `#0:%d` and `#1:%s`. Explicit directives such as `%2$d` already carry a position, so they do not need that extra label.

This check deliberately compares exact directive spelling. A change from `%03d` to `%d` is reported even when both accept an integer. The difference may be intentional, but a reviewer should decide whether removing zero padding is allowed.

## Run the complete example

Save the following as `placeholder-review.cjs` and run `node placeholder-review.cjs`. It uses Node's built-in assertion module and performs no file or network operations.

```javascript
'use strict';
const assert = require('node:assert/strict');

function recognizedTokens(text) {
  let ordinal = 0;
  const literalRemoved = text
    .replace(/\{\{[^{}]*\}\}/g, '')
    .replace(/%%/g, '');
  const pattern = /\{[A-Za-z_][A-Za-z0-9_.]*\}|\{\d+\}|%(?:\d+\$)?[-+ #0]*\d*(?:\.\d+)?[sdifuxXoegc]/g;
  return Array.from(literalRemoved.matchAll(pattern), ([token]) => {
    const unnamed = token.startsWith('%') && !/^%\d+\$/.test(token);
    return unnamed ? `#${ordinal++}:${token}` : token;
  }).sort();
}

function comparePlaceholders(source, target, syntax = 'simple') {
  if (typeof source !== 'string' || typeof target !== 'string') {
    throw new TypeError('Source and target must be strings');
  }
  // Route declared complex formats before the simple token comparison.
  if (syntax !== 'simple') return { status: 'manual-review' };
  const looksLikeICU = /\{[^{}]*,\s*(?:plural|select|selectordinal)\s*,/;
  if (looksLikeICU.test(source) || looksLikeICU.test(target)) {
    return { status: 'manual-review' };
  }
  const sourceTokens = recognizedTokens(source);
  const targetTokens = recognizedTokens(target);
  return {
    status: JSON.stringify(sourceTokens) === JSON.stringify(targetTokens)
      ? 'recognized-tokens-match' : 'mismatch',
    sourceTokens,
    targetTokens,
  };
}

const cases = [
  ['named reordering', '{name}: {score}', '{score} — {name}', 'recognized-tokens-match'],
  ['missing repetition', '{name} met {name}', '{name}来了', 'mismatch'],
  ['changed case', '{name}', '{Name}', 'mismatch'],
  ['explicit positions', '%1$s has %2$d coins', '%2$d 金币属于 %1$s', 'recognized-tokens-match'],
  ['unnamed positions', '%s has %d coins', '%d 金币属于 %s', 'mismatch'],
  ['literal escapes', '{{name}} 50%% {0}', '{{name}} 50%% {0}', 'recognized-tokens-match'],
  ['format changed', '%03d coins', '%d 金币', 'mismatch'],
  ['ICU hint', '{n, plural, one {leaf} other {leaves}}', '{n}片叶子', 'manual-review'],
];
for (const [name, source, target, expected] of cases) {
  assert.equal(comparePlaceholders(source, target).status, expected, name);
  console.log(`PASS ${name}`);
}
assert.equal(comparePlaceholders('text', '文字', 'icu').status, 'manual-review');
console.log('PASS declared complex format');
assert.throws(() => comparePlaceholders(null, '文字'), TypeError);
console.log('PASS nonstring input');
console.log('10 cases passed');
```

`matchAll` returns an iterator of regular-expression matches; the global flag makes the scan cover all matches. Here, `Array.from` turns each match into one comparison token. The callback labels unnamed directives while they are still in their original order. Sorting happens afterwards. [MDN reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/matchAll)

## Treat a matching result as a limited claim

The status name is intentionally specific: `recognized-tokens-match`. It says that this tokenizer found the same supported tokens. It does not certify that the entire input obeys a formatting grammar.

For example, unsupported directives such as `%*.*f`, malformed braces, or a different placeholder language can escape this comparison. The `syntax` argument lets an importer route a known complex format to manual review. The ICU-shaped regular expression is an additional hint for common plural and select expressions; it is not a complete ICU detector. File-format metadata and a parser for the actual formatter are stronger boundaries.

ICU messages can contain nested arguments and alternatives. Comparing only the simple brace fragments in such a message would lose the relationship between a selector and its branches. Use a parser compatible with the application's ICU implementation when that is the declared format. [ICU message documentation](https://unicode-org.github.io/icu/userguide/format_parse/messages/)

There is also a limit even within simple printf strings. Swapping two unnamed `%s` directives leaves the token sequence unchanged. A token checker cannot tell that one string represents a city and the other a person. Use explicit positions when the formatter supports them, and keep translator context describing the roles.

## Turn findings into a useful review queue

Keep each finding beside the stable string ID, source, translation, and both token arrays. That lets a reviewer see whether a variable disappeared, a format specifier changed, or a supported token was renamed. Preserve the submitted translation until someone approves a correction; automatically rewriting it can destroy an intentional change.

Repeated-token mismatches also require judgment. Some languages can avoid repeating a name while preserving the meaning. Whether that is acceptable depends on the product's formatting and translation policy. The checker provides a consistent signal, and the reviewer records the decision.

After structural review, render representative strings in the actual application with realistic values. Check narrow buttons, long names, plural variants, fonts, and right-to-left layouts when relevant. A matching token list tells you nothing about pixel width or whether the translated sentence makes sense.

The useful result is a smaller, better explained queue of strings to inspect. Keep the grammar narrow, preserve multiplicity, and label the limitations in the result so that downstream tools do not turn a lexical comparison into a release approval.


# Spring AI Output Converters — Bean, List, Map, and Fixing "Weird Text" Output

## 1. Why output converters exist

A language model only ever produces **text**. If your application actually needs a `List<String>`, a `Map<String, Object>`, or a real Java object with typed fields, something has to bridge the gap between "a blob of text the model wrote" and "an actual, usable Java value." That bridge is Spring AI's **output converter** system.

The contract every converter implements:

```java
public interface StructuredOutputConverter<T>
        extends Converter<String, T>, FormatProvider {
}
```

It's two responsibilities glued together:
- **`FormatProvider.getFormat()`** — produces a chunk of instruction text that tells the *model* exactly how to shape its answer (e.g. "respond only with RFC8259-compliant JSON matching this schema, no explanations, no markdown code blocks"). You splice this into your prompt so the model knows the rules before it ever generates anything.
- **`Converter<String, T>.convert(String)`** — takes the model's raw text response and actually turns it into your target type `T`, once the model (hopefully) followed the format instructions.

This two-part design is exactly why structured output is reliable rather than "ask nicely and hope" — you're both instructing the model on the shape you want, *and* doing the actual parsing yourself in a controlled way, rather than trusting the model's text to already be perfectly parseable.

Spring AI ships four things worth knowing here: three ready-to-use converters (**`BeanOutputConverter`**, **`ListOutputConverter`**, **`MapOutputConverter`**), plus the abstract base classes they're built on (**`AbstractConversionServiceOutputConverter`** and **`AbstractMessageOutputConverter`**) — which matter in their own right the moment you need to write a custom converter.

---

## 2. `BeanOutputConverter<T>` — model output straight into a Java object

The most commonly used converter. Give it a target class (or record), and it generates a JSON Schema from that type, instructs the model to produce JSON matching that schema, and deserializes the result back into an instance of your type.

```java
record ActorsFilms(String actor, List<String> movies) {}

public ActorsFilms getFilmography(String actor) {

    BeanOutputConverter<ActorsFilms> converter = new BeanOutputConverter<>(ActorsFilms.class);
    String format = converter.getFormat();

    String userTemplate = """
            Generate the filmography of 5 movies for {actor}.
            {format}
            """;

    String rawOutput = chatClient.prompt()
            .user(u -> u.text(userTemplate).param("actor", actor).param("format", format))
            .call()
            .content();

    return converter.convert(rawOutput);
}
```

Or, more conveniently, `ChatClient`'s own `.entity(Class<T>)` does all of this — building the format instructions, appending them, and converting the result — in one call (covered in the previous responses guide):

```java
ActorsFilms actorsFilms = chatClient.prompt()
        .user("Generate the filmography of 5 movies for Tom Hanks.")
        .call()
        .entity(ActorsFilms.class);
```

Under the hood, `.entity(ActorsFilms.class)` is literally constructing and using a `BeanOutputConverter` for you.

---

## 3. `ListOutputConverter` — model output as a `List<String>`

For when you just want a simple list back — no nested structure, just a set of string values (ingredients, tags, names, flavors, anything enumerable).

```java
public List<String> getFlavors() {

    ListOutputConverter converter = new ListOutputConverter(new DefaultConversionService());
    String format = converter.getFormat();

    return chatClient.prompt()
            .user(u -> u.text("List five {subject}\n{format}")
                    .param("subject", "ice cream flavors")
                    .param("format", format))
            .call()
            .entity(new ListOutputConverter(new DefaultConversionService()));
}
```

Its `getFormat()` instructs the model to respond as a **comma-separated list** (e.g. `foo, bar, baz`) — deliberately the simplest possible shape to parse reliably, since asking for a bulleted or numbered list instead invites exactly the kind of inconsistent formatting covered in §6.

---

## 4. `MapOutputConverter` — model output as a `Map<String, Object>`

For loosely-structured key/value data where defining a full Java class would be overkill.

```java
public Map<String, Object> getNumbersMap() {

    MapOutputConverter converter = new MapOutputConverter();
    String format = converter.getFormat();

    String template = """
            Provide me a list of {subject}
            {format}
            """;

    String rawOutput = chatClient.prompt()
            .user(u -> u.text(template)
                    .param("subject", "an array of numbers from 1 to 9 under the key name 'numbers'")
                    .param("format", format))
            .call()
            .content();

    return converter.convert(rawOutput);
}
```

`MapOutputConverter`'s format instructions ask for RFC8259-compliant JSON, then use a `MessageConverter` internally to turn that JSON into a plain `Map<String, Object>` — useful when you want structure without committing to a fixed schema/class.

---

## 5. The fourth important piece — the base classes, and writing your own converter

The "other important one" beyond the three ready-made converters is really **two abstract base classes** that back all of them, and that you extend when the built-ins don't fit:

- **`AbstractConversionServiceOutputConverter<T>`** — gives you a pre-configured `GenericConversionService` (Spring's general type-conversion machinery) to do the actual `String → T` transformation, but provides **no default `getFormat()`** — you supply your own format instructions. `ListOutputConverter` is built on this.
- **`AbstractMessageOutputConverter<T>`** — gives you a pre-configured `MessageConverter` (the same abstraction Spring MVC uses for HTTP message conversion) for the transformation step, again with no default format instructions of its own. `MapOutputConverter` is built on this.

**When you'd write a custom converter:** you need a target shape none of the three built-ins directly support — e.g. converting into a specific XML format, a CSV row, or a domain type with validation logic beyond what a JSON Schema can express. You implement `StructuredOutputConverter<T>` directly (or extend one of the two abstract bases), write your own `getFormat()` describing exactly what you want the model to produce, and your own `convert(String)` to parse it:

```java
public class CsvRowOutputConverter implements StructuredOutputConverter<List<String>> {

    @Override
    public String getFormat() {
        return """
                Respond with exactly one line of comma-separated values, no header row,
                no explanations, no markdown formatting, no surrounding quotes.
                """;
    }

    @Override
    public List<String> convert(String text) {
        return Arrays.stream(text.strip().split(","))
                .map(String::strip)
                .toList();
    }
}
```

Plugging a custom converter into `ChatClient` uses the exact same `.entity(StructuredOutputConverter<T>)` overload mentioned in the responses guide:

```java
List<String> row = chatClient.prompt()
        .user("Give me a CSV row of 3 European capital cities.")
        .call()
        .entity(new CsvRowOutputConverter());
```

---

## 6. "The text is weird" — where messy output actually comes from, and how to fix it

This is the problem almost everyone runs into the first time they try to get clean, structured, or plain text back from a model: extra asterisks, stray backticks, literal `\n` characters instead of real line breaks, brackets and commas that shouldn't be there, code fences wrapped around JSON, numbered-list artifacts mixed into what should be a clean comma list. None of this is a Spring AI bug — it's a mismatch between what the model *tends* to produce by default and what your code is *assuming* it will produce. Here's each specific symptom and its actual fix.

### 6.1 Markdown artifacts (`**bold**`, `# headers`, `- bullets`) leaking into plain text

**Why it happens:** chat models are heavily trained to format answers as Markdown for a chat UI — bold text, headers, bullet points — because that's how most of their training/RLHF data expects responses to look. If your application displays the raw string somewhere that doesn't render Markdown (a plain `<span>`, a log line, a text field), those literal asterisks and dashes show up as ugly noise instead of formatting.

**The fix:** explicitly tell the model not to use Markdown, in the system message, whenever you need genuinely plain text:

```java
chatClient.prompt()
        .system("Respond in plain text only. Do not use Markdown formatting — no asterisks, "
                + "no bullet points, no headers, no code blocks.")
        .user(question)
        .call()
        .content();
```

### 6.2 Raw JSON wrapped in ` ```json ... ``` ` code fences

**Why it happens:** even when explicitly asked for "just JSON," models frequently still wrap the JSON in a Markdown code fence out of habit, because that's the conventional way to present code/JSON in a chat response. If you're manually parsing the string yourself (rather than going through a converter), that leading ` ```json ` and trailing ` ``` ` breaks a naive `JSON.parse`/Jackson call immediately.

**The fix, in order of preference:**
1. **Use the actual converter (`BeanOutputConverter`, `.entity(...)`), don't hand-parse.** Its `getFormat()` instructions explicitly tell the model *"do not include markdown code blocks in your response,"* and the converter implementation is written to defensively strip fences if the model adds them anyway — this exact problem is precisely what the built-in converters are designed to absorb for you.
2. If you're stuck manually parsing for some reason, strip fences defensively before parsing:

```java
private String stripCodeFences(String text) {
    return text.strip()
            .replaceAll("^```[a-zA-Z]*\\s*", "")
            .replaceAll("```\\s*$", "");
}
```

### 6.3 Literal `\n`, `\"`, or other escape sequences showing up as actual backslash-characters

**Why it happens:** this almost always means you're looking at a **JSON string that was never actually deserialized** — you're displaying the raw JSON text (where newlines inside a string value are legitimately encoded as the two characters `\` and `n`) instead of the *decoded* Java `String` value. This is a classic sign of calling `.content()` on structured JSON output and displaying it as-is, rather than running it through a converter (or Jackson) first.

**The fix:** always run JSON output through the actual converter/deserializer before displaying it — `.entity(...)` or `converter.convert(...)` — never show `.content()`'s raw text directly when you asked the model for JSON. Once deserialized into a real Java `String` field, `\n` becomes an actual line break again, because the JSON decoding step is exactly what un-escapes it.

### 6.4 Extra `[`, `]`, and commas around what should be a clean list

**Why it happens:** two different root causes produce the same visual symptom:
- You asked for JSON (`["foo", "bar", "baz"]`) and are printing the raw string instead of parsing it — the brackets and quotes you're seeing are literally still there because nothing decoded them.
- You *did* get back a real `List<String>`, but then printed it with something like `System.out.println(list)` or string-concatenated it directly — Java's default `List.toString()` produces exactly `[foo, bar, baz]`, brackets and all, which is correct behavior for debugging output but not what you want to show a user.

**The fix:**
- For the first case: use `ListOutputConverter`/`.entity(...)` to actually parse it into a real `List<String>`, rather than displaying the raw text.
- For the second case: format the list yourself for display, rather than relying on `toString()`:

```java
String display = String.join(", ", flavorList); // "vanilla, chocolate, mint"
```

### 6.5 Inconsistent numbering (`1. foo, 2. bar`) mixed into what should be a plain comma list

**Why it happens:** if your prompt just casually says "give me a list of X" without the converter's explicit format instructions, the model defaults to whatever list style it thinks looks best — sometimes a numbered list, sometimes bullets, sometimes commas — and that inconsistency is exactly what breaks naive comma-splitting code.

**The fix:** always include the converter's actual `getFormat()` text in your prompt (via `{format}` in a template, or by using `.entity(...)` which does it automatically) rather than trusting the model to guess a machine-parseable format on its own. `ListOutputConverter`'s format instructions are deliberately explicit about wanting comma-separated values and nothing else, specifically to head off exactly this kind of drift.

### 6.6 The general principle behind all of the above

Every one of these symptoms comes from the same root cause: **treating the model's raw text as if it were already the data you wanted, instead of treating it as text that still needs to be validated and converted.** The output converter system exists precisely to remove that gap — `getFormat()` reduces the *chance* of messy output by telling the model exactly what's expected, and `convert(...)` (or `.entity(...)`, which uses `validateSchema()` to retry automatically on a mismatch, as covered in the responses guide) handles the case where the model doesn't perfectly comply anyway. The fix for "the text is weird" is almost always **"stop parsing it by hand, and let the actual converter (or `.entity(...)`) do the format instruction and parsing for you."**

---

## 7. Recap

| Symptom | Root cause | Fix |
|---|---|---|
| Stray `**`, `#`, `-` in plain text | Model defaults to Markdown-style chat formatting | Explicitly instruct "plain text only, no Markdown" in the system message |
| ` ```json ... ``` ` wrapping structured output | Model habitually fences code/JSON | Use `BeanOutputConverter`/`.entity(...)` — its instructions + defensive stripping handle this |
| Literal `\n`/`\"` visible in output | Displaying raw undecoded JSON instead of the parsed value | Deserialize first (`.entity(...)`/`convert(...)`), display the resulting Java value, not the raw text |
| Extra `[`, `]`, quotes around a list | Printing raw JSON, or `toString()`-ing a `List` for display | Parse with `ListOutputConverter`/`.entity(...)`; format lists explicitly with `String.join(...)` for display |
| Inconsistent numbering/bullets in a list | No explicit format instructions given to the model | Always include the converter's `getFormat()` text in the prompt, or use `.entity(...)` |

`BeanOutputConverter`, `ListOutputConverter`, and `MapOutputConverter` cover the three common target shapes (object, list, map); `AbstractConversionServiceOutputConverter`/`AbstractMessageOutputConverter` are what you build on for anything custom. And in every case, "weird" output is best treated not as a text-cleanup problem to patch after the fact, but as a signal that the model was never given (or never asked to honor) an explicit, machine-checkable format in the first place.

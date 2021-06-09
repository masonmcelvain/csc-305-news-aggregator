# <i class="fas fa-hat-wizard" style="color:#FA023C"></i> Decorator Pattern
When the same articles are repeatedly retrieved from a source, we only want to
print/display new articles to the user. This can be accomplished by caching the
articles seen in the most recent retrieval and filtering out any duplicates on
the next retrieval. There will be one cache per `Processor`. We also want to
make it easy for other types of post-processors (e.g. article sorters, article
mappers) to be added in the future. Filtering articles without modifying the
`Processor` itself is a good use case for the Decorator Pattern.

## Pattern Used & SOLID Principle Addressed
The Decorator Pattern is a both a behavioral and a structural pattern that adds
additional implementation to a base object by passing it as an argument to a
decorator class, and using the decorator instance in place of the base instance.

For example, if we had a base `PlainSequence` class along with the decorators
`SortedSequence` and `FilteredSequence` that all implemented the `Sequence`
interface, we might use the decorator pattern to impose new functionality upon a
`PlainSequence`.
```java
public interface Sequence<T> {
  public List<T> getAsList();
}

Sequence sequence = new SortedSequence(new FilteredSequence(new PlainSequence(3, 1, 2, 4)));
```

The `FilteredSequence` would filter the results of `PlainSequence.getAsList()`
and the `SortedSequence` would sort the results of
`FilteredSequence.getAsList()`.

With this pattern, each 'node' (decorator) adds additional functionality, as
opposed to the Chain of Responsibility Pattern where only a single node handles
a request. The Decorator Pattern allows the user to dynamically assign or
withdraw responsibilities. This pattern satisfies the Liskov substitution
principle, as any combination of decorators can be used in place of the base
class.

## Implementation of Design
To apply the Decorator Pattern to the task of filtering duplicate articles using
a cache, we'll introduce the `ProcessorArticleCache`. Both it and the base
`Processor` class implement the `ArticleProcessor` interface.
```java
public interface ArticleProcessor {
  public List<Article> process(Logger logger) throws IOException;
}
```

The `ProcessorArticleCache` must accept a `Processor` in its constructor in
order to be used as a decorator. Upon construction, it will also initialize an
empty `List` to use as a cache.

```java
public class ProcessorArticleCache implements ArticleProcessor {
  private final ArticleProcessor processor;
  private List<Article> cachedArticles;

  public ProcessorArticleCache(ArticleProcessor processor) {
    this.processor = processor;
    this.cachedArticles = List.of();
  }

  public List<Article> process(Logger logger) throws IOException {
    List<Article> mostRecentArticles = processor.process(logger);

    List<Article> filteredArticles = mostRecentArticles.stream()
        .filter(a -> !cachedArticles.contains(a);)
        .collect(Collectors.toList());

    cachedArticles = mostRecentArticles;
    return filteredArticles;
  }
}
```

When `ProcessorArticleCache.process()` is invoked, the decorator collects the
retrieved articles from the `process()` method of the `Processor`. It then
filters out any collected articles that exist in the cache. Before returning the
filtered articles to the caller, the `ProcessorArticleCache` updates the cache
to contain only the most recently retrieved articles.

The client might construct and use a decorated `Processor` much like an
undecorated `Processor`.
```java
ArticleProcessor processor = new ProcessorArticleCache(new Processor(parser, source));
List<Article> articles = processor.process(logger);
```

## Benefits of This Design in Context
Using the Decorator Pattern makes the implementation more flexible to change. If
the we needed to add other modifiers to the `Processor` in the future, we could
easily add more decorators to operate on the articles returned by the
`ProcessorArticleCache`. We could also easily combine or withdraw decorators in
different orders to achieve different effects, composing unique functionality
from small, simple pieces.

The Decorator Pattern allows the `Processor` to remain uncoupled from any
caching or filtering. The single-responsibility principle is satisfied since the
`Processor`'s role is to generate articles from a `Parser` and a `DataSource`,
and the `ProcessorArticleCache`'s role is to ensure no duplicate articles are
delivered to the client.

## Comparison Against Alternative Designs
Instead of using the Decorator Pattern, we could have made the
`ProcessorArticleCache` an independent utility class. This would completely
decouple the `ProcessorArticleCache` from the `Processor`, making the code a bit
simpler. However, instead of decorating the `Processor` during construction on
the client side, the client would be forced to collect the results of
`Processor.process()` and explicitly invoke `ProcessorArticleCache.process()` on
those results.
```java
List<Article> articles = processor.process(logger);
List<Article> filteredArticles = processorArticleCache.process(articles, logger);
```
One could imagine that if many decorators had been converted to utility classes,
this could require careful sequencing of post-processors by the client after
each invocation of `Processor.process()`.

As another alternative to the Decorator Pattern, we could make the
`ProcessorArticleCache` a subclass that extends the `Processor` class.
`ProcessorArticleCache.process()` could override `Processor.process()` and
filter the results of `super.process()`. This approach would work, and would
still let us use a `ProcessorArticleCache` wherever a `Processor` is expected.
However, this approach begins to become brittle when other post-processors are
introduced.

Say, for example, we wanted to add a `ProcessorArticleSorter` class that sorts
the articles. Should it inherit from `ProcessorArticleCache`? If so, then we
can't use a `ProcessorArticleSorter` without filtering. Should it then inherit
from `Processor`? If so, we couldn't sort and filter the same set of articles.
Composing a multitude of post-processors becomes nearly impossible due to the
limitations of single inheritance, making the Decorator Pattern a better choice
for long term maintanability.

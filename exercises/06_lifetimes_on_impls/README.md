# Lifetimes on Impls

When structs or enums have lifetimes on them, the way that `impl` blocks
work also changes slightly.

For example, say we want to create a struct that lets the user
iterate over a sentence. You might start off with something like this:

``` rust,ignore
// First, the struct:

/// This struct keeps track of where we are in the string.
struct WordIterator<'s> {
    position: usize,
    string: &'s str
}

impl WordIterator {
    /// Creates a new WordIterator for a string.
    fn new(string: &str) -> WordIterator {
        WordIterator {
            position: 0,
            string
        }
    }

    /// Gives the next word, or `None` if there aren't any words left.
    fn next_word(&mut self) -> Option<&str> {
        let start_of_word = &self.string[self.position..];
        let index_of_next_space = start_of_word.find(' ').unwrap_or(start_of_word.len());
        if start_of_word.len() != 0 {
            self.position += index_of_next_space + 1;
            Some(&start_of_word[..index_of_next_space]) 
        } else {
            None
        }
    }
}

fn main() {
    let text = String::from("Twas brillig, and the slithy toves // Did gyre and gimble in the wabe: // All mimsy were the borogoves, // And the mome raths outgrabe. ");
    let mut word_iterator = WordIterator::new(&text);

    assert_eq!(word_iterator.next_word(), Some("Twas"));
    assert_eq!(word_iterator.next_word(), Some("brillig,"));
}
```

When defining our `WordIterator` struct, we said it requires a lifetime to be specified.
But when we wrote the impl block, we didn't specify one. Rust requires that we do so.

The way we do this is by telling Rust about a lifetime, and then putting that lifetime onto
our struct. Let's see how we do that:

``` rust,ignore
impl<'lifetime> for WordIterator<'lifetime> {
    // ...
}
```

Note that we've done this in two parts -- first, `impl<'lifetime>` defines a lifetime `'lifetime`.
It doesn't make any promises about what that lifetime is, it just says that it exists. This allows us to use that lifetime within the impl block.

Next, `WordIterator<'lifetime>` uses the lifetime to say that "the references in `WordIterator` must live for `lifetime`". And in the rest of the impl block, any reference we annotate with `'lifetime'`
must have the same lifetime as any other reference annotated with `'lifetime'`.

``` rust,ignore
/// This struct keeps track of where we are in the string.
struct WordIterator<'s> {
    position: usize,
    string: &'s str
}

impl<'lifetime> WordIterator<'lifetime> {
    /// Creates a new WordIterator for a string.
    fn new(string: &'lifetime str) -> WordIterator<'lifetime> {
        WordIterator {
            position: 0,
            string
        }
    }

    /// Gives the next word, or `None` if there aren't any words left.
    fn next_word(&mut self) -> Option<&str> {
        let start_of_word = &self.string[self.position..];
        let index_of_next_space = start_of_word.find(' ').unwrap_or(start_of_word.len());
        if start_of_word.len() != 0 {
            self.position += index_of_next_space + 1;
            Some(&start_of_word[..index_of_next_space]) 
        } else {
            None
        }
    }
}

fn main() {
    let text = String::from("Twas brillig, and the slithy toves // Did gyre and gimble in the wabe: // All mimsy were the borogoves, // And the mome raths outgrabe. ");
    let mut word_iterator = WordIterator::new(&text);

    assert_eq!(word_iterator.next_word(), Some("Twas"));
    assert_eq!(word_iterator.next_word(), Some("brillig,"));
}
```

## Lifetime elision, redux

We previously discussed two rules for lifetime elision. They are:

1. Each place where an input lifetime is omitted (elided) is given its own lifetime.
2. If there's exactly one lifetime across all the input references, that lifetime is assigned to *every* output reference.

Now that we've seen `impl` blocks that have lifetimes, let's discuss one more:

3. If there are multiple input references, but one of them is `&self` or
   `&mut self`, the lifetime of the borrow of `self` is assigned to all elided output lifetimes.

This means that even if you take in many references in your arguments, Rust will assume that any references you return come from `self`, not from any of the other references.

## Exercise

In the following code, we annotate the function using the `'borrow` lifetime in addition to the `'lifetime` lifetime.
The `'borrow` lifetime exists only inside this function, and affects only the borrows of its arguments and return
value. The `'lifetime` value, as we saw before, also constrains the lifetime of the string inside the struct.

There are four ways we could implement this code. Describe the effect of each of these implementations. Scroll down for the answers.

Specifically:
 - Do they compile?
 - Are any of them identical to another one?
 - Are there any circumstances where their lifetimes are not general enough?
 - Which would be the "most correct" to write?

### Example 1
``` rust,ignore
    /// Gives the next word, or `None` if there aren't any words left.
    fn next_word<'borrow>(&'borrow mut self) -> Option<&'borrow str> {
        // ...
    }
```

### Example 2
``` rust,ignore
    /// Gives the next word, or `None` if there aren't any words left.
    fn next_word<'borrow>(&'borrow mut self) -> Option<&'lifetime str> {
        // ...
    }
```

### Example 3
``` rust,ignore
    /// Gives the next word, or `None` if there aren't any words left.
    fn next_word(&mut self) -> Option<&'lifetime str> {
        // ...
    }
```

### Example 4
``` rust,ignore
    /// Gives the next word, or `None` if there aren't any words left.
    fn next_word(&mut self) -> Option<&str> {
        // ...
    }
```

### Answers

- Example 1: This compiles. It's equivalent to Example 4. This version is problematic because it extends the lifetime of the mutable borrow of the iterator to match the lifetime of the returned word. Calling the function again to get the next word creates another mutable borrow, so you cannnot do so until you first drop the current word.
- Example 2: This compiles. It's equivalent to Example 3.
- Example 3: This compiles. It's probably the "most correct", because it's the shortest version that ensures you can retain the returned strings, even if you call this function multiple times.
- Example 4: This compiles. If expanded, it would be the same as Example 1.

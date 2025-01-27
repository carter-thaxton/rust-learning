# `&'a Struct<'a>` and covariance

Here's a situation that looks similar to [borrowing something forever,](./pf-borrow-forever.md)
but is actually somewhat different.

```rust
struct Person<'a> {
    given: &'a str,
    sur: &'a str,
}

impl<'a> Person<'a> {
    //       vvvvvvvv `&'a Person<'a>`
    fn given(&'a self) -> &'a str {
        self.given
    }
}

fn example(person: Person<'_>) {
    // Unlike when we borrowed something forever, this compiles
    let _one = person.given();
    let _two = person.given();
}
```

The difference is that `&U` is covariant in `U`, so
[lifetimes can "shrink" behind the reference](./st-invariance.md)
(unlike `&mut U`, which is invariant in `U`).  `Person<'a>` is also
covariant in `'a`, because all of our uses of `'a` in the definition
are in covariant position.

What all this means is that `&'long Person<'long>` can coerce to
a `&'short Person<'short>`.  As a result, calling `Person::given`
doesn't have to borrow the `person` forever -- it only has to borrow
`person` for as long as the return value is used.

## This is still a yellow flag

Even though it's not as problematic as the `&mut` case, there is
still something non-ideal about that signature: it forces the
borrow of `person` to be longer than it needs to be.  For example,
this fails:
```rust
# struct Person<'a> { given: &'a str }
# impl<'a> Person<'a> { fn given(&'a self) -> &'a str { self.given } }
# struct Stork(String);
# impl Stork { fn deliver(&self, _: usize) -> Person<'_> { Person { given: &self.0 } } }
fn example(stork: Stork) {
    let mut last = "";
    for i in 0..10 {
        let person = stork.deliver(i);
        last = person.given();
        // ...
    }
    println!("Last: {last}");
}
```

`person` has to remain borrowed for as long the return value is around,
because we said `&self` and the returned `&str` have to have the same
lifetime.

If we instead allow the lifetimes to be different:
```rust
# struct Person<'a> { given: &'a str }
# struct Stork(String);
# impl Stork { fn deliver(&self, _: usize) -> Person<'_> { Person { given: &self.0 } } }
impl<'a> Person<'a> {
    //       vvvvv We removed `'a` from `&self`
    fn given(&self) -> &'a str {
        self.given
    }
}

fn example(stork: Stork) {
    let mut last = "";
    for i in 0..10 {
        let person = stork.deliver(i);
        last = person.given();
        // ...
    }   
    println!("Last: {last}");
}   
```
Then the borrow of `person` can end immediately after the call, even
while the return value remains usable.  This is possible because we're
just copying the reference out.  Or if you prefer to think of it another
way, we're handing out a reborrow of an existing borrow we were holding 
on to, and not borrowing something we owned ourselves.

So now the `stork` still has to be around for as long as `last` is used,
but the `person` can go away at the end of the loop.

Allowing the lifetimes to be different is normally what you want to do
when you have a struct that's just managing a borrowed resource in some
way -- when you hand out pieces of the borrowed resource, you want them
to be tied to the lifetime of the original borrow and not the lifetime of
`&self` or `&mut self` on the method call. [It's how borrowing iterators
work,](https://doc.rust-lang.org/std/slice/struct.Iter.html#impl-Iterator-for-Iter%3C'a,+T%3E)
for example.

## A variation on the theme

Consider this version of the method:
```rust
# struct Person<'a> { given: &'a str }
impl<'a> Person<'a> {
    fn given(&self) -> &str {
        self.given
   }
}
```

It has the same downside as `given(&'a self) -> &'a str`: The return
value is tied to `self` and not `'a`.  It's easy to make this mistake
when developing borrowing structs, because the lifetime elision rules
nudge you in this direction.  It's also harder to spot because there's
no `&'a self` to clue you in.

## But sometimes it's perfectly okay

On the flip side, because of the covariance we discussed at the top of
this page, there's no practical difference between these two methods:
```rust
# struct Person<'a> { given: &'a str }
impl<'a> Person<'a> {
    fn foo(&self) {}
    fn bar(&'a self) {}
```

There's no return value to force the lifetime to be longer, so these
methods are going to act the same.  There's no reason for the `'a`
on `&'a self`, but it's not hurting anything either.

Similarly, within a struct there's rarely a benefit to keeping the
nested lifetimes separated, so you might as well use this:
```rust
struct Cradle<'a> {
    person: &'a Person<'a>
}
```
Instead of something with two lifetimes.

(That said, an even better approach is to not have complicated nested-borrow-holding data structures at all.)

---
layout: post
title: "What difference does it make?"
date: 2020-10-11
categories: Foundations
---

Some of the most important software we use makes heavy use of delta encoding. Delta encoding is, at a high level, the art of representing the differences between two values as concisely as possible. It's used when we have an old copy of some value, let's call it Copy A, and a new copy, Copy B, and for whatever reason we want to represent Copy B using a minimum of space. We may be sending it over the wire to a recipient who needs to know the new value, or alternatively we may be storing it in a log, as if to say "something changed: this was the value of this variable at this point in time". The latter is effectively the point of Git.

This blog post explores a few techniques for delta encoding. But ultimately it makes a more important point: the technique of delta encoding has huge unrecognised potential for making applications more performant.

## Techniques for delta encoding

First off, I'd like to make a counter-intuitive claim about diffing algorithms. At first blush you'd be forgiven for thinking that what makes a good diffing algorithm is - shock horror - being able to tell what differs. I would argue that what makes a good diffing algorithm is in fact _being able to tell what's stayed the same_. Yes, technically this amounts to the same thing. But when we talk about the maths behind diffing algorithms, this is worth keeping in mind as a north star.

What we're looking for when we look for a diffing algorithm is a partial [bijection](https://en.wikipedia.org/wiki/Bijection) between two vectors: that is, a function which successfully maps part of the set of values in Copy A to the set of values that constitutes Copy B. The best diffing algorithm is that which establishes the largest possible domain of that bijective function, _excluding_ incorrectly mapped values.

Let's express it in one rule:

> For each value in vector A, the perfect diffing algorithm maps it to some element in vector B iff that element _is the same element_ as it was in vector A. The more often the diffing algorithm gets it right (i.e. tends towards perfection) the better it is.

### The homomorphic algorithm

When talking about technical topics, it _always_ helps to start with the crudest technique as a jumping off point. The crudest algorithm for delta encoding would go something like this:

<iframe src="https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&code=fn%20main()%20%7B%0A%20%20%20%20let%20s1%20%3D%20%22Hello%20world%22%3B%0A%20%20%20%20let%20s2%20%3D%20%22Hllo%20world%22%3B%0A%0A%20%20%20%20let%20diff%20%3D%20calculate_diff(s1%2C%20s2)%3B%0A%20%20%20%20println!(%22Diff%3A%20%7B%3A%3F%7D%22%2C%20diff)%3B%0A%7D%0A%0Afn%20calculate_diff(s1%3A%20%26str%2C%20s2%3A%20%26str)%20-%3E%20Vec%3C(usize%2C%20char)%3E%20%7B%0A%20%20%20%20s1.chars()%0A%20%20%20%20%20%20%20%20.enumerate()%0A%20%20%20%20%20%20%20%20.filter_map(%7C(i%2C%20char)%7C%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20if%20let%20Some(s2_char)%20%3D%20s2.chars().nth(i)%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20if%20s2_char%20!%3D%20char%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20Some((i%2C%20s2_char))%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20None%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20None%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D)%0A%20%20%20%20%20%20%20%20.collect()%0A%7D%0A" style="width: 100%; height: 600px"></iframe>

> Step through vector A, element (i.e. byte or character) by element. For each element, if that element is different from the element at the same index in vector A, write it down along with the index. Now, on the other 'end', you can apply the changes to Copy A by simply going through the changelog and applying each change at each changed index.

Let's call this the homomorphic approach, because the mapping it establishes is like a topological homomorphism (in algebraic topology). In other words, the diff, if treated as a function, is an bijective function mapping one vector to another by pairing each element of one vector with an element of the other.

The downsides of this are quite easy to see. If we add one byte at the beginning, every subsequent byte will show as changed. The problem with the homomorphic algorithm is that it cannot cope with elements being added to or removed from the vector, since it treats _position in the vector_ as the only ground of identity between a value in Copy A and a value in Copy B.

### The structural algorithm

Most people at this point will have a vague inkling of what the better way of doing this is. I'd like to call this the 'structural' algorithm. If the homomorphic algorithm used _position_ to map an element in vector A to another in vector B, the structural algorithm takes account of the surrounding elements.

We could phrase this as something more like:

> Step through vector A, element (i.e. byte or character) by element. For each element, if that element is different from the element at the same index in vector A, write it down along with the index. Now, on the other 'end', you can apply the changes to Copy A by simply going through the changelog and applying each change at each changed index.

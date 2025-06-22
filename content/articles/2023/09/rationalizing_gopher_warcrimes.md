---
title: Rationalizing Gopher War-crimes
description: "With all the recent drama around Rust and Typescript, I thought why not join in
the fun and share some additional atrocities in another language common to my
blog, Go! Specifically, these are super unidiomatic helper functions which can
be found over all of my modern projects that I thought you may find humorous.
More so, you'll notice that all of my war-crimes here make heavy use of Go
generics, which were introduced in 1.18. I'm sure I'm part of the small user
group hoping for more to be added in the near future, but after reading a post
like this perhaps the community will _strongly_ disagree."
tags: ["Open Source", "Go", "Cloud Engineering", "Kubernetes", "Cloud Native", "Software Development"]
Cover: https://images.unsplash.com/photo-1561323578-0b8e6fdc11c0?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA
date: 2023-09-09
---

With all the recent drama around Rust and Typescript, I thought why not join in
the fun and share some additional atrocities in another language common to my
blog, Go! Specifically, these are super unidiomatic helper functions which can
be found over all of my modern projects that I thought you may find humorous.
More so, you'll notice that all of my war-crimes here make heavy use of Go
generics, which were introduced in 1.18. I'm sure I'm part of the small user
group hoping for more to be added in the near future, but after reading a post
like this perhaps the community will _strongly_ disagree. Still, one can draw
inspiration from projects such as [IBM/fp-go](https://github.com/IBM/fp-go)
which adds similar schools of thought in non-idiomatic ways

## Must

I wrote this for the times where the returned error from a function would
result in the system needing to `panic`, meaning there's zero way to properly
mitigate the error. I commonly use this in places where credentials or configs
are being referenced, implying that they are either present, or fail. It reminds
me of Rust's `unwrap`, which handles in the same way on error.

```go
func Must[T any](val T, err error) T {
  if err != nil {
    panic(err)
  }

  return val
}

func main() {
	// Tradional if err approach
	env, err := config.GetEnvironment()
	if err != nil {
		panic(err)
	}

	// Must
	env := Must(config.GetEnvironent())
}
```

## GetOrDefault

When not using pointers, I hate having to check for the existence of a
initialized value in a field when I know the check will result in me setting the
value to a sane default. `GetOrDefault` allows me to retrieve the value from a
field and, if its the default value of the initialized type, return the default
value we set instead. Essentially, it converts a ternary or if/else statement
into a nice one liner.

```go
func GetOrDefault[T any](val, def T) T {
	if init := new(T); init == val {
		return def
	}

	return val
}

func main() {
	// Traditional implementation(s)
	env := config.GetEnvironmentString()
	if len(env) == 0 {
		env = "local"
	}

	// GetOrDefault
	var env = GetOrDefault(config.GetEnvironmentString(), "local")
}
```

## GetOrDo

In the case where setting a sane default is not enough, I give you; the
blockbuster, action-packed version! Realistically, though my example here
doesn't show it, this helper is useful for when you need to encapsulate a lambda
handler in the case of failure retrieving the original value.

```go
func GetOrDo[T any](val T, f func() T) T {
	if init := new(T); init == val {
		return f()
	}

	return val
}

func main() {
	// Traditiional implementation?
	env := config.GetEnvironmentString()
	if len(env) == 0 {
		resp, err := config.SyncEnvironmentConfiguration()
		if err != nil {
			panic(err)
		}

		env = resp.Body.Config
	}

	// GetOrDo, using a terrible example with Must
	var env := GetOrDo(config.GetEnvironmentString(), func() string {
		return Must(config.SyncEnvironmentConfiguration()).Body.Config
	})
}
```

## Chain

Go doesn't have anonymous lambda syntax, so even I'll admit beyond the symmetry,
this is horrendous looking. Yet, I find myself to write these "chains" or "pipes"
because it allows me to declaratively explain transformations I want to occur
on a value - this mostly being my implementation out of spite that method
chaining is not a go language feature for the native types, and Elixir capturing
my interest with their `|>` operator.

```go
func Chain[T any](val T, fn ...func(T) T) T {
	for f := range fn {
		val = f(val)
	}

	return val
}

func main() {
	// Ray's idiomatic way
	var (
		handle				= "@raygervais@mastodon.social"
		domain				= strings.Split(email, "@")[3]
		domainFormatted		= strings.Replace(username, ".", " ", -1)
		domainTitled		= strings.Title(domainFormatted)
	)

	// Ray's Chain way
	domain := Chain("@raygervis@mastodon.social",
		func(s string) string { return strings.Split(s, "@")[3] },
		func(s string) string { return strings.Replace(s, ".", " ", -1) },
		strings.Title,
	)
}
```

## Resources

- [Cover Image: Photo by Chandler Cruttenden on Unsplash](https://unsplash.com/photos/c55NHPguULU)
- [IBM/fp-go](https://github.com/IBM/fp-go)
- [Rust by Example: Option & unwrap](https://doc.rust-lang.org/rust-by-example/error/option_unwrap.html)
- [Scaler: GetOrDefault in Java](https://www.scaler.com/topics/getordefault-in-java/)
- [Kotlin: getOrElse](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/get-or-else.html)

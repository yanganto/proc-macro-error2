# Drop-in replacement to panics in proc-macros

## Motivation

Error handling in proc-macros sucks. It's not much of a choice today:
you either "bubble up" the error up to top-level of you macro and convert it to
a `compile_error!` invocation or just use a good old panic. Both these ways suck:

- Former sucks because it's quite redundant to unroll a proper error handling
    just for critical errors that will crash the macro anyway so people mostly
    choose not to bother with it at all and use panic. I have yet to see a crate > 700
    lines of code that does it, simple `.expect` is too tempting.
- Later sucks because panics aren't for error-reporting; panics are for bug-detecting
    (like unwrapping on `None` or out-of range indexing) or for early development stages
    when you need a prototype ASAP and error handling can wait. But main disadvantage
    of this approach is even simpler: no way to highlight exact place that's an error's source.

## Solution

That said, we need a solution, but this solution must meet these conditions:

- It must be better than panics. The main point: it must offer a way to carry span information
    over to user.
- It must require as little effort as possible to migrate from panic. Ideally, a new
    macro with the same semantics plus ability to carry out span info.

This crate aims to provide such a mechanism. All you have to do is enclose all
the code inside your top-level `#[proc_macro]` function in `filter_macro_errors!`
invocation and change panics to `span_error!`/`call_site_error!` where appropriate:

```rust
// This is your main entry point
#[proc_macro]
pub fn make_answer(input: TokenStream) -> TokenStream {
    // This macro **must** be placed at the top level.
    // No need to touch the code inside though.
    filter_macro_errors! {
        // `parse_macro_input!` and its friends work just fine inside this macro
        let input = parse_macro_input!(input as MyParser);

        if let Err(err) = some_logic(&input) {
            // we've got a span to blame, let's use it
            let span = err.span_should_be_highlighted();
            let msg = err.message();
            // This call jumps directly at the end of `filter_macro_errors!` invocation
            span_error!(span, "You made an error, go fix it: {}", msg);
        }
        
        // You can use some handy shortcuts if your error type
        // implements Into<MacroError>         
        use proc_macro_error::ResultExt;
        more_logic(&input).expect_or_exit("What a careless user, behave!");

        if !more_logic_for_logic_god!(&input) {
            // We don't have an exact location this time,
            // so just highlight the proc-macro invocation itself
            call_site_error!(
                "Bad, bad user! Now go stand in the corner and think about what you did!");
        }

        // Now all the processing is done, return `proc_macro::TokenStream`
        quote!(/* stuff */).into()
    }
    
    // At this point we have a new shining `proc_macro::TokenStream`!
}
```

## How it works
I must confess: I used panics as a try/catch mechanism. I've committed this
sin so others may live in peace and prosperity, god save my soul. 

Essentially, the `filter_macro_errors!` macro is a 
```C++
try { 
    /* your code */ 
} catch (MacroError) { 
    /* conversion to compile_error! */ 
}
```

`span_error!` and co are 
```C++
throw MacroError::new(span, format!(msg...));
```

When you do `span_error!` you trigger panic
that will be caught by `filter_macro_errors!` and converted to `compile_error!` invocation.
All the panics triggered not by `span_error!` and co will be resumed as is.

Panic catching is indeed *slow* but the macro is about to abort anyway so speed is not
a concern here. Please note that this crate is not intended to be used in any other way
than a proc-macro error reporting, use `Result` and `?` instead.

## Testing
TODO: fork https://github.com/laumann/compiletest-rs and make it understand explicit line numbers.
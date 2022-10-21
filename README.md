# Elixir Development Style Guide
To ensure consistent code readability and quality please refer to this guide for Elixir code standards.

## General Guidelines
Please read and follow the [Community Elixir Style Guide](https://github.com/christopheradams/elixir_style_guide/blob/master/README.md).

When designing and writing code please follow Single Responsibility and [Tell Don't Ask](https://sullymustycode.medium.com/tell-dont-ask-with-ruby-and-elixir-25597d3d130b) Principles as much as possible. 

If methods get longer than 5 lines and contain multiple nesting for flow control, please step back and see if the method can be divided into multiple methods.

Keep the Modules surface area small by using private methods as much as possible.

Please design for success, our software should know how to behave and deliver the desired results please don’t focus on misbehaving or errors heavily. 

Organize your code into [Contexts](https://hexdocs.pm/phoenix/contexts.html) and properly namespace them.

At a minimum ensure your Contexts methods have valid [Typespecs](https://hexdocs.pm/elixir/1.13/typespecs.html) to make the code easier to read and understand return values etc.

## Error Handling
Elixir has a **let it crash** philosophy thus please keep this in mind while handling errors. More information can be found about this design paradigm.

- [Erlang Let it crash Philosophy](https://medium.com/@vamsimokari/erlang-let-it-crash-philosophy-53486d2a6da)
- [Let it crash](https://verraes.net/2014/12/erlang-let-it-crash)

### Catching errors
Try not to play catch with yourself
``` elixir
if something do
  raise MyCustomError1
end
```
... In another Module...
```elixir
try do
  something
  rescue MyCustomError1 -> raise MyCustomError2
end
```
... Yet in another Module...
```elixir
try do
  something
  rescue MyCustomError2 -> "got this error"
end
```
This is not optimal design. We should not raise custom errors just to catch them further up the food chain. If you need to raise an error, handle it properly as soon as possible.

## 3 Error Scenarios
Errors in general are often misunderstood. There are 3 general scenarios when things go wrong. 

1. The user is trying to do something wrong: i.e. sending in bad information with a form, or trying to do something they shouldn't such as paying something twice, or access a resource they shouldn’t such as view an admin page when they are not an admin.

2. The software is doing something it shouldn’t: such as returning a wrong balance or information. This is often due to software being mis-programmed or business logic being misunderstood.

3. System failure: modern software depends on libraries, hardware, external services, etc. these dependencies will fail, or our code is written wrong.

### Scenario 1 - Users misbehaving
If a user is doing something wrong then we need to inform that user what they are doing wrong and or guide the user towards the right actions. These types of situations **should not raise errors!** they should be distinguished from the other scenarios in your code and return feedback to the API or UI.

The only exception is authentication and security threats. In these scenarios, such as bad passwords or credentials, we should not provide productive feedback. Instead return a bad request response, but we **should still not raise errors!**

### Scenario 2  - Software (we wrote) is misbehaving
Sometimes the software is doing exactly as it should, but that is not what we intended or we have misunderstood business logic and the results are wrong. 

In these scenarios the errors should be informative to include what method and module, along with any other state related information available.

The error information should be output to a log and made actionable as much as possible by pinging a slack channel or emailing a developer or team. 

Ensure that if there is a message that someone is productively working on it. We do not want to flood the teams with meaningless messages that then over time get ignored. In other words **we should not cry wolf**.

### Scenario 3 - System or Dependency failures
These should be handled similar to scenario 2 failures. The error information should be output to a log and made actionable as much as possible by pinging a slack channel or emailing a developer or team. 

Ensure that if there is a message that someone is productively working on it. We do not want to flood the teams with meaningless messages that then over time get ignored. 

## API Error responses
Please follow standard [HTTP error codes](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) when returning error responses through an API endpoint or UI. HTTP error codes are supported by many libraries and frameworks. There is no need to re-invent this wheel. 

Internal error codes should never be returned to anyone using our API's. Our internal error handling is our business and others who depend on our API should be given a productive message from the above guidelines if it falls under scenario 1. If the error is scenario 2 or 3 (i.e. an internal problem) then follow HTTP conventions and return an appropriate HTTP response!

## Testing Standards
### General Guidelines
The most common library for testing in Elixir is the [ExUnit Testing Package](https://hexdocs.pm/ex_unit/ExUnit.html) to maintain consistency please use this library for testing projects. For a brief tutorial please visit the [Elixir School Testing Tutorial](https://elixirschool.com/en/lessons/basics/testing/).

As a general rule all public methods in every module should have proper edge case testing. With Phoenix projects views and templates may be tested through controller tests.

It is ok to be verbose and repetitive in tests. Tests should be highly readable and very easy to understand. Obviously efficiency and DRY (Don't Repeat Yourself) is important but not at the sacrifice of readability. 

When testing please keep in mind state and make the tests use real life state scenarios. This may mean setting up objects and data in the background to ensure there is sufficient state. It is the responsibility of the test to establish state and this state should be isolated to the test and not leak out into the rest of the tests.

Below is an example of ensuring proper state. In this example we could have tested just the AccessCookiesServer ability to delete just the users cookies, however we would not know if this is just deleting all the cookies or just that users cookies. 

Therefore adding other users cookies into the test ensured we had a proper state and thus we can now know just the users cookies were deleted.

```elixir
    test "with a users uid deletes all that users cookies" do
      cookie_user = insert(:user)

      for _ <- 1..5 do
        user = insert(:user)
        AccessCookiesServer.create_cookie(user)
      end

      assert {:ok, _cookie} = AccessCookiesServer.create_cookie(cookie_user)
      assert {:ok, _cookie} = AccessCookiesServer.create_cookie(cookie_user)
      
      assert length(AccessCookiesServer.list_cookies()) == 7

      assert length(AccessCookiesServer.get_cookies(%{uid: cookie_user.uid})) == 2

      assert :ok = AccessCookiesServer.delete_cookies(%{uid: cookie_user.uid})

      assert AccessCookiesServer.get_cookies(%{uid: cookie_user.uid}) == []

      assert length(AccessCookiesServer.list_cookies()) == 5
    end
  ```
### Mocking
As a general rule only external services such as HTTP calls to external servers etc. We should never mock our own code! if the **state** or methods are too cumbersome to test, then you do not have a mock issue, you have a **design issue!** Step back, evaluate, and redesign your code.

If you do need to use a Mock for an external service please refer to the codebase for the existing Mock library and follow the documentation for that library.

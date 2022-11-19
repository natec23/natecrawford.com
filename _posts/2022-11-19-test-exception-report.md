---
layout: post
author: "Nate Crawford"
title: "Testing If an Exception Was Caught and Reported"
date: 2022-11-19 10:30:00 -0500
category: development
tags: [laravel]
---

When testing if an exception is thrown, there are a couple built in tests available incluidng `expectException` :

{% highlight php %}
$this->expectException(InvalidArgumentException::class);
{% endhighlight%}

This only works for uncaught exceptions. What if the exception is caught without rendoring the error message. How do you test an exception was caught and logged in laravel?

Let's assume we have some code that's running a HTTP request.

<h3>HTTP Request</h3>
{% highlight php %}
try {
    Http::post('example.com')->throw();
}
catch(Illuminate\Http\Client\RequestException $exception) {
    report($exception);
}
{% endhighlight%}

We don't need to render the message if that request fails but we do want to log the error. Instead of expecting the exception directly, we'll [mock](https://laravel.com/docs/9.x/mocking){:target="\_blank"} laravel's ExceptionHandler class to test that the report method was called (this is what's called from the [report helper](https://laravel.com/docs/9.x/errors#the-report-helper){:target="\_blank"}). We can also test that it recieved a specifc exception, in this case `Illuminate\Http\Client\RequestException` which is thrown by laravel's [Http client](https://laravel.com/docs/9.x/http-client){:target="\_blank"} library.

In this example we'll test that if the HTTP response is 200 we don't expect to see an error reported. When the HTTP response is 500, we expect the report method of ExceptionHandler to be called.

{% highlight php %}
use Illuminate\Contracts\Debug\ExceptionHandler;
use Illuminate\Support\Facades\Http;
use Mockery\MockInterface;
use Tests\TestCase;

class HttpExampleTest extends TestCase
{

  public function test_200_response()
  {
    Http::fake([
      '*' => Http::response('', 200)
    ]);
    $mock = $this->partialMock(ExceptionHandler::class, function(MockInterface $mock) {
      $mock->shouldNotReceive('report');
    });
    exampleRequest();
  }

  public function test_error_response()
  {
    Http::fake([
      '*' => Http::response('', 500)
    ]);

    $this->partialMock(ExceptionHandler::class, function(MockInterface $mock) {
      $mock->shouldReceive('report')->once()->with(RequestException::class);
    });

    exampleRequest();
  }
}
{% endhighlight %}
